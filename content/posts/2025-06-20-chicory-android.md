---
title: 'Porting Chicory to Android: A Hitchhiker’s Guide to Lower-Level Android Development'
author: 'Edoardo Vacchi'
date: 2025-06-20
tags: [Android, Chicory, WebAssembly]
---

If you are interested in Wasm and Java, you might have heard about [Chicory](https://chicory.dev), a pure-Java Wasm runtime. Ever since I started working at [Dylibso](https://dylibso.com), I have been contributing to the project. 

[Extism](https://extism.org) is Dylibso's family of open-source projects to develop and host WebAssembly (Wasm) plugins. Going forward, we believe the [Chicory SDK][chicory-sdk] is a solid foundation for running Wasm modules in Java applications, making it our default Wasm runtime for Extism on the Java platform. We're putting our money where our mouth is: the [mcpx4j][mcpx4j] library for running [mcp.run servlets][mcp-run] is built on top of the Chicory SDK.

At Dylibso, we have also been working on [mcp.run](https://www.mcp.run), a platform for writing, sharing and evaluating [secure, sandboxed MCP "servlets"](https://docs.mcp.run/blog/2025/04/07/mcp-run-security). The secret sauce of course remains Wasm, as the foundations are still laid on [Extism](https://extism.org) and our [XTP](https://getxtp.com) plugin registry.

Because Chicory is written in Java, it can run on any platform that supports the Java language. In fact, it is [pretty easy to run the Chicory interpreter on Android][chicory-android]. But the Chicory runtime is not just an interpreter: it also includes [a compiler][chicory-compiler] that translates Wasm into Java bytecode!

Unfortunately, the Android platform does not run Java bytecode, but Dalvik bytecode (DEX files). The Android toolchain translates Java bytecode into Dalvik bytecode at build-time. Does that mean you can use the Chicory compiler to translate Wasm into Dalvik bytecode? Well, yes—but it heavily depends on your use case.

This blog post documents why and how we ported the Chicory compiler to Android, the challenges we faced, and how we solved them. We also document how we are now testing the Android backend of the Chicory compiler, which deviates from how Android apps are usually developed. Ever wondered how to run and debug code on Android without the overhead of the Android Instrumentation framework? Read on!

## Porting Chicory to Android

As mentioned, the Chicory compiler can run at build-time or at run-time. The build-time flavor translates Wasm into Java bytecode and dumps the result into class files on disk. The run-time compiler loads this same bytecode dynamically, allowing you to load and run Wasm modules on-the-fly.

The interpreter is our reference implementation; as runtime maintainers, our key priority is keeping it easy to understand, debug and test. Development tends to move faster than the compiler: features are developed and tested first in the interpreter, then ported to the compiler as they stabilize. Therefore, the interpreter is meant to follow the spec rather strictly, not to be efficient. The compiler, on the other hand, is useful when you want to run Wasm modules with better performance.

The **build-time compiler** is the best pick when the Wasm modules you want to run are **known at build-time**. For instance, if a specific library has been ported to Wasm, you can convert it to Java bytecode and use it as plain Java code: this is the case for [sqlite4j][sqlite4j], a library that allows you to run SQLite databases in pure-Java.

On the other hand, you will reach for the **run-time compiler** when you want to run Wasm modules that are **_not_ known at build-time**. This is where Wasm's sandboxing capabilities shine: your users can extend the software you write, while you can rest easy knowing that these untrusted Wasm binaries run in a secure environment. This is the primary use case when your Wasm binaries are Extism plugins!

Now, while Java bytecode can normally be translated into Dalvik bytecode at build-time, this approach falls short if you want to **load Wasm modules dynamically**. The problem is that while most Java APIs for classloading and reflection work on Android, you cannot simply generate Java bytecode and load it into an Android class-loader, because that's not the bytecode format Android expects.

Well, that's alright—you might say—we can just translate the Java bytecode into Dalvik bytecode at run-time. If only! At the time of writing, there is no convenient tooling to perform that translation on-device.

There is, however, a bytecode engineering library called [DexMaker](https://github.com/linkedin/dexmaker/). DexMaker's main purpose is to allow you to generate Dalvik bytecode at run-time. The library is better known as a tool for generating dynamic proxies and as a dependency of [Mockito](https://site.mockito.org/) for dynamically generating mocks.

So we thought: why not use DexMaker to generate Dalvik bytecode from Wasm at run-time?

## DEX bytecode vs Java bytecode

The Dalvik bytecode format is a distant relative of Java bytecode, with one fundamental difference: Dalvik is register-based rather than stack-based. This means Dalvik instructions operate on registers instead of a stack of values, making the mapping between the two formats non-trivial.

In a modern Android toolchain, your build tool produces Java bytecode, which is then translated into Dalvik bytecode by the [d8 tool](https://developer.android.com/studio/command-line/d8) from the Android SDK. The [d8 bytecode translator](https://developer.android.com/tools/d8) follows a traditional multi-stage compiler architecture: it translates Java bytecode into an intermediate representation (IR), performs instruction selection and optimization, handles register allocation, and finally outputs Dalvik bytecode. [DexMaker](https://github.com/linkedin/dexmaker/) layers on top of d8, abstracting away these low-level details to provide a more streamlined API for generating Dalvik bytecode.

This means we need a separate Android backend for the Chicory compiler that works with register-based instructions. We can still reuse a lot of the JVM compiler code, but we have to watch out for the differences between the two approaches—especially with control-flow structures like branches and loops, where registers and stacks work pretty differently.

_TODO: add an example_

## Runtime Limitations

Even though nowadays Android devices can be pretty beefy, the Dalvik runtime is still designed to run on devices with limited resources. In particular, to ensure a smooth user experience, the Dalvik runtime has some limitations on the amount of memory and stack space that can be used by an application.

### Limitations on the Stack Size

Threads on the Dalvik runtime default to a stack size of **1 MB**. This is a strict limit on the `main` thread, which is usually the one running the UIlogic. If you try to run a Wasm module that uses more than 1 MB of stack space, you will get a `java.lang.StackOverflowError`.

Add to that, that **dynamically-loaded** code is usually not pre-compiled to native code, and in fact, in most cases it will initially run in the Dalvik interpreter! This is not only a performance issue (that would be fine when we are only running tests): it also means that the stack usage is higher than it would be if the code was compiled to native code. For instance:

```
example stack trace here
```

As you might notice from the stack trace above, for each guest function call, the Dalvik interpreter introduces 3 stack frames! That makes it quite easy to hit the stack overflow limit, especially if you are running a Wasm module that has a lot of nested function calls.

Now, even though this limit cannot be overcome on the main thread, you can still create a new thread with a custom stack size. This is generally the recommended way to run long computations on Android, as it allows you to avoid blocking the main thread and keep the UI responsive. The default stack size is still 1MB, but you can [configure the stack size of a thread when you create it.](https://developer.android.com/reference/java/lang/Thread#Thread(java.lang.ThreadGroup,%20java.lang.Runnable,%20java.lang.String,%20long)).

```java
    var stackSize = 8 * 1024 * 1024;
    var t = new Thread(
            new ThreadGroup("chicory-thread-group"), 
            myRunnable, 
            "chicory-thread",
            stackSize);
```

There is still one big caveat; when you spawn a thread with a custom stack size, apparently you will get less guarantees from the Dalvik runtime. In particular, in our experiments, sometimes you might get a hard crash, instead of a managed StackOverflowError!

```
Fatal signal 11 (SIGSEGV), code 2 (SEGV_ACCERR), fault addr 0x7284415700 in tid 4791 (thread), pid 4754 (ndroid.aot.test)
Cmdline: com.dylibso.chicory.android.aot.test
pid: 4754, tid: 4791, name: thread  >>> com.dylibso.chicory.android.aot.test <
      #04 pc 00000000000004a8  /data/data/com.dylibso.chicory.android.aot.test/cache/chicory196337394512609375/com/dylibso/chicory/_gen/CompiledMachine_Chunk0/classes.dex (com.dylibso.chicory._gen.
      #09 pc 00000000000004a8  /data/data/com.dylibso.chicory.android.aot.test/cache/chicory196337394512609375/com/dylibso/chicory/_gen/CompiledMachine_Chunk0/classes.dex (com.dylibso.chicory._gen.
      #14 pc 00000000000004a8  /data/data/com.dylibso.chicory.android.aot.test/cache/chicory196337394512609375/com/dylibso/chicory/_gen/CompiledMachine_Chunk0/classes.dex (com.dylibso.chicory._gen.
      #19 pc 00000000000004a8  /data/data/com.dylibso.chicory.android.aot.test/cache/chicory196337394512609375/com/dylibso/chicory/_gen/CompiledMachine_Chunk0/classes.dex (com.dylibso.chicory._gen.
      #24 pc 00000000000004a8  /data/data/com.dylibso.chicory.android.aot.test/cache/chicory196337394512609375/com/dylibso/chicory/_gen/CompiledMachine_Chunk0/classes.dex (com.dylibso.chicory._gen.
      #29 pc 00000000000004a8  /data/data/com.dylibso.chicory.android.aot.test/cache/chicory196337394512609375/com/dylibso/chicory/_gen/CompiledMachine_Chunk0/classes.dex (com.dylibso.chicory._gen.
      #34 pc 00000000000004a8  /data/data/com.dylibso.chicory.android.aot.test/cache/chicory196337394512609375/com/dylibso/chicory/_gen/CompiledMachine_Chunk0/classes.dex (com.dylibso.chicory._gen.
// ... rest omitted ...
```

At this time we are not aware of a simple way to avoid this issue, but there is still a lot of work to do!


## Memory Management

In our experience, starting around 150MB of heap usage, the Garbage Collector kicks in more frequently: this is especially problematic when that memory belongs to the compiler!

It's widely known that the JVM has limitations on class file size and method count. The Dalvik runtime has similar constraints, with a limit of **65,535 methods** per class file. On the JVM compiler, we handle this by generating multiple class files with reduced method counts and falling back to the interpreter when individual methods become too large.

The Dalvik runtime introduces an additional constraint: memory usage. With reasonably-sized Wasm modules, we've observed that the Dalvik runtime struggles when compiler memory usage exceeds **150MB** and throws `java.lang.OutOfMemoryError` around **200MB**.

This wouldn't be a major issue if DexMaker allowed us to emit bytecode incrementally, but `DexMaker#generate()` produces an entire DEX file at once. This forces us to keep the complete intermediate representation in memory until compilation finishes. 

Luckily, the solution is relatively straightforward: split compilation into chunks, and produce a separate class at a low boundary (currently 200 methods per chunk). This approach finally allowed us to run complex Wasm modules on Android, including, notably, the [mcp.run `fetch` servlet](https://www.mcp.run/dylibso/fetch), without memory issues.

## Testing the Android backend

Android's development tooling is not designed for this kind of lower-level work. Testing Android applications typically uses the [Android Instrumentation framework](https://developer.android.com/training/testing/). The instrumentation framework is a powerful tool but comes with significant overhead, because internally it transfers a lot of data between the host and the device. This is ok for end-to-end, integration testing, where flows are usually long and complex, but not so much for unit testing, where you want to run a lot of tests quickly.

The instrumentation framework is the default way to run tests on the device, but normally this is not necessary, you can run tests on the host machine on a plain JVM; even when Android-specific APIs are involved, [Roboelectric](https://robolectric.org/) simulates the Android environment, allowing you to run tests without the overhead of the instrumentation framework.

But with this approach we can't test the Chicory compiler for Android, because the JVM does not support DEX bytecode! The Chicory compiler _must_ run on Android, and we _need_ to test it on Android!

Some work [has been done to run the suite against the interpreter](https://github.com/dylibso/chicory/tree/main/android-tests). Even though there is some overhead, it is manageable, because the interpreter can be tested and debugged on a traditional JVM, and then only testing that we do on Android is to ensure that there aren't any platform-specific issues.

However, with the instrumentation framework, running the entire Wasm spec test suite against the Android compiler takes about **8 minutes** in Android Studio, and about **4 minutes** using Gradle on the command line. This is not terrible, but it is still a lot of time to wait for a test suite to run, especially when you are iterating on the code. Compare this to the plain JVM, where the same test suite takes about **30 seconds** to run! (_FIXME: the JVM number is made up, need to measure again_)

So, I decided to investigate how to run unit tests on Android _without_ the overhead of the instrumentation framework. 

### Running Plain Java on ART

Running unit tests on Android is not a common practice, but it is possible. Unfortunately documentation is scattered and a bit outdated, so we had to figure out a lot of things on our own. It doesn't help that while some people still rever to the Android Java platform as the "Dalvik runtime", Android has been using the Android Runtime (ART) since Android 5.0 (Lollipop), and that "ART" is a pretty unsearchable term. Moreover, even internally, the ART runtime is sometimes referred to as "Dalvik", which adds to the confusion. 

First of all, is it possible to run Java console applications on Android? It is; but first you have to figure out where the Java launcher is, and how to package your application.

In fact thanks to [this lone Github repo](https://github.com/WanghongLin/StandaloneDVM) we learn that the Java (ART) launcher is located at `/apex/com.android.art/bin/dalvikvm64` (`/apex/com.android.art/bin/dalvikvm` is the 32-bit version). This is the launcher that you can use to run Java applications on Android.

Let's write a simple Java application that prints the name of the host operating system to the console. 

```java
class Main {
    public static void main(String... args) { 
        System.out.printf("Hello, %s %s\n", 
            System.getProperty("os.name"),
            System.getProperty("os.version")); 
    }
}
```

On my machine, I can compile and run this code with the following commands:

```sh
❯ javac Main.java
❯ java Main
Hello, Mac OS X 15.5
```

Now, assuming your Android tools are on the PATH (on my machine they are in `$HOME/Library/Android/sdk/build-tools/35.0.0`), you can convert the class into a DEX file with the following command:

```sh
d8 Main.class
```

This will create a file called `classes.dex` in the current directory. This is the Dalvik bytecode that you can run on Android. Then, assuming you have a running Android emulator (or a connected device), you can upload the file to `/tmp/main.dex` with the following command:

```sh
❯ adb push classes.dex /tmp/main.dex
classes.dex: 1 file pushed, 0 skipped. 2.3 MB/s (1076 bytes in 0.000s)
```

Now you can run it on Android:

```sh
❯ adb shell /apex/com.android.art/bin/dalvikvm64 -cp /tmp/main.dex Main
Hello, Linux 6.6.30-android15-8-gdd9c02ccfe27-ab11987101-4k
```

Congratulations! You have just run a plain Java application on Android!

### Running Unit Tests on Android

Putting it all together, assuming your android test cases are in `androidTest` you can run them with the following commands:

```sh
./gradlew assembleAndroidTest
adb push MY_PROJECT/build/outputs/apk/androidTest/debug/MY_PROJECT-debug-androidTest.apk /tmp/main.apk
adb shell /apex/com.android.art/bin/dalvikvm64 -cp /tmp/main.apk com.example.Main
```

for instance, if you are using junit4 you can start the launcher with the following command:

```sh
adb /apex/com.android.art/bin/dalvikvm64 -cp main.apk org.junit.runner.JUnitCore <test class>
```

This will run the JUnit test class on the Android device, and you will see the output in the console. Notice this is sidestepping the usual Android launcher. This means that you can run your tests without the overhead of the instrumentation framework, but also you will not be able to use any Android-specific features, such as the Android UI framework or other Android APIs. Just to give you an example, you will not be able to use the Android logging framework.

### Debugging the Android backend

It is also possible to attach a debugger to the Android process running the tests. This is useful to debug the Chicory compiler and the generated Dalvik bytecode. In this case, the instructions are even more hard to find, but with some trial and error, especially following the `logcat` output I was able to figure out that you can attach a debugger to the Android process running the tests by using the following command:

```sh
/apex/com.android.art/bin/dalvikvm64 -Xplugin:libopenjdkjvmti.so -agentpath:libjdwp.so=server=y,suspend=y,transport=dt_socket,address=0.0.0.0:8000 -cp /tmp/main.apk com.example.Main
```

Then you will be able to attach a remote debugging session using your favorite IDE (e.g. IntelliJ IDEA or Android Studio) to the port `8000` on the Android device.

The critical part is `-Xplugin:libopenjdkjvmti.so`, otherwise the `-agentpath` flag has no effect.

## Conclusion

The Android backend of the Chicory compiler is still a work in progress, but it is showing promising results. If you are interested in lower-level Android development, or if you want to run Wasm modules on Android, I hope this article has given you some insights into the challenges and solutions we have found so far. If you want to try it out, you can ...


[chicory-android]: https://docs.mcp.run/blog/2024/12/27/running-tools-on-android/
[chicory-compiler]: https://chicory.dev/blog/chicory-1.4.0
[sqlite4j]: https://github.com/roastedroot/sqlite4j
[chicory-sdk]: https://github.com/extism/chicory-sdk
[mcpx4j]: https://github.com/dylibso/mcpx4j
[mcp-run]: https://www.mcp.run