---
title: Rewriting Exodus Privacy's Android App
date: 2022-10-14 18:00:00 +0530
categories: [Blog]
tags: [event, blog]
---

Last year while browsing my Twitter feed, I stumbled on a [tweet from Exodus Privacy](https://twitter.com/ExodusPrivacy/status/1462533262279073795) looking for help with their Android app, which was removed from the Google Play Store. This article aims to share how the said app was reinstated in the store but also had multiple improvements it was done.

# Issue

The team had opened an [issue on their GitHub regarding the Google Play Store requirement](https://github.com/Exodus-Privacy/exodus-android-app/issues/111). As per the issue, **the app was missing a valid privacy policy and prominent disclosure** regarding the fact that it was uploading the app details to a remote server.

There are a couple of essential points to note here, the important one being that the Exodus Privacy app doesn't perform the analysis on the device but on the project's server and stores the results there. Instead, the app collects all the installed packages and queries the server regarding them to obtain and show the relevant data to the user. Unfortunately, the app lacked a valid privacy policy and a prominent disclosure of this fact, which led to the issue.

# First Attempt & Failure

After a short discussion with the team, I decided to pick this issue. We concluded by adding a link to the privacy policy in the app by adding a new fragment to notify the user about it. This was relatively simple compared to the codebase's situation at that time (more on it below). I opened a [pull request](https://github.com/Exodus-Privacy/exodus-android-app/pull/128), and after review, it was merged. Everything should be good now, right? Unfortunately, it wasn't. After submitting the app Google Play Store, it was rejected again for the same reason as before; it was missing a prominent disclosure. After rereading the support documents, we figured out what the issue was.

The document states: **Your in-app disclosure must accompany and immediately precede a request for user consent and, where available, associated runtime permission. You may not access or collect personal and sensitive data until the user consents.**. This meant that we had to show an explicit dialog box informing the user on what the app was doing (collecting all app's names and communicating with Exodus Privacy's server). Without the user's explicit approval, we were not permitted to collect the app details and communicate with the remote server.

> In case you are wondering why explicit approval from the user was required, the app collected information about all applications and uploaded it to a remote source. Google limits this on newer API levels by [package visibility filtering on Android](https://developer.android.com/training/package-visibility). Using this permission is subject to the [new policy requirements](https://support.google.com/googleplay/android-developer/answer/10158779?hl=en#zippy=%2Cinvalid-uses%2Cexceptions%2Cpermitted-uses-of-the-query-all-packages-permission) for publishing the app on the Google Play Store as well. So even when the app was not using the said permission, it was behaving in a way that bypassed this requirement.

# The Great Kotlin Rewrite

At this point, we were forced to add an alert dialog at the app startup to ask user permission regarding the process. However, it was not by any means as easy as it sounds. Some of the significant issues in the existing codebase were:

- Deprecated methods and libraries such as ViewPager to change between different fragments,
- Navigation was implemented in a complicated way (no navigation components implementation),
- Working with SQLite databases, markdown and network calls directly without any convenience library,
- No abstraction between the Views and Model classes.

Apart from these, the app also used an outdated design and theme. These issues were making the required changes very difficult. After discussing with the team, everyone agreed that the app needs to be rewritten from scratch following the recommended practices today. To help with the UI design, one of the members of the Exodus Privacy team, [Jean-Baptiste](https://twitter.com/JbCHARRON88), designed mockups of the app in [Material3](https://m3.material.io/) UI in [Figma](https://www.figma.com/proto/D5dSSeiAvCwdeBDVVKT9ME/Exodus).

At this point, I started working on rewriting the app in Kotlin. I decided to follow the [Single Activity Architeture](https://www.youtube.com/watch?v=2k8x8V77CrU) pattern that involves using [navigation components](https://developer.android.com/jetpack/androidx/releases/navigation) library. This made the navigation easy between different screens and allowed better control of the overall navigation, which was required to fix the prominent disclosure issue. Thus, a DialogFragment was added to inform the user about the process to resolve it.

![Image of dialog](/assets/img/posts/exodus_dialog.png){: w="500" h="280" }

For communicating with the [Exodus REST API](https://github.com/Exodus-Privacy/exodus/blob/v1/doc/api.md), I decided to use the [Retrofit](https://square.github.io/retrofit/) as it integrates nicely with [coroutines](https://developer.android.com/kotlin/coroutines). As for the local database to store the fetched results, I went with the [room](https://developer.android.com/jetpack/androidx/releases/room) library, recommended by Android. These libraries allowed me to rewrite the entire backend part of the app nicely. Finally, I went with the [Hilt](https://dagger.dev/hilt/) library to inject the dependencies of the database and network classes.

The REST API sends details of the trackers in markdown format. In the old version, parsing was done manually, which wasn't performing well and took a lot of work to maintain. During the rewrite, I searched for a library that would allow me to parse markdown and ended up using [Markwon](https://github.com/noties/Markwon). While easy to work with, the development of the library seems halted. Still, it felt a lot better than trying to parse markdown manually.

The process of getting details of each app present on the device from the server can be a long-running one. To ensure it works smoothly and remains decoupled from the UI, I used a foreground service for fetching and saving the reports in the database.

The rewrite was by no means easy and quickly done. It took almost 3 quarters as everyone was busy in real life as well. [Jean-Baptiste](https://twitter.com/JbCHARRON88) was quite helpful in reviewing and testing changes done by me everyday. His feedback gave me ideas to improve code quite a lot and squash bugs I failed to identify. After I got too busy and left the rewrite task, he tookover the lead and finished it quite nicely.
[Kunzisoft](https://twitter.com/KunziSoft) was also kind enough to help with the rewrite and addressed some of the remaining parts.

# Submitting on Google Play Store Again

After the rewriting task was finished, team members opened and reviewed a [pull request](https://github.com/Exodus-Privacy/exodus-android-app/pull/197). It passed the required testing and was merged into the repository's default branch. After that, the team submitted the app to the Google Play Store again, and this time it was [approved without any issues](https://twitter.com/ExodusPrivacy/status/1565655787673792514).

The new app is live right now on both the [Google Play Store](https://play.google.com/store/apps/details?id=org.eu.exodus_privacy.exodusprivacy) and [F-Droid](https://f-droid.org/packages/org.eu.exodus_privacy.exodusprivacy/). The team worked for quite a long time and improved it. The new app is not only compliant with all the stores' policies but is also following the latest UI/UX guidelines to offer a great experience to the users. Feel free to check it out!

# Reference Links

- [Exodus Privacy's Website](https://exodus-privacy.eu.org/en/)
- [Exodus - Android Application](https://github.com/Exodus-Privacy/exodus-android-app)
