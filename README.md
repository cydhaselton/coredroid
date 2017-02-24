CoreDroid - Tips and scripts to get .NET Core running on Android
================================================================

This repository contains a bunch of scripts and some information on how you can get
.NET Core to run on Android.

You can't compile .NET Core on Android (yet), so we'll cross-compile .NET Core. In practical terms,
it means compiling .NET Core on a 64-bit Linux machine and then copying the files to your Android device.

You'll need:
- A Linux machine (or VM, or Docker container) running 64-bit Ubuntu 16.04
- An Android phone or tablet, with a 64-bit ARM processor
- The Android Debug Bridge (adb) to copy files to your Android device

Ubuntu Packages
---------------

There are a couple of Ubuntu packages you need to compile .NET Core for Android, so let's get them first:

```
apt-get install -y unzip python build-essential autoconf libtool cmake git wget libicu55 libunwind8 clang-3.6
```

Repository Layout
-----------------

To get started, we're assuming you've cloned the coreclr and corefx repositories in the `~/git` folder.

If you haven't done so already:

```
cd ~
mkdir git
cd git
git clone https://github.com/dotnet/coreclr
git clone https://github.com/dotnet/corefx
git clone https://github.com/qmfrederik/coredroid
```

Initializing the RootFS
-----------------------

Both coreclr and corefx use a so-called RootFS to cross-compile for Android. In practical terms, it means
you need to run the following commands at least once to prepare your machine for cross-building for Android.

Let's get that out of the way for both coreclr and corefx:

```
cd ~/git/coreclr
cross/build-android-rootfs.sh
cd ~/git/corefx
cross/build-android-rootfs.sh
```

Cross-building coreclr and corefx
---------------------------------

```
cd ~/git/coreclr
CONFIG_DIR=`realpath cross/android/arm64` ROOTFS_DIR=`realpath cross/android-rootfs/toolchain/arm64/sysroot` ./build.sh cross arm64 skipgenerateversion skipnuget
cd ~/git/corefx
CONFIG_DIR=`realpath cross/android/arm64` ROOTFS_DIR=`realpath cross/android-rootfs/toolchain/arm64/sysroot` ./build-native.sh -debug -buildArch=arm64 -- verbose cross
```

Building Hello, World!
----------------------

First, make sure you've installed the latest .NET Core CLI:

```
cd ~/git/coredroid
mkdir cli
cd cli

wget https://dotnetcli.blob.core.windows.net/dotnet/Sdk/master/dotnet-dev-ubuntu.16.04-x64.latest.tar.gz
tar -xvzf dotnet-dev-ubuntu.16.04-x64.latest.tar.gz
```

```
cd ~/git/coredroid/apps/helloworld
~/git/coredroid/cli/dotnet restore
~/git/coredroid/cli/dotnet publish
```

Preparing the Android build of Hello, World!
--------------------------------------------

You've built a Hello World app which can run on Ubuntu. Next, let's patch that application so that it will run on
Android instead of Ubuntu:

In this step, you need to:
* Replace the copy of `mscorlib.dll` and `System.Private.CoreLib.dll` with the one you cross-built for Android
* Replace the copy of the .NET Core Runtime (coreclr) with the one you cross-built for Android
* Replace the native libraries used by .NET Core with the ones you cross-built for Android
* Remove any ngen'ed images

```
cd ~/git/coredroid
mkdir -p dist/helloworld
cd dist/helloworld
cp ~/git/coredroid/apps/helloworld/bin/Debug/netcoreapp2.0/ubuntu.16.04-x64/publish/* .
rm *.ni.dll
rm *.so
rm mscorlib.dll
rm System.Private.CoreLib.dll
cp ~/git/coreclr/bin/Product/Linux.arm64.Debug/*.so .
cp ~/git/coreclr/bin/Product/Linux.arm64.Debug/*.dll .
cp ~/git/coreclr/bin/Product/Linux.arm64.Debug/corerun .
cp ~/git/corefx/bin/Linux.arm64.Debug/native/*.so .
```

And, finally, you also need to:
* Copy the dependencies of .NET Core to the folder

So:

```
cp ~/git/coreclr/cross/android-rootfs/android-ndk-r13b/sources/cxx-stl/gnu-libstdc++/4.9/libs/arm64-v8a/libgnustl_shared.so .
cp ~/git/coreclr/cross/android-rootfs/toolchain/arm64/sysroot/usr/lib/libandroid-support.so .
cp ~/git/coreclr/cross/android-rootfs/toolchain/arm64/sysroot/usr/lib/libuuid.so.1 .
cp ~/git/coreclr/cross/android-rootfs/toolchain/arm64/sysroot/usr/lib/libuuid.so .
cp ~/git/coreclr/cross/android-rootfs/toolchain/arm64/sysroot/usr/lib/libintl.so .

```

Push your application to Android
--------------------------------

For testing purposes, let's deploy the .NET Core application to `/data/local/tmp/coredroid/`.

Make sure your devices is connected to your Linux box:

```
adb devices
```

Then, run:

```
adb shell mkdir -p /data/local/tmp/coredroid
adb push . /data/local/tmp/coredroid/
```

Finally, you are ready to start your app on Android:

```
adb shell LD_LIBRARY_PATH=/data/local/tmp/coredroid/ /data/local/tmp/coredroid/corerun /data/local/tmp/coredroid/helloworld.dll
```

Getting ready to debug
----------------------

Since .NET Core on Android is still new, you may need to attach a debugger from time to time. Luckly, Android ships with the LLDB debugger.
Here's how you install it.

PS: The 2.2 you see in the LLDB file name is an internal Google version number. It's really LLDB 4.0.0 you're installing; you can check by running `lldb-server v`.

For starters, download the latest LLDB version:

```
mkdir -p ~/git/coredroid/lldb
cd ~/git/coredroid/lldb
wget http://dl-ssl.google.com/android/repository/lldb-2.3.3420744-linux-x86_64.zip
unzip lldb-2.3.3420744-linux-x86_64.zip
```

Then, push the LLDB server (the part that will run on Android) to your Android device:

```
adb push ~/git/coredroid/lldb/android/arm64-v8a/lldb-server /data/local/tmp/
```

Next, start the LLDB server on your device:

```
adb shell
/data/local/tmp/lldb-server platform --listen *:1234
```

In another shell:

```
adb forward tcp:1234 tcp:1234
```

and then:

```
lldb-3.8
(lldb) platform select remote-android
  Platform: remote-android
 Connected: no
(lldb) platform connect connect://localhost:1234
  Platform: remote-android
    Triple: aarch64-*-linux
OS Version: 21.0.0 (3.10.61-g4ece278)
    Kernel: #2 SMP PREEMPT Wed May 25 16:16:16 CST 2016
  Hostname: localhost
 Connected: yes
WorkingDir: /data/local/tmp
(lldb) target create /data/local/tmp/coredroid/corerun
Current executable set to '/data/local/tmp/coredroid/corerun' (aarch64).
(lldb) env LD_LIBRARY_PATH=/data/local/tmp/coredroid
(lldb) run /data/local/tmp/coredroid/helloworld.dll
```

