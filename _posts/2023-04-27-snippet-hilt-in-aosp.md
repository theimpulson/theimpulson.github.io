---
title: Working with Hilt in AOSP
date: 2023-04-27 13:11:00 +0530
categories: [Engineering]
tags: [engineering]
---

Hilt is the recommended dependency injection library for Android used in most projects nowadays. Due to this, Hilt is also present in AOSP (with Dagger2). This post is a short snippet aiming to highlight how one can add Hilt support to their Android app being compiled in AOSP.

## Dependencies

Add a dependency upon the `hilt_android` static library to add Hilt support.

> You may also need to add a dependency upon the `jetbrains-annotations` library, as the generated classes seem to use annotations from it last time I checked.

```go
android_app {
    ...
    static_libs: [
        "jetbrains-annotations",
        "hilt_android"
    ],
    ...
}
```

## Annotations

AOSP doesn't have support for the Hilt Gradle Plugin. This requires us to use annotations without creating a dependency upon them.

> This is only required for specific annotations such as `HiltAndroidApp` and `AndroidEntryPoint`. The majority of the other annotations work fine.

Example usage (without the Hilt Gradle Plugin) in Kotlin:

- **Application Class**

```kotlin

import android.app.Application
import dagger.hilt.android.HiltAndroidApp

@HiltAndroidApp(Application::class)
class FooApplication : Hilt_FooApplication()
```

- **Activity Class**

```kotlin

import android.os.Bundle
import androidx.appcompat.app.AppCompatActivity
import dagger.hilt.android.AndroidEntryPoint
import org.calyxos.systemupdater.R

@AndroidEntryPoint(AppCompatActivity::class)
class MainActivity : Hilt_MainActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
    }
}
```

Remember that the annotations supported in AOSP are limited to what the `hilt_android` package provides. This means annotations provided by other dependencies, such as `HiltWorker` (for WorkManager), are absent. For them, prefer using factory methods such as `WorkerFactory` (for WorkManager) instead. Check the source for an updated list of supported annotations.

- **Example:** [WorkManager with Hilt (WorkerFactory)](https://review.calyxos.org/c/CalyxOS/platform_packages_apps_SystemUpdater/+/15730/33)

## Gradle

While supporting AOSP is excellent, using annotations as shown above (without the Gradle plugin) breaks Gradle builds. To mitigate this, disable Hilt's aggregating task in your app's `build.gradle` file.

```kotlin
hilt {
    enableAggregatingTask = false
}
```

> It is recommended to use KSP instead of KAPT. KAPT is known to have issues and requires enabling `correctErrorTypes`.

That should be all that one may need to start working with Hilt for projects compiled in AOSP.

## Reference Links

- [Official Documentation](https://developer.android.com/training/dependency-injection/hilt-android)
- [Hilt Source in AOSP](https://android.googlesource.com/platform/external/dagger2/)
- [AndroidEntryPoint - Example Usage With & Without Gradle Plugin](https://dagger.dev/api/latest/dagger/hilt/android/AndroidEntryPoint.html)
- [Gradle Setup - Hilt (Aggregating Task)](https://dagger.dev/hilt/gradle-setup.html)
