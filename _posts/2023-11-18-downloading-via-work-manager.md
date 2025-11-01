---
title: Downloading Files using Work Manager
date: 2023-11-18 13:41:00 +0530
categories: [Engineering]
tags: [engineering]
---

Since Android 14, the recommended way to handle FGS jobs that fall into the category of `dataSync` is via [WorkManager](https://developer.android.com/topic/libraries/architecture/workmanager). Google explicitly states in the [Android 14 Behaviour Changes](https://developer.android.com/about/versions/14/changes/fgs-types-required#data-sync) docs that:

> Note: In a future version of Android, this foreground service type will be deprecated. We recommend you migrate to one of the listed alternatives.

In the aforementioned alternatives, [DownloadManager](https://developer.android.com/reference/android/app/DownloadManager) API is suggested to be used for downloading files. However, it has some limitations such as not being able to observe the download progress, enqueing group downloads to be considered as one, and many more.

As a result, the only other recommended alternative would be to download the files via the `WorkManager` library by creating a dedicated [CoroutineWorker](https://developer.android.com/reference/kotlin/androidx/work/CoroutineWorker).

This article's goal isn't to show you how to build `WorkRequest` using `WorkManager` but showcasing the useful additions that can be done on it to develop a good enough `DownloadManager` alternative. You might want to read [Step by Step Guide to Download Files With WorkManager](https://proandroiddev.com/step-by-step-guide-to-download-files-with-workmanager-b0231b03efd1) article on Medium by Rahul Ray, which builds up a good base for this one.

## Worker Requirements

There are multiple requirements or better to say restrictions we want from our implementation, such as:

- Unrestricted background operations to allow triggering downloads from background,
- Immediately triggering downloads as we enqueue them,
  - Only if there is no other download running, otherwise we want the worker to wait till previous work is finished
- Observe the download progress while downloading, and
- Do cleanup on failures

## Background Operations

Since Android 8.0+, there are multiple restrictions on running background work for Android apps to improve battery life and device performance. Considering downloads will be enqueued by the user and we may want to download the file at a certain time or condition, it's important to consider these implications imposed by the system.

The best recommendation would be to request users to allow running our app in the background and use the permission carefully. This can be done by showing users a dialog to disable the background optimizations and only enqueue background operations/downloads if permitted.

Add [REQUEST_IGNORE_BATTERY_OPTIMIZATIONS](https://developer.android.com/reference/android/Manifest.permission#REQUEST_IGNORE_BATTERY_OPTIMIZATIONS) permission to the app's `AndroidManifest.xml` file.

```xml
<uses-permission android:name="android.permission.REQUEST_IGNORE_BATTERY_OPTIMIZATIONS"/>
```

Now create a new `ActivityResultContracts` and start an `Intent` with [ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS](https://developer.android.com/reference/android/provider/Settings#ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS) with the app's package name as data.

```kotlin
import android.content.Context
import android.os.PowerManager
import android.provider.Settings
import android.util.Log

class PermissionsFragment : Fragment(R.layout.fragment_permissions) {

    private val startForDozeResult =
        registerForActivityResult(ActivityResultContracts.StartActivityForResult()) {
            // Check if the permission was granted or denied
            context?.let {
                val powerManager = it.getSystemService(Context.POWER_SERVICE) as PowerManager
                if (isMAndAbove() && powerManager.isIgnoringBatteryOptimizations(it.packageName)) {
                    Log.i(TAG, "Permission Granted!")
                } else {
                    Log.i(TAG, "Permission Denied!")
                }
            }
        }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        // Request permission after explaining to the user why it's required
        requestDozePermission()
    }

    private fun requestDozePermission() {
        // We only need permission if the app is running on an Android 6.0+ device
        if (isMAndAbove()) {
            val intent = Intent(Settings.ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS).apply {
                data = Uri.parse("package:${requireContext().packageName}")
            }
            startForDozeResult.launch(intent)
        }
    }

```

Once the permission is granted, we can easily schedule and trigger downloads even when the app isn't in the foreground.

> In case you are considering publishing your app on the Play Store, keep in mind that usage of this permission is subject to policy and your app can be denied if it doesn't meet the area of exemption mentioned [here](https://developer.android.com/training/monitoring-device-state/doze-standby.html#exemption-cases).

## Immediate Downloads

while `WorkManager` is generally used to schedule works in a near future, it does support running the task immediately since v2.7.0 as expedited work. Expedited work runs as FGS on Android versions older than 12.0. Therefore, for backwards compatability, we also need to define the FGS type in our `AndroidManifest.xml` file.

```xml
<service
    android:name="androidx.work.impl.foreground.SystemForegroundService"
    android:foregroundServiceType="dataSync"
    tools:node="merge" />
```

Once defined, we can simply call the [setExpedited](https://developer.android.com/reference/androidx/work/WorkRequest.Builder#setExpedited(androidx.work.OutOfQuotaPolicy)) method on the `WorkRequest` we will define to enqueue the download and the download will be triggered immediately.

While immediate downloads are great, we face an another issue which is that we want to eqneue only one download at once and wait till the previous one is finished before triggering the work. While `WorkManager` provides a convinent method to enqueue multiple work requests as defined [here](https://developer.android.com/guide/background/persistent/how-to/chain-work), this fails all other enqueued works even if a single one fails.

In some cases, this might be convinent such as group downloads, but in general sense, we might be downloading multiple files that may not be dependent on each other. In such cases, chaining work is not appropriate solution. Sadly, there is no alternative provided by `WorkManager` for this issue, though it is pretty easy to mitigate by storing the enqueued downloads either in a persistant storage or memory.

```kotlin
import com.example.app.data.models.DownloadFile

import kotlinx.coroutines.flow.MutableStateFlow

class ExampleApplication : Application() {

    companion object{
        val enqueuedDownloads = MutableStateFlow<MutableSet<DownloadFile>>(mutableSetOf())
    }
}
```

This `MutableStateFlow` can be then read via both `Activity` and the `Worker` class we are downloading the said files. `Activity` can trigger a `WorkRequest` for new downloads on list modification using the first object and the `Worker` can remove the same once it has finished downloading the file resulting in another modification for the loop to continue.

```kotlin
import com.example.app.data.work.DownloadWorker
import com.example.app.data.work.DownloadWorker.Companion.DOWNLOAD_WORKER

import kotlinx.coroutines.DelicateCoroutinesApi
import kotlinx.coroutines.GlobalScope
import kotlinx.coroutines.delay
import kotlinx.coroutines.flow.collect

class MainActivity : AppCompatActivity() {

    @OptIn(DelicateCoroutinesApi::class)
    override fun onCreate(savedInstanceState: Bundle?) {
        // Handle enqueued downloads
        GlobalScope.launch {
            ExampleApplication.enqueuedDownloads.collect { enqueuedDownloads ->
                try {
                    if (enqueuedDownloads.isNotEmpty()) {
                        // Let any existing work related to download get finished
                        delay(3000)
                        Log.i("Downloading ${enqueuedDownloads.first().name}")
                        // Call the companion method to enqueue the download by passing first object
                        DownloadWorker.download(applicationContext, enqueuedDownloads.first())
                    }
                } catch (exception: Exception) {
                    Log.i("Failed to download enqueued apps", exception)
                }
            }
        }
    }
```

> In case you would like to be more error-safe, consider creating a room database to maintain list of downloads.

## Observing Progress

`WorkManager` allows a convinetant API [setProgress](https://developer.android.com/reference/kotlin/androidx/work/CoroutineWorker#setProgress(androidx.work.Data)) to allow the `CoroutineWorker`'s progress to be observed from UI. Among the various methods available, we can use [getWorkInfosLiveData](https://developer.android.com/reference/androidx/work/WorkManager#getWorkInfosLiveData(androidx.work.WorkQuery)) or [getWorkInfosFlow](https://developer.android.com/reference/androidx/work/WorkManager#getWorkInfosFlow(androidx.work.WorkQuery)) to easily get a filtered list of the `WorkInfo` objects containing information of `WorkRequest` of our choice.

```kotlin
import com.example.app.data.models.DownloadInfo
import com.example.app.data.model.Request

import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.delay
import kotlinx.coroutines.flow.collectLatest
import kotlinx.coroutines.withContext
import java.io.File
import java.net.URL

class DownloadWorker(private val appContext: Context, workerParams: WorkerParameters) :
    CoroutineWorker(appContext, workerParams) {

    companion object {
        const val DOWNLOAD_PROGRESS = "DOWNLOAD_PROGRESS"
        const val DOWNLOAD_TIME = "DOWNLOAD_TIME"
        const val DOWNLOAD_SPEED = "DOWNLOAD_SPEED"
    }

    private var downloading = false

    private var totalBytes by Delegates.notNull<Long>()
    private var totalProgress = 0
    private var downloadedBytes = 0L

    override suspend fun doWork(): Result {
        // Download and verify all files exists
        val requestList = getDownloadRequest(files)

        // Total size of all associated files for UI progress
        totalBytes = requestList.sumOf { it.size }

        requestList.forEach { request ->
            downloading = true
            runCatching { downloadFile(request) }
                .onSuccess { downloading = false }
                .onFailure {
                    Log.e(TAG, "Failed to download ${file.name}", it)
                    downloading = false
                    // Do cleanup on failure
                    doFailureCleanup()
                    return Result.failure()
                }
            // Suspend the loop untill the download is finished
            while (downloading) {
                delay(1000)
                // Handle user-initiated cancellations as well
                if (isStopped) {
                    // Do cleanup on failure
                    doFailureCleanup()
                    break
                }
            }
        }

        if (!requestList.all { File(it.filePath).exists() }) return Result.failure()
    }

    private suspend fun downloadFile(request: Request): Result {
        return withContext(Dispatchers.IO) {
            val requestFile = File(request.filePath)
            try {
                requestFile.createNewFile()
                URL(request.url).openStream().use { input ->
                    requestFile.outputStream().use {
                        input.copyTo(it, request.size).collectLatest { p -> onProgress(p) }
                    }
                }
                // Ensure downloaded file exists
                if (!File(request.filePath).exists()) {
                    Log.e(TAG, "Failed to find downloaded file at ${request.filePath}")
                    return@withContext Result.failure()
                }
                return@withContext Result.success()
            } catch (exception: Exception) {
                Log.e(TAG, "Failed to download ${request.filePath}!", exception)
                requestFile.delete()
                return@withContext Result.failure()
            }
        }
    }

    private suspend fun onProgress(downloadInfo: DownloadInfo) {
        if (!isStopped) {
            val progress = ((downloadedBytes + downloadInfo.bytesCopied) * 100 / totalBytes).toInt()

            // Individual file progress can be negligible in contrast to total progress
            // Only notify the UI if progress is greater to avoid being rate-limited by Android
            if (progress > totalProgress) {
                val bytesRemaining = totalBytes - (downloadedBytes + downloadInfo.bytesCopied)
                val speed = if (downloadInfo.speed == 0L) 1 else downloadInfo.speed

                if (downloadInfo.progress == 100) {
                    downloadedBytes += downloadInfo.bytesCopied
                }

                val data = Data.Builder()
                    .putInt(DOWNLOAD_PROGRESS, progress)
                    .putLong(DOWNLOAD_SPEED, downloadInfo.speed)
                    .putLong(DOWNLOAD_TIME, bytesRemaining / speed * 1000)
                    .build()

                setProgress(data)
                // Update the UI notification manually using NotificationManager
                notifyStatus(progress, notificationID)
                totalProgress = progress
            }
        }
    }

}
```
You might notice that the example logic above is using a custom extension of `InputStream` as the officially available ones don't allow observing the progress. Below is the logic for that:

```kotlin
import com.example.app.data.model.DownloadInfo
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.flow
import kotlinx.coroutines.flow.flowOn
import java.io.InputStream
import java.io.OutputStream
import kotlinx.coroutines.flow.distinctUntilChangedBy
import kotlin.concurrent.fixedRateTimer

fun InputStream.copyTo(out: OutputStream, streamSize: Long): Flow<DownloadInfo> {
    return flow {
        var bytesCopied: Long = 0
        val buffer = ByteArray(DEFAULT_BUFFER_SIZE)
        var bytes = read(buffer)

        // Compute download speed every second using a Timer
        var lastTotalBytesRead = 0L
        var speed: Long = 0
        @Suppress("KotlinConstantConditions") // False-positive for bytesCopied always being zero
        val timer = fixedRateTimer("timer", true, 0L, 1000) {
            val totalBytesRead = bytesCopied
            speed = totalBytesRead - lastTotalBytesRead
            lastTotalBytesRead = totalBytesRead
        }

        while (bytes >= 0) {
            out.write(buffer, 0, bytes)
            bytesCopied += bytes
            // Emit stream progress in percentage
            emit(DownloadInfo((bytesCopied * 100 / streamSize).toInt(), bytesCopied, speed))
            bytes = read(buffer)
        }
        timer.cancel()
    }.flowOn(Dispatchers.IO).distinctUntilChangedBy { it.progress }
}

data class DownloadInfo(
    val progress: Int = 0,
    val bytesCopied: Long = 0,
    val speed: Long = 0
)
```

While updating the UI to communicate the progress is great, it should be kept in mind that

- Notifying UI from the worker should be carefully as `setProgress` is a suspend method (downloads will be suspended when its called),
- Notifying when the `Worker` has been stopped will result in bad UX,
- Updating the notification too-often will result in the app hitting the rate-limit imposed by the Android system (to prevent abuse).

Ongoing notifications cannot be dismissed on pre-Android 14.0 devices and will severaly degrade UX if the app gets rate-limited. Handle this with care, especially if you are running in the background unrestricted.

![Uncle Ben GIF](https://media.tenor.com/Hz0fHyUQzfwAAAAC/spider-man-uncle-ben.gif)

Once we have the `WorkInfo` object of our choice that we want to monitor and communicate the progress in UI, we can call [getProgress](https://developer.android.com/reference/androidx/work/WorkInfo#getProgress()) method on it, to get the progress shared from the `Worker`. The [getState](https://developer.android.com/reference/androidx/work/WorkInfo#getState()) method can be further used to handle the different situations on which we would like to do some action such as updating the UI on failure, success, etc.

```kotlin
import androidx.work.WorkInfo
import androidx.work.WorkManager
import androidx.work.WorkQuery

import com.example.app.data.work.DownloadWorker

class DownloadFragment : Fragment(R.layout.fragment_download) {

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        // Observe downloads progress by creating a WorkQuery from TAGS given to WorkRequest
        val workQuery = WorkQuery.fromTags(
            listOf(
                DownloadWorker::class.simpleName,
                downloadRequest.name,
                downloadRequest.file.versionCode.toString()
            )
        )
        WorkManager.getInstance(view.context)
            .getWorkInfosLiveData(workQuery)
            .observe(viewLifecycleOwner) { workList ->
                workList.getOrNull(0)?.let {
                    // Run custom action if download is ongoing or finished
                    if (it.state.isFinished) downloadFinished() else downloadOngoing()

                    // Update the progress bar in UI
                    when (it.state) {
                        WorkInfo.State.ENQUEUED,
                        WorkInfo.State.RUNNING -> {
                            downloadStatus = DownloadStatus.DOWNLOADING
                            updateProgress(
                                it.progress.getInt(DownloadWorker.DOWNLOAD_PROGRESS, 0),
                                it.progress.getLong(DownloadWorker.DOWNLOAD_SPEED, -1),
                                it.progress.getLong(DownloadWorker.DOWNLOAD_TIME, -1)
                            )
                        }

                        WorkInfo.State.SUCCEEDED -> {
                            downloadFinished()
                        }

                        else -> {}
                    }
                }
            }
    }
}
```

> `WorkManager` implicitly adds the `Worker`'s simple class name as a TAG to allow developers query all the work requests for a specific worker.

## Cancelling Downloads

As we are triggering downloads, its possible that we might want to cancel the download or allow user to do it (from notification or UI). `WorkManager` provides a convientant [createCancelPendingIntent](https://developer.android.com/reference/kotlin/androidx/work/WorkManager#createCancelPendingIntent(java.util.UUID)) that can create a `PendingIntent` which can be used to cancel an ongoing work from the notification.

```kotlin
val builder = NotificationCompat.Builder(context, Constants.NOTIFICATION_CHANNEL_GENERAL)
builder.setOngoing(true)
builder.setCategory(Notification.CATEGORY_PROGRESS)
builder.setProgress(100, progress, progress == 0)
builder.foregroundServiceBehavior = NotificationCompat.FOREGROUND_SERVICE_IMMEDIATE
builder.addAction(
    NotificationCompat.Action.Builder(
        R.drawable.ic_download_cancel,
        context.getString(R.string.action_cancel),
        WorkManager.getInstance(context).createCancelPendingIntent(id)
    ).build()
)
```

> This method takes the `WorkRequest`'s `UUID` and not the ID you assign when enqueuing the work. UUID can be obtained while building the `WorkRequest` or via using [id](https://developer.android.com/reference/kotlin/androidx/work/ListenableWorker#getId()) method inside the `Worker`.

In case we want to cancel the `WorkRequest` using TAGs, that's also possible using the [cancelAllWorkByTag](https://developer.android.com/reference/kotlin/androidx/work/WorkManager#cancelAllWorkByTag(java.lang.String)) and other available methods.

## Handling Failures

As it might be already apparent from some of the code examples above, we need to handle failures and cancellations related to a specific request. The simplest way is to check [isStopped](https://developer.android.com/reference/kotlin/androidx/work/ListenableWorker#isStopped()) variable every now and then and terminating the operations and doing related cleanups (which might involve deleting in-complete downloads).

![Rick and Morty GIF](https://y.yarn.co/fb9a76e5-2025-4990-a5a7-64467804a2fa_text.gif)

A suggestion would also be to have a very simple periodic work request enqueued that cleans up the downloads as we don't want to run out of storage space on the device (or bother user to do it manually). This can be [constrained](https://developer.android.com/guide/background/persistent/getting-started/define-work#work-constraints) to be only run while device is charging or idle to avoid interferring with other ongoing work.

## Reference Links

- [Platform Power Management](https://source.android.com/docs/core/power/platform_mgmt)
- [App Standby Buckets](https://developer.android.com/topic/performance/appstandby)
- [Optimize for Doze and App Standby](https://developer.android.com/training/monitoring-device-state/doze-standby)
- [Get a result from an activity](https://developer.android.com/training/basics/intents/result)
- [Support for long running Worker](https://developer.android.com/guide/background/persistent/how-to/long-running)
- [Observe intermediate Worker progress](https://developer.android.com/guide/background/persistent/how-to/observe)
- [Schedule expedited work](https://developer.android.com/guide/background/persistent/getting-started/define-work#expedited)
