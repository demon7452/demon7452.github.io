---
layout: post
title: Google DayDream-Android篇
category: VR
tags: VR
keywords: Google DayDream Android 
description: Google DayDream Documentation For Android
---
# Google DayDream-Android篇

#### *<a href="https://developers.google.com/vr/android/" target="_blank">官方文档</a>*

## 一、Google VR SDK for Android

The Google VR SDK for Android supports both Daydream and Cardboard, including a simple API used for creating apps inserted into Cardboard viewers, and the more complex API for supporting Daydream-ready phones and the Daydream controller.

The Google VR NDK for Android provides a C/C++ API for developers writing native code.

Developers familiar with OpenGL can quickly start creating VR applications using the Google VR SDK, simplifying common VR development tasks such as:

- Lens distortion correction.
- Spatial audio.
- Head tracking.
- 3D calibration.
- Side-by-side rendering.
- Stereo geometry configuration.
- User input event handling.
	
We're keeping the hardware and software open to encourage community participation and compatibility with VR content available elsewhere.
To learn more:

- Use our Get Started guide for the Android **<a href="https://developers.google.com/vr/android/get-started" target="_blank">SDK</a>** and **<a href="https://developers.google.com/vr/android/ndk/get-started" target="_blank">NDK</a>**.
- Download the **<a href="https://developers.google.com/vr/android/download" target="_blank">Google VR SDK for Android</a>**.
- To explore the Google VR API, see the **<a href="https://developers.google.com/vr/android/reference_overview" target="_blank">Android API Reference</a>**.


## 二、Getting Started for Android SDK

This document describes how to get started using the Google VR for Android SDK by building and running one of our sample apps on your Android device.

### Treasure Hunt sample app

---

You will build the Treasure Hunt app, which uses the following features of the Google VR SDK:

- **Binocular rendering:** A split-screen view for each eye in VR.
- **Spatial audio:** Sound seems to come from specific areas of the VR world.
- **Head movement tracking:** The VR world view updates as the user moves their head.
- **Trigger input:** The user can interact with the VR world by pressing a button.

In this game, you'll look around the game world to find and collect objects as quickly as possible. It's a basic game, but it demonstrates the core features of the Google VR SDK.

### Open and run Treasure Hunt

---

#### Prerequisites

Building the sample app requires:

- Android Studio, 1.0 or higher.
- Version 23 of the Android SDK.
- Gradle 23.0.1 or higher. Android Studio will allow you to upgrade if your installed version is too low.
- A physical Android device running Android 4.4 (KitKat) or higher.

#### Download the sample code

Clone the sample app from the [gvr-android-sdk](https://github.com/googlevr/gvr-android-sdk) GitHub repository by running the following command:

```
git clone https://github.com/googlevr/gvr-android-sdk.git
```

#### Build the sample app

1.Open Android Studio. On the **Welcome to Android Studio** screen, choose **Open an existing Android Studio** project, and select the *android-sdk* directory. Android Studio will display the various gradle modules on the **Project** tab on the left side and the various run targets on the top toolbar.

![](https://developers.google.com/vr/images/android/android-studio.png)

2.Connect your phone to your machine, select the **samples-treasurehunt** target and click **Run** to compile and run the application on your phone.

For more information about the code behind the Treasure Hunt game, see our [explanation of it in the samples section](https://developers.google.com/vr/android/samples/treasure-hunt).



### Start your own project

---

#### Using Android Studio

After you've read up on the Google VR SDK for Android, it'll be time to create your own applications. Here's how.

1.First, grab all the required .AAR files from the **libraries** folder of the sdk. To determine which .AARs you need to depend on, you can examine the **build.gradle** files of the various sample apps. For example, **samples/treasurehunt/build.gradle**'s dependency section has the following entries:

```
dependencies {
  compile project(':libraries-audio')
  compile project(':libraries-base')
  compile project(':libraries-common')

  compile 'com.google.protobuf.nano:protobuf-javanano:3.0.0-alpha-7'
}
```

This indicates that an application similar to the Treasure Hunt sample needs the **audio**, **base**, and **common** libraries.

2.Create new modules for each of these libraries. Using Android Studio's GUI, this can be done via **File -> New -> New Module**. Select **Import .JAR/.AAR Package**. Locate one of the .AARs and import it.

3.Then add this new module as a dependency to your main app via **File > Project Structure > Modules (on the left side's section list) > YOUR APP's MODULE NAME > Dependencies (on the right side's tab list) -> '+' -> Module Dependency**.

Once this is done for all of the required libraries, you will be able to reference code in the Google VR SDK in your app.

#### Directly using Gradle

Using the steps above will cause Android Studio to generate new modules and edit your .gradle files automatically. Alternatively, you can directly include the .AARs in your application's module by editing that specific module's **build.gradle** file and adding the following entries:

```
dependencies {
  compile(name:'audio', ext:'aar')
  compile(name:'common', ext:'aar')
  compile(name:'base', ext:'aar')

  compile 'com.google.protobuf.nano:protobuf-javanano:3.0.0-alpha-7'
}

repositories{
  flatDir{
    dirs 'libs'
  }
}
```

This will tell Gradle to look in the **libs** subdirectory of your module for the three .AARs. Create that **libs** subdirectory inside your module's directory and copy the .AARs there. You will need to repeat this for every module that uses those .AARs which will lead to many duplicate .AARs as your project becomes more complex.

#### Using ProGuard with Google VR

If you are using [ProGuard](http://d.android.com/studio/build/shrink-code.html) with your project, make sure it keeps the code used by Google VR. We do this in our code samples **build.gradle** file with the following:

```
android {
    ...
    buildTypes {
        release {
            minifyEnabled true
            proguardFiles.add(file('../../proguard-gvr.txt'))
        }
    }
}
```

where [proguard-gvr.txt](https://github.com/googlevr/gvr-android-sdk/blob/master/proguard-gvr.txt) contains the ProGuard rules you need.
