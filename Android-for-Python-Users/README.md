Android for Python Users
========================

*An unofficial Users' Guide*

Revised 2021/03/19

# Introduction

Python and Kivy are portable across operating systems because of POSIX, Python wheels, and pip. However Android is not POSIX compliant, wheels are not usually available for Android, and pip is not installed on Android. Clearly many apps won't 'just work' on Android. The document is about porting a Kivy app to Android.

For a well written app that only paints the screen, and does nothing else, building for Android will be 'push button'. One can for example build 'Hello World' with the default 'buildozer.spec' file. Of course this does not describe many apps.

This document is not a substitute for Reading The Fine Manual [(RTFM)](https://en.wikipedia.org/wiki/RTFM). Your basic resources are [Buildozer](https://github.com/kivy/buildozer/tree/master/docs/source), [Python for Android](https://github.com/kivy/python-for-android/tree/develop/doc/source), and [Android](https://developer.android.com/guide).

We assume android.api >= 29

Python-for-Android is a truly amazing achievement, but there are some details to understand. The alternative is Java and the Android api, which has been shown to induce insanity in laboratory mice. 

Lastly this document is my understanding as of the date above. I am not a developer in any of the Kivy sub projects (except for one PR in p4a). The document is guaranteed to be incomplete is some way, and may possibly be wrong in another way. But reading it will hopefully provide you with come context with which to read the official documentation and to experiment. Those two steps are the key to insight and understanding, which in turn is key to making your idea real.

# What is Different about Android?

## Posix

   Android is not POSIX compliant (POSIX is so deep in your assumptions about computers you probably don't know it exists).
   
The file system model is different, the app cannot use any directory in the file system. The app has specific private storage directories that support file operations. The app also has storage shared between apps, this is implemented as a database.

Threads are available and unlike the desktop, run on all available cores. This is cool. But a subprocess can't execute an app, and there is no command shell to run inside a subprocess.

Multi-tasking; on the desktop when an app loses focus or is minimized it continues to execute - it just doesnt see UI events. Android is not multi-tasking, when an app is removed from the UI it *pauses*, and then does not execute any app code. Execution without a UI requires an Android service.

## Wheels

Some Python packages are not written in Python (are not pure-Python), they contain code that must be compiled. Pip provides pre-compiled packages for desktop OSes, but not for Android. P4a addresses this with [recipes](https://github.com/kivy/python-for-android/tree/develop/pythonforandroid/recipes), but not all impure packages are available. AVOID DISAPPOINTMENT, check availability first.

## Meta-information

Unlike the desktop you must provide information *about* your Python code, this requirement causes everybody who doesn't understand it to crash and burn. This information must be supplied in the buildozer.spec file. It includes *all* the pip packages your app depends on, any file types your app depends on for data, and the Android permissions your app depends on. Meta-information may not be limited to that short list, but that list is critical.

# Android Storage

Android's view of its file system has changed a few times over the years. The result is a messy pile of jargon: local, system, internal, external, primary, secondary, scoped, all describe views of storage.

However in the storage model described here storage is either *Private Storage* or *Shared Storage*. Private storage content is only visible to the app that created it, shared storage is visible to all apps. This model is valid if the build api>=29.

Physically storage may be 'internal' or 'external', this is a somewhat historical view where 'external' represents the (on older devices possibly removable) sdcard. The apis referenced here all default to 'external', with 'internal' as an option.

## Private Storage

An app can perform Python file operations (read, write, shutil) on its private storage. Files in Private Storage are persistent over the life of the app, they are retained when the app is updated, and deleted when the app is uninstalled.

The install directory './' is also private storage, but files do not persist between installs and updates. Only use this directory for reading data files packaged in the apk.

Do not confuse private with secure, on older Android versions it is possible for other apps to read private storage. In this case internal storage is more secure than external storage.

No permissions are requires to read or write an app's Private Storage. [More on permissions below](#app-permissions)

## Shared Storage

Shared storage is visible to all apps, is persistent after the app is uninstalled, and implemented as a database. The nearby [storage example](https://github.com/RobertFlatt/Android-for-Python/tree/main/storage) demonstrates the database insert(), delete(), and retrieve() operations.

Shared storage is organized based on root directories located in Android's 'Main Storage'. They are 'Music', 'Movies', 'Pictures', 'Documents', and 'Downloads'.

No permissions are requires to read or write an app's own Shared Storage. Reading another app's Shared Storage requires READ_EXTERNAL_STORAGE permission.

On devices running api < 29 the MediaStore database is flat, there are no simulated directories. The app designer has a choice to use this or the traditional file based primary shared storage. The best choice depends on the application. The MediaStore approach enables a share between apps (see next section), and data transfer is transparent when the user upgrades their device to 29 or greater. The file system approach provides the user with the familiar hierarchy, and requires WRITE_EXTERNAL_STORAGE permission.

## Sharing a file between apps

Files in Shared Storage can be shared between apps. Either using the file picker, or the Share mechanism. See the examples: [Sending a Share](https://github.com/RobertFlatt/Android-for-Python/tree/main/share_snd) and [Receiving a Share](https://github.com/RobertFlatt/Android-for-Python/tree/main/share_rcv).

# Threads and Subprocesses

Threads are available and are have more utility than on the desktop because they are executed on all available cores. The single core thread implementation on desktops allows lazy assumptions about thread result ordering. These will bite you on Android if the code is not written in a thread safe manner. Always use callbacks.

Kivy executes on the 'UI thread', Android requires that this thread is always responsive to UI events. As a consequence long latency operations (e.g. network access, sleep()) or computationally expensive operations must be performed in their own Python threads. Threads must be truly asynchronous to the UI thread, so do not join() in the UI thread. A non-UI thread may not write to a UI widget. A very thread safe way to return results to the UI thread is to use Clock_schedule_once().

The Python subprocess is not available. The Android equivalent is an Activity, an Activity with no UI is a Service. An Activity can be created through Pyjnius and Java, by first creating an Intent. The special case of creating an Android Service can be automated using Buildozer.

# App Permissions

Android restricts access to many features. An app must declare the permissions it requires. There are two different declarations, manifest and user. User permissions are a subset of manifest permissions.

Manifest permissions are declared in the buildozer.spec file. Common examples are  CAMERA, INTERNET, READ_EXTERNAL_STORAGE, RECORD_AUDIO. The [full list is](https://developer.android.com/reference/android/Manifest.permission). Apps that scan Bluetooth or scan Wifi may require multiple permissions. In general you must research the permissions needed, resist the temptation to blindly guess.

Python for Android always enables manifest WRITE_EXTERNAL_STORAGE permission. WRITE_EXTERNAL_STORAGE implies READ_EXTERNAL_STORAGE. [WRITE_EXTERNAL_STORAGE is never required](https://developer.android.com/training/data-storage#permissions) for devices running api >= 30.

Any app manifest permission documented in that list as having "Protection level: dangerous" additionally require a user permission. The four listed above are all "dangerous". User permissions generate the dialog that Android apps present to the user. In a Kivy App this must be called from the `build()` method using `request_permissions()` which is part of the `android.permissions` package.

See any of the nearby examples.

# Buildozer and p4a

## Install
[RTFM](https://github.com/kivy/buildozer/blob/master/docs/source/installation.rst), really. Errors during a Buildozer build are usually a failure to [read the install instructions](https://github.com/kivy/buildozer/blob/master/docs/source/installation.rst), or an attempt to build an impure Python package.

Buildozer runs on Linux, Windows users need a Linux virtual machine such as WSL or VirtualBox to run Buildozer.

Buildozer's behavior can be non-deterministic in any of these cases:

It is run as root

It is run on an NTFS partition mounted on a Linux system.

There are Python style trailing comments in the buildozer.spec


## Changing buildozer.spec

Note that Buildozer allows *specification* of build files, versions, and options; but unlike most other build tools it *does not do version management*. If buildozer.spec is changed the change probably *won't* propagate into the apk on the next build. After changing the buildozer.spec file users *must* do an appclean.
```
buildozer appclean
buildozer android debug
```
There may be some exceptions to this, the only one I know to be safe is one can add (but not change version of, or remove) a package in the [requirements](#requirements) list without the appclean.

There is no magic universal buildozer.spec, its configuration depends on the functionality of your app. 

## Some buildozer.spec options

[RTFM](https://github.com/kivy/buildozer/blob/master/docs/source/specifications.rst), really. And see the [KivyMD section](#kivymd).

### requirements

This is basically the list of pip packages and the Python version that your app depends on. It is important that you understand what your app depends on. The current Buildozer default version for Kivy is obsolete, change it to:
```
requirements = python3,kivy==2.0.0
```
The list must be complete, to miss one item will be fatal. Some packages don't automatically references their dependencies, so these will have to be added as well. A common worst case example is adding 'requests' which becomes:
```
requirements = python3,kivy==2.0.0,requests,urllib3,chardet,idna
```
Be careful that the packages you add here are pure Python. If the package is not pure Python and does not have a [recipe in this list](https://github.com/kivy/python-for-android/tree/develop/pythonforandroid/recipes) then there is an issue. The options are to either rewrite the app, locally modify an existing recipe [see Appendix C](#appendix-c-locally-modifying-a-recipe), [create a new recipe](https://github.com/kivy/python-for-android/blob/develop/doc/source/recipes.rst), or import the functionality from Java. None of these options are trivial. That is why it said AVOID DISAPPOINTMENT in [the Wheels section above](#wheels).

### source.include_exts

If the app contains any data files, for example icons. Then add the file extension to this list:
```
source.include_exts = py,png,jpg,kv,atlas
```

### android.permissions

Research the Android permissions your app needs. For example
```
android.permissions = INTERNET, CAMERA, READ_EXTERNAL_STORAGE
```

### android.api

The current buildozer default is 27, but should be "as high as possible". 
```
android.api = 30
```

### android.minapi

Python for Android enables `android.minapi = 21`.

### android.ndk

Probably best not to change this from the current 19c. But if there is some reason you really need to, 21d also appears to work.

### android.arch

Currently defaults to 32-bit ARM. For performance improvement, if you have a 64 bit device, change this to:
```
android.arch = arm64-v8a
```
An install message INSTALL_FAILED_NO_MATCHING_ABIS means the apk was built for a different architecture than the phone or emulator.

# Debugging

On the desktop your friends are the Python stack trace, and logging or print statements. It is no different on Android. To get these we [run the debugger](https://github.com/kivy/buildozer/blob/master/docs/source/quickstart.rst#run-my-application).

First connect the device via USB, on the Android device enable 'Developer Mode' and 'USB debugging'.

If Buildozer was run on a virtual machine then it may not be able to use the the physical USB port and the 'deploy run logcat' options will not work. [In this case use adb instead.](#appendix-a-using-adb) Successful setup is indicated by log output similar to:
```
List of devices attached
0A052FDE40019P  device
```
If 'List of devices attached' is followed by an empty line then the connection failed, regardless of what the Buildozer log say afterwards. Either because the device debug options are not set or the debugger is run from a virtual machine.

In the logcat output look for 'Traceback', what follows is a Python stack trace, which usually indicates the cause of the issue. For example:
```
ModuleNotFoundError: No module named 'some-import-name'
```
Where 'some-import-name' is in 'some-pip-package', then this pip package name is missing from [buildozer.spec requirements](#requirements).

It is possible to [debug using an emulator](#appendix-b-using-an-emulator) but this is not recomended initially, as it adds unknowns to the debug process. The emulator is useful for checking a debugged app on various devices and Android versions.

# Some Related Topics

## Layout

A portable layout is not an Android specific issue. In general for Kivy apps use [density-independent pixels](https://kivy.org/doc/stable/api-kivy.metrics.html) 'dp' to specify an absolute size, and scale-independent pixels 'sp' to specify a font.

## KivyMD

The KivyMD widgets have the look and feel that Android users expect, but the Material Design rules mean you don't have the same flexibility as Kivy widgets.

KivyMD is in development, which means some functionality [is still changing](https://kivymd.readthedocs.io/en/latest/changelog/index.html). Next time KivyMD is downloaded the version number may be the same, but some widget may be different!

[How to use KivyMD with Buildozer](https://github.com/kivymd/KivyMD/blob/master/README.md#how-to-use-with-buildozer). There may be additional Buildozer settings required for KivyMD, see KivyMD's sample [buildozer.spec](https://github.com/kivymd/KivyMD/blob/master/demos/kitchen_sink/buildozer.spec).

## Camera

The Kivy Camera widget does not work on Android, neither does the OpenCV camera. Try the [Xcamera widget](https://github.com/kivy-garden/xcamera) from the Kivy Garden, it still has issues but is currently the only choice for an camera preview as part of a layout. Another option is [CameraXF](https://github.com/RobertFlatt/Android-for-Python/tree/main/cameraxf), a turnkey full screen photo, video, and image analysis camera.

## Back Button and Gesture

On Android a back button/gesture pauses an app. From Kivy it does not appear possible to overload this behavior. Anybody know a way?

## Kivy Lifecycle

Follow the [Kivy Lifecycle](https://kivy.org/doc/stable/guide/basic.html#kivy-app-life-cycle), it abstracts the app behavior that Android expects.

Do not place code in the app that interacts with Android 'script style', to be executed before the Kivy build() call.

`request_permissions()` must only be called from the App's `build()` method, and only one once with an argument that is a list of all required permissions.

The App's `on_stop()` method is not always called, use `on_pause()` to save state.

## The Android package

P4a provides Android specific utilities in the android package, this is only available on Android. It is as they say 'self documenting', which really means there isn't any documentation. [Read the code](https://github.com/kivy/python-for-android/tree/develop/pythonforandroid/recipes/android/src/android).

## Plyer

Plyer is an OS independent api for some non-POSIX OS features. See [available features](https://github.com/kivy/plyer#supported-apis).

The [Plyer examples](https://github.com/kivy/plyer/tree/master/examples) are the documentation. Some Plyer examples work on Android but not all Android versions, for example Camera, Speech to text, Storage. Also some run time permissions are missing.

If you plan to use Plyer, and the idea of it is very appealing, first try a small test case on your target Android versions. If it does what you expect, check that all the features you need are available; as Plyer has a platform lowest common denominator design.


## Pyjnius

[Pyjnus](https://github.com/kivy/pyjnius/tree/master/docs/source) allows import of Java code into Python code. It is an interface to the Java api and the Android api. The Android api is only available on Android devices, Android api calls must be debugged on Android.

```python
from jnius import autoclass
# declare a Java Class
DownloadManager = autoclass('android.app.DownloadManager')
# Java sub classes are delimited by a '$' in place of a '.'
DownloadManagerRequest = autoclass('android.app.DownloadManager$Request')	

   # get a class constant
   visibility = DownloadManagerRequest.VISIBILITY_VISIBLE_NOTIFY_COMPLETED
   # instance a Java class
   self.request = DownloadManagerRequest(uri)
   # call a method in that class
   self.request.setNotificationVisibility(visibility)
```

Then use this to write code with Python syntax and semantics, and Java class semantics added. Some basic knowledge of Java semantics is required, get over it. Android classes will require (possibly extensive) reading of the [Android Developer Guide](https://developer.android.com/guide) and [Android Reference](https://developer.android.com/reference).

It is also possible to write Java class implementations in Python, [RTFM](https://github.com/kivy/pyjnius/blob/master/docs/source/api.rst#java-class-implementation-in-python) and [look at some examples](https://github.com/RobertFlatt/Android-for-Python/blob/main/cameraxf/cameraxf/listeners.py).

It is not possible to import Java *abstract* classes, as they have no *implementation* (abstract and implementation are Java keywords). And it it is not possible to provide the implementation in Python. You must write the implementation in Java and import that.

One thing to watch out for is you will be using two garbage collectors working on the same heap, but they don't know each other's boundaries. Python may free a local reference to a Java object because it cant see that the object is used. Obviously this will cause the app to crash in an ugly way. So use class variables, as in the example above, to indicate persistence to the Python garbage collector.

Python for Android builds an apk with a minimum device api. Importing Java modules can invalidate this minimum. Check the [Added in API level field](https://developer.android.com/reference/android/provider/MediaStore.Downloads) in the class or method reference documentation.

Note: some documentation examples are obsolete. If you see '.renpy.' as a sub field in an autoclass argument replace it with '.kivy.'.

## Kivy Garden

[Kivy Garden](https://github.com/kivy-garden/) is a library of components ('flowers'). It is mostly not maintained. Anybody who has had a garden knows a garden needs a gardener, Kivy Garden doesn't have one. Kivy Garden is a useful resource, it is mostly best used as examples to copy and modify (fix) rather than components you can install and instantiate.

## Android for Python

[Android for Python](https://github.com/RobertFlatt/Android-for-Python) contains examples of some Android features as used from Python (and this document). These examples only run on Android.

## How to create a Release APK

[The instructions are here](https://github.com/kivy/kivy/wiki/Creating-a-Release-APK) but don't just follow the instructions, read all the annotated comments by HeRo002. The instructions are flawed, but in combination with the comments they are good.

## Android Store

Always build with the latest api and arm64-v8a when building for the Store.

Apparently one can upload an armeabi-v7a apk to the play store, but you must first upload an arm64-v8a version of the app.

HeRo002 tells us what to [expect](https://github.com/kivy/buildozer/issues/1290).

The store will soon require that apps are submitted as an [app bundle](https://developer.android.com/guide/app-bundle). Presumably this measn unzipping our apk(s) and rebuilding with bundletool. If anybody want to figure out the details of how to this, that would be great. Please publish instructions or a script.

# Appendix A Using adb

The easiest way to get adb is to install Android Studio.

Add something like this to your PATH:
     C:\Users\UserName\AppData\Local\Android\Sdk\platform-tools

Some adb commands:
```
adb devices
adb install -r  cameraxf-0.1-arm64-v8a-debug.apk
adb logcat -c
adb logcat > log.txt
```

# Appendix B Using an emulator

Install Android Studio.

In Android Studio, go to Tools->AVD Manager

Use the 'Create Virtual Device..' button if the emulator you want is not listed.

Right click on an emulator (view details) to see it's name, for example: Nexus_4_API_21

Add something like this to your PATH:
     C:\Users\UserName\AppData\Local\Android\Sdk\emulator

Start an emulator using it's name
```
emulator @Nexus_4_API_21
```
Check the emulator is running using 'adb devices'. Then install an app in the emulator from adb. The apk MUST be built with android.arch set to the same as ABI above.

# Appendix C Locally modifying a recipe

Modify an existing recipe by making a local copy.
Replace RECIPE_NAME with whatever recipe you are changing:

1) in buildozer.spec set `p4a.local_recipes =  ./p4a-recipes`

2) `mkdir ./p4a-recipes`

3) [Download python-for-android](https://github.com/kivy/python-for-android)

4) Copy the recipe you want from python-for-android/tree/develop/pythonforandroid/recipes/RECIPE_NAME to ./p4a-recipes

5) Change the files in a way that makes you happy

6) buildozer appclean

7) buildozer android debug


