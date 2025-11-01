---
title: Android and FOSS
description: Thoughts on recent changes in Android
date: 2025-11-01 13:50:00 +0830
categories: [Blog]
tags: [blog]
---

I started working on Android development almost a decade ago. I still vividly remember how excited I was rooting my device, using Xposed framework, and all other kinds of mods to pimp my phone. That time, I was very disappointed that my phone (Panasonic T31) wasn't going to receive any Android OS upgrades, which led me to buy my second Android phone (Asus Zenfone 2 Laser) that had an awesome FOSS community.

Fast-forward, we are at the point where Android is no longer a viable fun project by any means and arguably a mess of options when it comes to everyday non-techie users, or better called "noob". I still, almost regularly, guide people around me on how to do something simple on their devices, which is hidden deep in settings, to not install scammy malware-ridden apps from the Play Store and wade through the ad-infested world Google has created for Android users.

### Android developer verification

Recently, Google announced its new plan, which is [Android Developer Verification](https://android-developers.googleblog.com/2025/08/elevating-android-security.html). This would require every single developer who ever plans to develop an Android app to provide Google with their government-issued ID, keys, and so on. This new verification would allow Google to block apps similar to Apple has recently done [like here](https://appleinsider.com/articles/25/08/28/torrent-app-removal-proves-that-third-party-app-stores-will-not-be-free-of-apple-control) and might be something government's around the world see as a welcome move as that would allow them to ban apps they don't [like here](https://tuta.com/blog/apps-banned-india) and [here](https://apnews.com/article/apple-ice-iphone-app-immigration-fb6a404d3e977516d66d470585071bcc) for good from what the [new API](https://developer.android.com/reference/android/content/pm/PackageInstaller#DEVELOPER_VERIFICATION_FAILED_REASON_DEVELOPER_BLOCKED) indicates.

> Would a new developer still think of releasing their app first/only on a FOSS-only app store when they need to provide their ID to Google anyway, which can also open the door to the Play Store with billions of users?
{: .prompt-warning }

Lots of developers are against this move, including me. I suggest checking out the [Keep Android Open](https://keepandroidopen.org/) to read more about this topic.

### De-googled utopia

I see lots of people suggesting on various forums and social media platforms that switching to an AOSP fork would solve this verification problem and is the way to go, but I humbly disagree. Google seems to have recently taken a [complete U-turn on open-sourcing pixel and Android](https://9to5google.com/2025/06/12/android-open-source-project-pixel-change/).

Now developers need to [fill a form to request](https://source.android.com/opensourcerequest) kernel sources for their pixel devices, which is provided without any git history, where it was hosted with development history publicly accessible to anyone. On the other hand, the slow releases of source code for AOSP seem very intentional to cripple available FOSS projects that are a fork of it and aim to provide a de-googled Android.

Some of these projects have started to work together with OEMs to get source code for AOSP on time to deal with security issues instead of waiting forever for empty ASB bulletins and missing sources, though I doubt that is something every AOSP-based project out there can do, especially since you may need to sign an NDA.

> What if tomorrow these OEMs amend (intentionally or get forced) their NDA to not provide any sources related to the OS? What then? How would your favorite de-googled project survive and be the solution you dreamed of?
{: .prompt-warning }

### Linux distributions

I see Linux distributions such as [Ubports](https://ubports.com/) and [postmarketOS](https://postmarketos.org/) as a very good real alternative for those looking to move out and away from the Android and iOS platforms. Be warned, there is a lack of apps, and the fact that the majority of the supported devices were launched with Android is something you may want to be concerned about, as ultimately that support might not come for new devices if there is no AOSP.

> Linux distributions work on devices launched with Android thanks to [Halium](https://halium.org/). Halium builds upon LineageOS, which is in turn based upon AOSP.
{: .prompt-info }

Of course, these projects aren't funded by big corporate companies, resulting in a small number of supported devices and the resulting userbase, but they are open and are welcoming to everyone looking for a FOSS OS for their smartphones.

### What would I do?

I am highly considering picking up a dumbphone. Putting all your eggs in one basket was a mistake I felt I and a lot of people out there have made in the past few years. I have already moved my gaming needs to [Steam Deck](https://www.steamdeck.com/) and studies to [reMarkable](https://remarkable.com/), both of which run Linux and allow me to do whatever I want to do with the hardware and the running software. I am again picking up the habit of using cash to pay in real life (instead of banking apps, which hate FOSS OS), buying and renting books from the library and small book shops, and buying and downloading my music locally. I hope by next year I can totally either switch to a phone running a Linux distribution or a feature phone. 
