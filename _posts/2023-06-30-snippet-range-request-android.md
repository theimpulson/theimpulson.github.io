---
title: Using Range Requests in Android
date: 2023-06-30 20:18:00 +0530
categories: [Engineering]
tags: [engineering]
---

While working with big files, there are times when we would like to access a specific part of it. This is usually not an issue when the file is present locally, as devices nowadays have an incredible r/w speed allowing us to quickly parse the file in question. However, it becomes completely different when the file is hosted on a remote server. In that case, parsing/draining the big file isn't a good option as that can lead to a waste of data and be incredibly slow, depending upon the speed of the data connection.

> Accessing a file is usually done with a [ContentUris](https://developer.android.com/reference/android/content/ContentUris) nowadays, as that works everywhere (but doesn't gives us an absolute file path). We can open an [InputStream](https://developer.android.com/reference/java/io/InputStream) to read data from it and [OutputStream](https://developer.android.com/reference/java/io/OutputStream) to save it into another file.

## What is the range request

Quoting from Mozilla's article [here](https://developer.mozilla.org/en-US/docs/Web/HTTP/Range_requests):

> An HTTP range request asks the server to send only a portion of an HTTP message back to a client. Range requests are useful for clients like media players that support random access, data tools that know they need only part of a large file, and download managers that let the user pause and resume the download.

The ability to request only a portion of the file makes the range request useful. However, the **[Range](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Range) header that is used for this functionality requires us to specify the exact start and end `bytes` of the file which we want to request** and **the server must also support the range request functionality**.

## Working with files using range request

Reading a file from a remote server using a range request is quite easy. While opening an [HttpsURLConnection](https://developer.android.com/reference/javax/net/ssl/HttpsURLConnection) or [HttpURLConnection](https://developer.android.com/reference/java/net/HttpURLConnection), specify the [Range](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Range) header.

> Making network request and working with them can block the main thread is not recommended. Use Kotlin Coroutines's [IO](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-i-o.html) dispatcher for such purposes.

> The output might have whitespaces and multiple lines per the case. Ensure to handle those after reading the said data.

```kotlin
try {
    val connection = URL(url).openConnection() as HttpsURLConnection
    // Request a specific range to avoid skipping through load of data
    connection.setRequestProperty(
        "Range",
        "bytes=${packageFile.offset}-${packageFile.offset + packageFile.size}"
    )
    val properties = connection.inputStream.bufferedReader().use { it.readText() }
} catch (exception: Exception) {
    Log.e(TAG, "Failed fetching properties!", exception)
}
```

Similarly, you can also download parts of the file by saving the data from [InputStream](https://developer.android.com/reference/java/io/InputStream) to an [OutputStream](https://developer.android.com/reference/java/io/OutputStream) of a local file.

> The system/library might automatically append or remove additional bytes during download. Handle those manually by incrementing or decrementing as per the use case. The example below already does such for convenience purposes.

```kotlin
val metadataFile = File("${context.filesDir.absolutePath}/${packageFile.filename}")
try {
    metadataFile.createNewFile()
    val connection = URL(url).openConnection() as HttpsURLConnection
    // Request a specific range to avoid skipping through load of data
    // Also do a [-1] to the range end to adjust the file size
    connection.setRequestProperty(
        "Range",
        "bytes=${packageFile.offset}-${packageFile.offset + packageFile.size - 1}"
    )
    connection.inputStream.use { input ->
        metadataFile.outputStream().use { input.copyTo(it) }
    }
} catch (exception: Exception) {
    Log.e(TAG, "Failed to download metadata file! ", exception)
}
```

That should be all one may need to read and download files using range request on Android.

## Reference Links

- [Mozilla Documentation](https://developer.mozilla.org/en-US/docs/Web/HTTP/Range_requests)
- [Helpful Link - StackOverflow Q/A on why ZipInputStream's skip method is a bad choice for large remote files](https://stackoverflow.com/a/67304394/8446131)
- [Helpful Link - CalyxOS's SystemUpdater app that uses range request to read and download files](https://gitlab.com/CalyxOS/platform_packages_apps_SystemUpdater)