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

The Chicory compiler can run at build-time or at run-time. The build-time flavor translates Wasm into Java bytecode and dumps the result into class files on disk. The run-time compiler loads this same bytecode dynamically, allowing you to load and run Wasm modules on-the-fly.

The interpreter is our reference implementation; as runtime maintainers, our key priority is keeping it easy to understand, debug and test. Development tends to move faster than the compiler: features are developed and tested first in the interpreter, then ported to the compiler as they stabilize. Therefore, the interpreter is meant to follow the spec rather strictly, not to be efficient. The compiler, on the other hand, is useful when you want to run Wasm modules with better performance.

The **build-time compiler** is the best pick when the Wasm modules you want to run are **known at build-time**. For instance, if a specific library has been ported to Wasm, you can convert it to Java bytecode and use it as plain Java code: this is the case for [sqlite4j][sqlite4j], a library that allows you to run SQLite databases in pure-Java.

On the other hand, you will reach for the **run-time compiler** when you want to run Wasm modules that are **_not_ known at build-time**. This is where Wasm's sandboxing capabilities shine: your users can extend the software you write, while you can rest easy knowing that these untrusted Wasm binaries run in a secure environment. This is the primary use case when your Wasm binaries are plugins.

A lot of Android software is written in Java (and, of course, Kotlin!), but you may already know that the Android platform does not run **Java bytecode**. Instead, it uses the [Android Runtime (ART)](https://developer.android.com/guide/app-basics/runtime) to run **Dalvik** bytecode ("DEX" files or "Dalvik EXecutables").

The good news is that Java bytecode can be translated into Dalvik bytecode—in fact, that's how modern Android software is built. Normally, Java bytecode is translated into Dalvik bytecode at build-time by your build tool, which makes Chicory's build-time compiler a great fit for Android software.

However, this approach falls short if you want to **load Wasm modules dynamically**—which, as we explained earlier, is actually a very common use case for us and the Extism community! The problem is that while most Java APIs for classloading and reflection work on Android, you cannot simply generate Java bytecode and stuff it into an Android class-loader, because that's not the bytecode format Android expects.

Well, that's alright—you might say—we can just translate the Java bytecode into Dalvik bytecode at run-time. If only! At the time of writing, there is no convenient tooling to perform that translation on-device.

There is, however, a bytecode engineering library called [DexMaker](https://github.com/linkedin/dexmaker/). DexMaker's main purpose is to allow you to generate Dalvik bytecode at run-time. The library is better known as a tool for generating dynamic proxies and as a dependency of [Mockito](https://site.mockito.org/) for dynamically generating mocks.

So we thought: why not use DexMaker to generate Dalvik bytecode from Wasm at run-time?

## DEX bytecode vs Java bytecode

The Dalvik bytecode format is really a distant relative of the Java bytecode format. The most glaring difference is that the Dalvik bytecode format is not stack-based, but register-based. This means that the Dalvik bytecode instructions operate on a certain number of registers, rather than a stack of values. This is a fundamental difference, and it means that there mapping between Java bytecode and Dalvik bytecode is not trivial.

In a normal, modern Android toolchain, your build tool initially produces Java bytecode; then this bytecode is fed into additional tools that translate the Java bytecode into Dalvik bytecode. This is usually done by the [d8 tool](https://developer.android.com/studio/command-line/d8), which is part of the Android SDK. The d8 tool is a command-line tool that can be used to convert Java bytecode into Dalvik bytecode, and it is also used by the Android Gradle plugin to perform this translation at build-time.

The architecture of the [d8 bytecode translator](https://developer.android.com/tools/d8) is that of a traditional multi-stage compiler: it first translates Java bytecode into an intermediate representation (IR), it does instruction selection, optimization, register allocation, and then it translates the IR into Dalvik bytecode. DexMaker is a layer on top of the d8 library that abstracts away some details (such as the way registers are allocated) and allows you to generate Dalvik bytecode in a more streamlined way.

Unfortunately, that still means that the Android backend of the Chicory compiler has to be different from the Java backend, at the very least because it has to be designed around register-based instructions. This does not mean we cannot share and reuse a lot of code of the JVM compiler, but it does mean that we have to be careful about the differences between the two backends; this is especially true for control-flow instructions, such as branches and loops, where using register-based instructions is quite different from using stack-based instructions.

<!-- FIME maybe let's give an example -->

## Runtime Limitations


Even though nowadays Android devices can be pretty beefy, the Dalvik runtime is still designed to run on devices with limited resources. In particular, to ensure a smooth user experience, the Dalvik runtime has some limitations on the amount of memory and stack space that can be used by an application.

### Limitations on the Stack Size

The Dalvik runtime has a strict limit of about **1MB** on the size of the stack frame **for the main thread**. This means that if you try to run a Wasm module that has a lot of nested function calls there, you may easily hit a stack overflow error. 

Add to that, that **dynamically-loaded** code is usually not pre-compiled to native code, and in fact, in most cases it will initially run in the Dalvik interpreter! This is not only a performance issue (that would be fine when we are only running tests): it also means that the stack usage is higher than it would be if the code was compiled to native code. For instance:

```
example stack trace here
```

As you might notice from the stack trace above, for each guest function call, the Dalvik interpreter introduces 3 stack frames! That makes it quite easy to hit the stack overflow limit, especially if you are running a Wasm module that has a lot of nested function calls.

Now, 1 MB may not sound like a lot, but it is actually quite reasonable for a typical Android application; the main thread usually just runs the UI and some background tasks, and it is not expected to run heavy computations. You would normally run heavy computations in a separate thread, and leave the main thread alone. The default stack size is still 1MB, but you can [configure the stack size of a thread when you create it.](https://developer.android.com/reference/java/lang/Thread#Thread(java.lang.ThreadGroup,%20java.lang.Runnable,%20java.lang.String,%20long)).

    var stackSize = 8 * 1024 * 1024;
    var t = new Thread(
            new ThreadGroup("chicory-thread-group"), 
            myRunnable, 
            "chicory-thread",
            stackSize);

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

In our experience, starting at around 150MB of heap, the Garbage collector starts to kick in more frequently: this is especially a problem if the memory belongs to the compiler! 

It is widely known that the JVM has a few limitations when it comes to class file size and number of methods. Indeed, even the Dalvik runtime has a limit of **65535 methods** per class file. In fact, on the JVM compiler, we specifically set up some features to allow the compiler to generate multiple class files, each with a reduced number of methods, and to fall-back to the interpreter if the size of a method is too large.

In the case of the Dalvik runtime, however, we have an additional limit on the amount of memory that we can use for the compiler: in the case of reasonably-sized Wasm modules, we have seen that the Dalvik runtime starts to struggle with memory usage when the compiler uses more than **150MB** of memory; it will throw a `java.lang.OutOfMemoryError` if the memory usage exceeds about **200MB**. 

This would not a huge problem if DexMaker allowed us to emit bytes for each method we are compiling and flush the compiled methods, but unfortunately, `DexMaker#generate()` produces an entire DEX file; this means that we have to keep the entire intermediate representation of the file in memory until the compilation is complete. Luckily, the solution is relatively simple: we can just split the compilation into multiple chunks, each producing a separate DEX file at an arbitrary, low boundary (we are currently using 200 methods per chunk). 

This way we were finally able to run a relatively complex Wasm module on Android (the mcp.run `fetch` servlet) without further issues.

## Testing the Android backend

The Android tooling for application development is not designed for this kind of lower-level development. For instance, testing Android applications is usually done using the [Android Instrumentation framework](https://developer.android.com/training/testing/). The instrumentation framework is a powerful tool that allows you to run tests on an Android device or emulator, but we figured it comes with a lot of overhead, because internally it transfers a lot of data between the host and the device. This is ok for end-to-end, integration testing, where normally flows are long and complex, but not so much for unit testing, where you want to run a lot of tests quickly.

With a normal flow, the entire Wasm spec test suite takes about 8 minutes to run in Android Studio, and about 4 minutes using Gradle on the command line. This is not terrible, but it is still a lot of time to wait for a test suite to run, especially when you are iterating on the code. 

In our case, we wanted to run Wasm spec test suite against our compiler. Some work [has been done to run the suite against the interpreter](https://github.com/dylibso/chicory/tree/main/android-tests). The overhead is there, but it is manageable, because the interpreter can be tested and debugged on a traditional JVM, and then only testing that we do on Android is to ensure that there aren't any platform-specific issues.

In the case of the compiler, however, we **need** to run and debug the compiler on Android, because there is no other way to test the generated Dalvik bytecode!

During the initial phases of the development we noticed that instrumentation tests added quite a bit of overhead. So, after some initial work, we decided to investigate the possibility of running unit tests directly on the Android device, without the instrumentation framework.

### Running Unit Tests on Android

Running unit tests on Android is not a common practice, but it is possible. Unfortunately documentation is scattered and a bit outdated, so we had to figure out a lot of things on our own. It doesn't help that while some people still rever to the Android Java platform as the "Dalvik runtime", Android has been using the Android Runtime (ART) since Android 5.0 (Lollipop), and that "ART" is a pretty unsearchable term. Moreover, even internally, the ART runtime is sometimes referred to as "Dalvik", which adds to the confusion. 

First of all, is it possible to run Java console applications on Android? It is; but first you have to figure out where the Java launcher is, and how to package your application.


In fact thanks to [this lone Github repo](https://github.com/WanghongLin/StandaloneDVM) we learn that the Java (ART) launche is located at `/apex/com.android.art/bin/dalvikvm64`. This is the launcher that you can use to run Java applications on Android.

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