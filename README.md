# Android - patching Apps
One think you sometimes like to do is patching Apps - you can do that dynamically using tools like [Frida](https://github.com/frida/frida).  
Dynamic instrumentation is excellent but has strong requirements (often requires rooting your phone), so one thing one could do is patch Apps statically.  
Of course, if you want to patch native code (called via JNI) you can easily patch the `*.so` files in the App and repackage, but how do you patch the Java bytecode?  
For that, we'll introduce how Java Apps bytecode looks like, how to patch it, repackage the App etc., without rooting your phone.

## Legal notice
You should only patch Apps that you own or have permissions to patch.  
This guide is only going to be used for information on how to patch Apps, and in no way encourages bypassing DRM or creating fake Apps.  
In other words - use at your own risk!

## Tooling
There are many tools out there, but my favorite is probably [apktool](https://apktool.org).  
We'll need a few tools to resign the App, and maybe understand *where* to patch. So:
- [apktool](https://apktool.org) is going to be used for decoding and rebuilding the App.
- [keytool](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/keytool.html) that comes with JDK would allow us to re-sign the App.
- [adb](https://developer.android.com/tools/adb) is the Android Debug Bridge and will allow us to debug Apps.

### Installations
- `apktool` is quite easy to install - it's a `JAR` file so it can be run with Java. However, you can install it on a Linux box easily with `apt`.
- For `keytool`, install the default `JDK`.
- For `adb`, install the Android debugging tools. Again I will be using `apt` for that on a modern Linux, but you can easily install those on Windows too.

```shell
# apktool
sudo apt install apktool

# JDK
sudo apt install default-jdk

# adb
sudo apt install android-tools-adb android-tools-fastboot
```

