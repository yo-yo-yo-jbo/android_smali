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

## SMALI in a nutshell
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

At this point, we need to examine that smali file, I will be focusing on small parts of it to keep things neat.  
However, this is how the file starts:

```smali
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
```

Let's analyze line by line:
- `.class` describes the class - its name is `com.jbo.toy.MainActivity` (note `.` is replaced by `/`). As I mentioned, class types start with an `L` and end with a semicolon.
- `.super` declares the superclass - our class derives from a base class `androidx.appcompat.app.AppCompatActivity`.
- `.source` is helpful for deubgging - the source file is called `MainActivity.java`. Note this will be omitted in release builds.
- We then start with our first method - the class's constructor, which is called `<init>`, gets no arguments and returns void - thus declared `.method public constructor <init>()V`.
- Our method does not use any local registers, so it says `.locals 0`. As I mentioned previously, `p0` in non-static methods is `this`, so it's well-defined.
- The `.line` directive is useful for debugging, but will be omitted in release builds.
- Our first instruction is `invoke-direct`, which is a method call. In this case it calls another method - `AppCompatActivity`'s constructor! Note it pushes `p0` as an argument, since the constructor must get a `this` instance.
- Lastly, `return-void` instruction basically means we return from the method without specifying a return value, as expected.

Note the JVM will check types of everything - so, if we try returning an integer in this function the JVM will crash the App.

Let us continue and examine the `getTitle` method:

```smali
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
```

Notes:
- This method is `private` and `static`. It gets a boolean (`Z`) and returns a String, thus declared `.method private static getTitle(Z)Ljava/lang/String;`.
- This method declares it requires one local register (`.locals 1`). It will use the register `v0` for that. Note JVM will crash if you try to refer to higher numbers (e.g. `v1`).
- The `.param` directive is useful for debugging, as it shows you parameter `p0` (the first argument, since this is a static method) is called `showName`. You won't find it in release builds though.
- Logic starts later at a condition: `if-eqz` should be read as "if equal to zero" - in this case, `p0` is our boolean, so it'd be equal to 0 if it's `false` (in JVM, zero is false and non-zero is true). If `p0` is zero we jump to a *label* named `:cond_0`. Labels start with a colon.
- Assuming we did not jump, we assign register `v0` a constant string (via `const-string` instruction). If this looks odd (how can a 32-bit register contain an entire long string?) keep in mind this is implemented by having v0 refer to a data table in the class in which that string resides. We then call `return-object v0` to exit the function and return that string.
- If we do jump, we end up in `:cond_0`, which assigns a different string and returns, similarly.

To get a complete understanding, let us analyze the `showToast` method:

```smali
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

Let us quickly analyze it:
- This method uses 5 local variables - registers `v0`, `v1`, `v2`, `v3` and `v4` - that's significantly higher than before!
- We start by creating a new instance of the Java `Random` class and assigning it to `v0`. Note `new-instance` simply allocates bytes for the instance - we need to initialize it by calling its constructor with `invoke-direct` and calling `Random`'s `<init>` method as we saw earlier!
- We then use `invoke-virtual` to call a virtual function - `Random`'s [nextBoolean](https://docs.oracle.com/javase/8/docs/api/java/util/Random.html#nextBoolean--) method.
- We save its result (a boolean, as indicated by its signature `nextBoolean()Z`) into `v1`, using `move-result`.
- We then use `invoke-static` to call a static method `getTitle`, which we have seen. We push `v1` as its argument and expecting get a resulting `String` instance.
- We use `move-result-object` to save that `String` instance into `v2`.
- Use call another virtual function - this time `getApplicationContext` function from our `MainActivity` - we push `this` (i.e. `p0`) and expect to get a `Context` back.
- We save the `Context` into `v3`.
- We assign `v4` the value of 0. Note `const/4` specifies the number of bytes - 4 corresponds to a 32-bit variable.
- We invoke a static method - `Toast.makeText` and push `v3`, `v2` and `v4`. Note `makeText` gets 3 arguments: the `Context`, the `CharSequence` (which `String` is) and an integer (0 in our case).
- We save the resulting `Toast` into `v3`. Yes - we can override `v3` with a completely different type and the JVM will keep track of that change.
- We call the virtual method `show` on the resulting `Toast`.
- We finish it off by invoking `return-void`.

### Our first patch
Let's do something easy - let's make the functionality not so random and always show the website! We have several options:
1. Patching the `showToast` method and passing `false` always to `getTitle`.
2. Patching `getTitle` and making it return what we want, always.
3. Ignoring `getTitle` completely - patch `showToast` and assign the string we want to the right register before creating the `Toast` object.

I chose option #2 just for the sake of demonstration, but you are free to try with anything you like.  
For that, we'd want to change `getTitle`'s `if-eqz p0, :cond_0` to `goto :cond_0`. How did I know `goto` is a valid instruction and its meaning?  
Well, I look it up - a great reference exists [here](http://pallergabor.uw.hu/androidblog/dalvik_opcodes.html) and shows all instructions (and their opcodes too!).  
Sometimes I'd also like to add some sort of debug strings - for that, using the [Log](https://developer.android.com/reference/android/util/Log) class really helps out!  
I'll call `Log.d` and add a message, implementing that only in Smali!  
So, I modified the `if` condition and added an invocation to `Log.d` - this is the "final" result:


```smali
.method private static getTitle(Z)Ljava/lang/String;
    .locals 1
    .param p0, "showName"    # Z

    .line 38
    const-string v0, "Patched by JBO"
    invoke-static {v0, v0}, Landroid/util/Log;->d(Ljava/lang/String;Ljava/lang/String;)I
    goto :cond_0

    .line 39
    const-string v0, "JBO (@yo_yo_yo_jbo)"

    return-object v0

    .line 42
    :cond_0
    const-string v0, "https://jonathanbaror.com"

    return-object v0
.end method
```

Note how I simply ignore the resulting `int` (`I`) from `Log.d` and how I used `v0` for the log message. I could do other things (like increasing `.locals` by one and using `v1`) but for the sake of simplicity I chose not to.
