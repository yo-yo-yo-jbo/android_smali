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

## Introduction to SMALI
As I mentioned in [a previous blogpost](https://github.com/yo-yo-yo-jbo/android_appsec_intro/), Android Apps are bundled as `*.apk` files.  
Those are essentially zip files, with a predefined structure:
- A metadata file called `AndroidManifest.xml`.
- Bytecode files: those are `*.dex` files - you will most likely examine `classes.dex` even though `classes-2.dex` (etc.) files do exist.
- Native libraries: will be `*.so` files under `lib/`.
- Resources and assets: such as pictures, audio and others - will be under `res/` and `assets/`, with the difference being resources are compiled into a class (named `R`) and assets are "raw".

Since we will be focusing on modifying the bytecode, `classes.dex` is our primary target, but we will have to start with `AndroidManifest.xml` to understand what we'd like to focus on in the App.
I did not mention it previously, but when treating `*.apk` files as archives - this file is binary (even though it has a `.xml` extension). Tools like `apktool` can convert it to a readable form - more on that later.

Tools like `apktool` will extract all definitions and code in `classes.dex` and extract them to files with the extension `*.smali`.  
Each file will commonly be a Java class, and contain the definitions of the class, as well as the Android bytecode.  
Thus, let us start by describing the bytecode structure itself in each `*.smali` file.

So, some starting points:
- In Smali, registers are 32-bit long and hold 32-bit values. To handle 64-bit values (e.g. Java `long` or `double` instances), two registers are used.
- Smali files contain declaration of a class (with a `.class` directive), that includes information about the class - its superclass, its members and methods. Our patching will mostly focus on methods.
- Methods can get parameters in registers, and will be references as `p0`, `p1` and so on. Note that non-static methods will have the Java `this` instance be assigned to `p0`!
- Each method declares how many registers it uses. There are two ways to do so - `.registers` (total number of registers) and `.locals` (the number of non-parameter registers).
- Method declaration contains complete type information (including the return type and the input types). Each type can be a primitive type or a class instance. Return type can also be `V` (void).
- All method invocations also contain type information about the called method and resulting type. It makes sense since in Java you can have multiple methods with the same name, differentiated by argument types.
- Stack is heavily used: arguments are pushed into the call stack, the stack is used to sometimes handle large data, exception handling uses the stack and so on.

Basic types supported are the following:

Java type      | Smali     | Description           | Remarks
---------------|-----------|-----------------------|--------------------------------------------------
boolean        | Z         | Java `false`or `true` |
char           | C         | Single character      |
byte           | B         | Single byte           |
short          | S         | Signed 16-bit integer | 
int            | I         | Signed 32-bit integer | 
long           | J         | Signed 64-bit integer | Uses 2 registers
float          | F         | 32-bit IEEE754        |
double         | D         | 64-bit IEEE754        | Uses 2 registers
Class instance | Lclass;   | Class instance        | Example: `Ljava/lang/String;` is a `String` type

## Toy example
Let us start with a toy example - I have uploaded an [toy.apk](toy.apk) to this repository.  
It has one activity (`MainActivity`) that will call a method called `showToast`, and here is the relevant source code:

```java
private void showToast() {
    Random random = new Random();
    boolean randomBoolean = random.nextBoolean();
    String title = getTitle(randomBoolean);
    Toast toast = Toast.makeText(getApplicationContext(), title, Toast.LENGTH_SHORT);
    toast.show();
}

private static String getTitle(boolean showName) {
    if (showName) {
        return "JBO (@yo_yo_yo_jbo)";
    }
    else {
        return "https://jonathanbaror.com";
    }
}
```

Let's run `apktool` with the `d` flag (for decoding) and examine the output directory:

```
$ jbo@nix:~ apktool d ./toy.apk
I: Using Apktool 2.5.0-dirty on toy.apk
I: Loading resource table...
I: Decoding AndroidManifest.xml with resources...
I: Loading resource table from file: /home/jbo/.local/share/apktool/framework/1.apk
I: Regular manifest package...
I: Decoding file-resources...
I: Decoding values */* XMLs...
I: Baksmaling classes.dex...
I: Baksmaling classes3.dex...
I: Baksmaling classes2.dex...
I: Copying assets and libs...
I: Copying unknown files...
I: Copying original files...
I: Copying META-INF/services directory
$ jbo@nix:~ cd toy/
$ jbo@nix:~/toy ll
total 48
drwxrwxr-x  10 jbo jbo 4096 Sep 21 07:59 ./
drwxrwxr-x   3 jbo jbo 4096 Sep 21 07:59 ../
-rw-rw-r--   1 jbo jbo 2618 Sep 21 07:59 AndroidManifest.xml
-rw-rw-r--   1 jbo jbo 2765 Sep 21 07:59 apktool.yml
drwxrwxr-x   8 jbo jbo 4096 Sep 21 07:59 kotlin/
drwxrwxr-x   3 jbo jbo 4096 Sep 21 07:59 META-INF/
drwxrwxr-x   3 jbo jbo 4096 Sep 21 07:59 original/
drwxrwxr-x 144 jbo jbo 4096 Sep 21 07:59 res/
drwxrwxr-x   9 jbo jbo 4096 Sep 21 07:59 smali/
drwxrwxr-x   4 jbo jbo 4096 Sep 21 07:59 smali_classes2/
drwxrwxr-x   3 jbo jbo 4096 Sep 21 07:59 smali_classes3/
drwxrwxr-x   2 jbo jbo 4096 Sep 21 07:59 unknown/
```

You see here `AndroidManifest.xml`, and if you examine it you'll see there is one activity - `com.jbo.toy.MainActivity`.  
Now we need to find that activity as a `*.smali` file - `apktool` might generate several SMALI class directories, so we need to find it!  
```
$ jbo@nix:~/toy find . -name MainActivity.smali
./smali_classes3/com/jbo/toy/MainActivity.smali
```

At this point, we need to examine that smali file, I will present it here in a raw form but skip a few parts to keep things neat:

```
.class public Lcom/jbo/toy/MainActivity;
.super Landroidx/appcompat/app/AppCompatActivity;
.source "MainActivity.java"


# direct methods
.method public constructor <init>()V
    .locals 0

    .line 14
    invoke-direct {p0}, Landroidx/appcompat/app/AppCompatActivity;-><init>()V

    return-void
.end method

.method private static getTitle(Z)Ljava/lang/String;
    .locals 1
    .param p0, "showName"    # Z

    .line 38
    if-eqz p0, :cond_0

    .line 39
    const-string v0, "JBO (@yo_yo_yo_jbo)"

    return-object v0

    .line 42
    :cond_0
    const-string v0, "https://jonathanbaror.com"

    return-object v0
.end method

.method private showToast()V
    .locals 5

    .line 30
    new-instance v0, Ljava/util/Random;

    invoke-direct {v0}, Ljava/util/Random;-><init>()V

    .line 31
    .local v0, "random":Ljava/util/Random;
    invoke-virtual {v0}, Ljava/util/Random;->nextBoolean()Z

    move-result v1

    .line 32
    .local v1, "randomBoolean":Z
    invoke-static {v1}, Lcom/jbo/toy/MainActivity;->getTitle(Z)Ljava/lang/String;

    move-result-object v2

    .line 33
    .local v2, "title":Ljava/lang/String;
    invoke-virtual {p0}, Lcom/jbo/toy/MainActivity;->getApplicationContext()Landroid/content/Context;

    move-result-object v3

    const/4 v4, 0x0

    invoke-static {v3, v2, v4}, Landroid/widget/Toast;->makeText(Landroid/content/Context;Ljava/lang/CharSequence;I)Landroid/widget/Toast;

    move-result-object v3

    .line 34
    .local v3, "toast":Landroid/widget/Toast;
    invoke-virtual {v3}, Landroid/widget/Toast;->show()V

    .line 35
    return-void
.end method
```
