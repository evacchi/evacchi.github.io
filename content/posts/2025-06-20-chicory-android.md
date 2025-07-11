---
title: 'Wasm the Hard Way: Porting the Chicory Compiler to Android'
author: 'Edoardo Vacchi'
date: 2025-06-20
tags: [Android, Chicory, WebAssembly]
---

I am resurrecting my old ["Wasm the Hard Way"](/tags/wasm/) series with a some fresh new content about a recent project I have been working on: porting the [Chicory](https://chicory.dev) Wasm compiler to Android.

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

Consider this example:

```c
int example() {
    int x;
    int param1 = 1;
    int param2 = 2;
    
    while (1) {
        int sum = param1 + param2;
        x = sum;
        
        if (!(x < 10)) {
            break;
        }
        
        // Prepare for next iteration
        param1 = sum;
        param2 = 3;
    }
    
    return x;
}
```

The equivalent Wasm code would look like this:

```scheme
  (func (export "example") (result i32)
    (local $x i32)
    (i32.const 1)
    (i32.const 2)
    (loop (param i32 i32) (result i32)
      (i32.add)
      (local.tee $x)
      (i32.const 3)
      (local.get $x)
      (i32.const 10)
      (i32.lt_u)
      (br_if 0)
      (drop)
    )
  )
```

The equivalent Java bytecode is relatively straightforward, because the JVM is also stack-based. The loop condition is checked at the end of the loop, and the parameters are updated at the end of each iteration.

```
// Method: func_0(Lcom/dylibso/chicory/runtime/Memory;Lcom/dylibso/chicory/runtime/Instance;)I
// Stack: 3, Locals: 3

 0: iconst_0          // push 0 (initialize local variable)
 1: istore 2          // x = 0 (local variable initialization)
 2: iconst_1          // push 1 (loop param 1)
 3: iconst_2          // push 2 (loop param 2)

 // Loop start L0 - stack: [param1, param2]
 4: iadd              // add top two stack values -> stack: [sum]
 5: dup               // duplicate sum -> stack: [sum, sum]
 6: istore 2          // store sum in local x -> stack: [sum]
 7: iconst_3          // push 3 -> stack: [sum, 3]
 8: iload 2           // load x -> stack: [sum, 3, x]
 9: bipush 10         // push 10 -> stack: [sum, 3, x, 10]
10: invokestatic com/dylibso/chicory/runtime/OpcodeImpl.I32_LT_U (II)I  // compare x < 10 -> stack: [sum, 3, result]
11: ifeq L1           // if result == 0 (false), exit loop -> stack: [sum, 3]
12: invokestatic com/dylibso/chicory/runtime/Memory.checkInterruption ()V  // interruption check -> stack: [sum, 3]
13: goto L0           // loop back -> stack: [sum, 3] (these become new params)

// Loop exit L1 - stack: [sum, 3]
14: pop               // remove the 3 -> stack: [sum]
15: ireturn           // return sum

// Local variable table:
// 0: Memory parameter
// 1: Instance parameter  
// 2: x (local variable)
```
   
A relatively direct Dalvik translation could pre-allocate the registers assigning them specific roles, 
and ensure that they are preserved as invariants across iterations. In the following, the `v0` register 
is used to hold the loop result, `v1` and `v2` are the loop parameters, and `v3` is the sum. 
The loop condition checks if the sum is less than 10, and if so, it continues iterating.


```dalvik
;; ... function prologue (reads args etc) ...
0011: const/4             v1 #1 #1     ; param1 = 1
0012: const/4             v2 #2 #2     ; param2 = 2
0013: add-int             v3, v1, v2   ; sum = param1 + param2
0015: move/from16         v0, v3       ; loop result = sum (allocated at loop header)
0017: const/16            v4 #3 #3     ; constant 3 (pushed to stack)
0019: move/from16         v5, v0       ; setup comparison operand 1
001b: const/16            v6 #10 #10   ; setup comparison operand 2
001d: invoke-static/range Lcom/dylibso/chicory/runtime/OpcodeImpl;->I32_LT_U(II)I {v5, v6}  ; compare loop_result < 10
0020: move-result         v7           ; branch condition result
0021: move/from16         v1, v3       ; param1 = sum (loop param update)
0023: move/from16         v2, v4       ; param2 = 3 (loop param update)
0025: move/from16         v0, v4       ; loop result = current stack top (systematic update)
0027: if-eqz              v7 -> 002d   ; if condition false, exit loop
0029: invoke-static       Lcom/dylibso/chicory/experimental/android/aot/AotMethods;->checkInterruption()V {}  ; runtime interruption check
002c: goto                -> 0013      ; continue loop
002d: move/from16         v0, v3       ; loop result = final sum (systematic update)
;; ... function epilogue (return the values) ...
```

The loop result is consistently maintained in `v0`, which is allocated at the loop header (0011) and updated at each iteration (0025) and at the final exit (002d).
Notice how, at the end of the loop, we ensure that `v0`, `v1` and `v2` are updated with the latest values, so, as we re-enter the loop, the registers hold the correct values for the next iteration. 

The main limitation of this approach is that it will waste a few registers, and generate a few more moves than strictly necessary. However this makes the compiler simpler: while more sophisticated approaches could be used to optimize the register usage, they could add overhead to the compiler and slow it down. We will revisit this in the future, but for now, we are happy with this approach. 

## Runtime Limitations

Even though nowadays Android devices can be pretty beefy, you cannot give resources for granted. In particular, to ensure a smooth user experience, the ART runtime poses some limitations on the amount of memory and stack space that can be used by an application. Let's see what these limitations are, and how we can work around them.

### Limitations on the Stack Size

Threads on the Dalvik runtime default to a stack size of **1 MB**. This is a strict limit on the `main` thread, which is usually the one running the UIlogic. If you try to run a Wasm module that uses more than 1 MB of stack space, you will get a `java.lang.StackOverflowError`.

Add to that, that **dynamically-loaded** code is usually not pre-compiled to native code, and in fact, in most cases it will initially run in the Dalvik interpreter! This is not only a performance issue (that would be fine when we are only running tests): it also means that the stack usage is higher than it would be if the code was compiled to native code. For instance:

```
--------- beginning of crash
07-11 11:18:25.889 27566 27607 F libc    : Fatal signal 11 (SIGSEGV), code 2 (SEGV_ACCERR), fault addr 0x6ddb4fd000 in tid 27607 (roidJUnitRunner), pid 27566 (ndroid.aot.test)
07-11 11:18:25.898 27612 27612 E crash_dump64: failed to get the guest state header for thread 27566: Bad address
07-11 11:18:25.898 27612 27612 E crash_dump64: failed to get the guest state header for thread 27567: Bad address
07-11 11:18:25.898 27612 27612 E crash_dump64: failed to get the guest state header for thread 27568: Bad address
07-11 11:18:25.899 27612 27612 E crash_dump64: failed to get the guest state header for thread 27569: Bad address
07-11 11:18:25.899 27612 27612 E crash_dump64: failed to get the guest state header for thread 27570: Bad address
07-11 11:18:25.899 27612 27612 E crash_dump64: failed to get the guest state header for thread 27571: Bad address
07-11 11:18:25.899 27612 27612 E crash_dump64: failed to get the guest state header for thread 27572: Bad address
07-11 11:18:25.900 27612 27612 E crash_dump64: failed to get the guest state header for thread 27573: Bad address
07-11 11:18:25.900 27612 27612 E crash_dump64: failed to get the guest state header for thread 27574: Bad address
07-11 11:18:25.900 27612 27612 E crash_dump64: failed to get the guest state header for thread 27575: Bad address
07-11 11:18:25.901 27612 27612 E crash_dump64: failed to get the guest state header for thread 27576: Bad address
07-11 11:18:25.901 27612 27612 E crash_dump64: failed to get the guest state header for thread 27577: Bad address
07-11 11:18:25.901 27612 27612 E crash_dump64: failed to get the guest state header for thread 27606: Bad address
07-11 11:18:25.902 27612 27612 E crash_dump64: failed to get the guest state header for thread 27607: Bad address
07-11 11:18:25.902 27612 27612 E crash_dump64: failed to get the guest state header for thread 27608: Bad address
07-11 11:18:25.907 27612 27612 I crash_dump64: obtaining output fd from tombstoned, type: kDebuggerdTombstoneProto
07-11 11:18:25.908   216   216 I tombstoned: received crash request for pid 27607
07-11 11:18:25.908 27612 27612 I crash_dump64: performing dump of process 27566 (target tid = 27607)
07-11 11:18:25.975 27612 27612 F DEBUG   : *** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
07-11 11:18:25.975 27612 27612 F DEBUG   : Build fingerprint: 'google/sdk_gphone64_arm64/emu64a:15/AE3A.240806.043/12960925:userdebug/dev-keys'
07-11 11:18:25.975 27612 27612 F DEBUG   : Revision: '0'
07-11 11:18:25.975 27612 27612 F DEBUG   : ABI: 'arm64'
07-11 11:18:25.975 27612 27612 F DEBUG   : Timestamp: 2025-07-11 11:18:25.911063122+0200
07-11 11:18:25.975 27612 27612 F DEBUG   : Process uptime: 2s
07-11 11:18:25.975 27612 27612 F DEBUG   : Cmdline: com.dylibso.chicory.android.aot.test
07-11 11:18:25.975 27612 27612 F DEBUG   : pid: 27566, tid: 27607, name: roidJUnitRunner  >>> com.dylibso.chicory.android.aot.test <<<
07-11 11:18:25.975 27612 27612 F DEBUG   : uid: 10238
07-11 11:18:25.975 27612 27612 F DEBUG   : tagged_addr_ctrl: 0000000000000001 (PR_TAGGED_ADDR_ENABLE)
07-11 11:18:25.976 27612 27612 F DEBUG   : pac_enabled_keys: 000000000000000f (PR_PAC_APIAKEY, PR_PAC_APIBKEY, PR_PAC_APDAKEY, PR_PAC_APDBKEY)
07-11 11:18:25.976 27612 27612 F DEBUG   : signal 11 (SIGSEGV), code 2 (SEGV_ACCERR), fault addr 0x0000006ddb4fd000
07-11 11:18:25.976 27612 27612 F DEBUG   :     x0  0000006ddb4faa80  x1  0000000000000000  x2  00000000000107a8  x3  0000006ddb4fd000
07-11 11:18:25.976 27612 27612 F DEBUG   :     x4  0000006ddb50d7e8  x5  0000000000000004  x6  0000006ddb50daf8  x7  0000006d5565be76
07-11 11:18:25.976 27612 27612 F DEBUG   :     x8  0000000000012d70  x9  0000006ddb50d7f0  x10 00000000000025a5  x11 0000006ddfb79040
07-11 11:18:25.976 27612 27612 F DEBUG   :     x12 0000006ddfb79040  x13 00000000000025a5  x14 0000006ddb50d7c8  x15 b400006fa33821b0
07-11 11:18:25.976 27612 27612 F DEBUG   :     x16 0000006de0224c10  x17 0000007082a97bc0  x18 0000006d528f8000  x19 b400006eb33a56a0
07-11 11:18:25.976 27612 27612 F DEBUG   :     x20 0000006ddb50db40  x21 0000006de040d000  x22 b400006f933820d0  x23 0000006ddb4faa80
07-11 11:18:25.976 27612 27612 F DEBUG   :     x24 0000006d00213840  x25 00000000000025a5  x26 0000000000012d28  x27 00000000000025a5
07-11 11:18:25.976 27612 27612 F DEBUG   :     x28 0000000000000003  x29 0000006ddb50d8d0
07-11 11:18:25.976 27612 27612 F DEBUG   :     lr  0000006ddfb5b774  sp  0000006ddb4faa80  pc  0000007082a97c84  pst 0000000020001000
07-11 11:18:25.976 27612 27612 F DEBUG   : 512 total frames
07-11 11:18:25.976 27612 27612 F DEBUG   : backtrace:
07-11 11:18:25.976 27612 27612 F DEBUG   :       #00 pc 0000000000055c84  /apex/com.android.runtime/lib64/bionic/libc.so (__memset_aarch64+196) (BuildId: 1b9fecf834d610f77e641f026ca7269b)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #01 pc 000000000035b770  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+476) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #02 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #03 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #04 pc 00000000000004a8  [anon:dalvik-DEX data] (com.dylibso.chicory._gen.CompiledMachine_Chunk0.func_0+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #05 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #06 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #07 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #08 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #09 pc 00000000000004a8  [anon:dalvik-DEX data] (com.dylibso.chicory._gen.CompiledMachine_Chunk0.func_0+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #10 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #11 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #12 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #13 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #14 pc 00000000000004a8  [anon:dalvik-DEX data] (com.dylibso.chicory._gen.CompiledMachine_Chunk0.func_0+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #15 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #16 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #17 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #18 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #19 pc 00000000000004a8  [anon:dalvik-DEX data] (com.dylibso.chicory._gen.CompiledMachine_Chunk0.func_0+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #20 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #21 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #22 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #23 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #24 pc 00000000000004a8  [anon:dalvik-DEX data] (com.dylibso.chicory._gen.CompiledMachine_Chunk0.func_0+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #25 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #26 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #27 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #28 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #29 pc 00000000000004a8  [anon:dalvik-DEX data] (com.dylibso.chicory._gen.CompiledMachine_Chunk0.func_0+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #30 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #31 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #32 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #33 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #34 pc 00000000000004a8  [anon:dalvik-DEX data] (com.dylibso.chicory._gen.CompiledMachine_Chunk0.func_0+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #35 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #36 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #37 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #38 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #39 pc 00000000000004a8  [anon:dalvik-DEX data] (com.dylibso.chicory._gen.CompiledMachine_Chunk0.func_0+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #40 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #41 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #42 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #43 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #44 pc 00000000000004a8  [anon:dalvik-DEX data] (com.dylibso.chicory._gen.CompiledMachine_Chunk0.func_0+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #45 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #46 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #47 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #48 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #49 pc 00000000000004a8  [anon:dalvik-DEX data] (com.dylibso.chicory._gen.CompiledMachine_Chunk0.func_0+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #50 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #51 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #52 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #53 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #54 pc 00000000000004a8  [anon:dalvik-DEX data] (com.dylibso.chicory._gen.CompiledMachine_Chunk0.func_0+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #55 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #56 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #57 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #58 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #59 pc 00000000000004a8  [anon:dalvik-DEX data] (com.dylibso.chicory._gen.CompiledMachine_Chunk0.func_0+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #60 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #61 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #62 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #63 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #64 pc 0000000000000cb0  [anon:dalvik-DEX data] (com.dylibso.chicory._gen.CompiledMachine_Chunk0.call+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #65 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #66 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #67 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #68 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #69 pc 00000000000023b4  [anon:dalvik-DEX data] (com.dylibso.chicory._gen.CompiledMachine.call+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #70 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #71 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #72 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #73 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #74 pc 000000000000168c  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (com.dylibso.chicory.experimental.android.aot.TestAotAndroidMachine.call+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #75 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #76 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #77 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #78 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #79 pc 000000000036a664  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (com.dylibso.chicory.runtime.Instance$Exports.lambda$function$0$com-dylibso-chicory-runtime-Instance$Exports+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #80 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #81 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #82 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #83 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #84 pc 000000000036a464  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (com.dylibso.chicory.runtime.Instance$Exports$$ExternalSyntheticLambda0.apply+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #85 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #86 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #87 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #88 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #89 pc 00000000002aa140  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (com.dylibso.chicory.test.gen.SpecV1FacTest.test1+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #90 pc 000000000034d5a8  /apex/com.android.art/lib64/libart.so (artQuickToInterpreterBridge+1932) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #91 pc 0000000000379098  /apex/com.android.art/lib64/libart.so (art_quick_to_interpreter_bridge+88) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #92 pc 0000000000362774  /apex/com.android.art/lib64/libart.so (art_quick_invoke_stub+612) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #93 pc 000000000035e36c  /apex/com.android.art/lib64/libart.so (_jobject* art::InvokeMethod<(art::PointerSize)8>(art::ScopedObjectAccessAlreadyRunnable const&, _jobject*, _jobject*, _jobject*, unsigned long)+540) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #94 pc 00000000006c87f8  /apex/com.android.art/lib64/libart.so (art::Method_invoke(_JNIEnv*, _jobject*, _jobject*, _jobjectArray*) (.__uniq.165753521025965369065708152063621506277)+32) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #95 pc 00000000001c8670  [anon_shmem:dalvik-jit-code-cache] (offset 0x2000000) (art_jni_trampoline+144)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #96 pc 0000000000362774  /apex/com.android.art/lib64/libart.so (art_quick_invoke_stub+612) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #97 pc 000000000035bd1c  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+1928) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #98 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #99 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #100 pc 000000000048c248  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.commons.util.ReflectionUtils.invokeMethod+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #101 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #102 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #103 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #104 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #105 pc 000000000047defc  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.jupiter.engine.execution.MethodInvocation.proceed+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #106 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #107 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #108 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #109 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #110 pc 000000000047d814  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.jupiter.engine.execution.InvocationInterceptorChain$ValidatingInvocation.proceed+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #111 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #112 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #113 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #114 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #115 pc 0000000000483088  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.jupiter.engine.extension.TimeoutExtension.intercept+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #116 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #117 pc 000000000034f868  /apex/com.android.art/lib64/libart.so (art::interpreter::ArtInterpreterToInterpreterBridge(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame*, art::JValue*)+100) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #118 pc 00000000003489ac  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<true>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+792) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #119 pc 000000000076db3c  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12452) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #120 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #121 pc 00000000004830e0  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.jupiter.engine.extension.TimeoutExtension.interceptTestableMethod+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #122 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #123 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #124 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #125 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #126 pc 00000000004834e4  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.jupiter.engine.extension.TimeoutExtension.interceptTestMethod+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #127 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #128 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #129 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #130 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #131 pc 000000000047852c  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.jupiter.engine.descriptor.TestMethodTestDescriptor$$ExternalSyntheticLambda6.apply+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #132 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #133 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #134 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #135 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #136 pc 000000000047d5b0  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.jupiter.engine.execution.InterceptingExecutableInvoker$ReflectiveInterceptorCall.lambda$ofVoidMethod$0+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #137 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #138 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #139 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #140 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #141 pc 000000000047d574  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.jupiter.engine.execution.InterceptingExecutableInvoker$ReflectiveInterceptorCall$$ExternalSyntheticLambda0.apply+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #142 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #143 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #144 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #145 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #146 pc 000000000047d6a0  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.jupiter.engine.execution.InterceptingExecutableInvoker.lambda$invoke$0+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #147 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #148 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #149 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #150 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #151 pc 000000000047d528  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.jupiter.engine.execution.InterceptingExecutableInvoker$$ExternalSyntheticLambda0.apply+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #152 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #153 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #154 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #155 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #156 pc 000000000047d6dc  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.jupiter.engine.execution.InvocationInterceptorChain$InterceptedInvocation.proceed+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #157 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #158 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #159 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #160 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #161 pc 000000000047da3c  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.jupiter.engine.execution.InvocationInterceptorChain.proceed+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #162 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #163 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #164 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #165 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #166 pc 000000000047d9cc  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.jupiter.engine.execution.InvocationInterceptorChain.chainAndInvoke+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #167 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #168 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #169 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #170 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #171 pc 000000000047da00  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.jupiter.engine.execution.InvocationInterceptorChain.invoke+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #172 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #173 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #174 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #175 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #176 pc 000000000047d678  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.jupiter.engine.execution.InterceptingExecutableInvoker.invoke+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #177 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #178 pc 000000000034f868  /apex/com.android.art/lib64/libart.so (art::interpreter::ArtInterpreterToInterpreterBridge(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame*, art::JValue*)+100) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #179 pc 00000000003489ac  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<true>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+792) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #180 pc 000000000076db3c  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12452) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #181 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #182 pc 000000000047d628  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.jupiter.engine.execution.InterceptingExecutableInvoker.invoke+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #183 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #184 pc 000000000034f868  /apex/com.android.art/lib64/libart.so (art::interpreter::ArtInterpreterToInterpreterBridge(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame*, art::JValue*)+100) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #185 pc 00000000003489ac  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<true>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+792) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #186 pc 000000000076db3c  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12452) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #187 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #188 pc 0000000000478d80  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.jupiter.engine.descriptor.TestMethodTestDescriptor.lambda$invokeTestMethod$8$org-junit-jupiter-engine-descriptor-TestMethodTestDescriptor+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #189 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #190 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #191 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #192 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #193 pc 00000000004783dc  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.jupiter.engine.descriptor.TestMethodTestDescriptor$$ExternalSyntheticLambda1.execute+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #194 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #195 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #196 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #197 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #198 pc 000000000049caec  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #199 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #200 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #201 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #202 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #203 pc 0000000000478bd8  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.jupiter.engine.descriptor.TestMethodTestDescriptor.invokeTestMethod+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #204 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #205 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #206 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #207 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #208 pc 0000000000478700  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.jupiter.engine.descriptor.TestMethodTestDescriptor.execute+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #209 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #210 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #211 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #212 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #213 pc 0000000000478848  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.jupiter.engine.descriptor.TestMethodTestDescriptor.execute+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #214 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #215 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #216 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #217 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #218 pc 000000000049ba44  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$6$org-junit-platform-engine-support-hierarchical-NodeTestTask+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #219 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #220 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #221 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #222 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #223 pc 000000000049aec4  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.NodeTestTask$$ExternalSyntheticLambda0.execute+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #224 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #225 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #226 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #227 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #228 pc 000000000049caec  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #229 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #230 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #231 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #232 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #233 pc 000000000049bb38  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$8$org-junit-platform-engine-support-hierarchical-NodeTestTask+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #234 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #235 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #236 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #237 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #238 pc 000000000049aefc  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.NodeTestTask$$ExternalSyntheticLambda10.invoke+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #239 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #240 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #241 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #242 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #243 pc 000000000049c3c0  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.Node.around+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #244 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #245 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #246 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #247 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #248 pc 000000000049bb78  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$9$org-junit-platform-engine-support-hierarchical-NodeTestTask+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #249 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #250 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #251 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #252 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #253 pc 000000000049af34  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.NodeTestTask$$ExternalSyntheticLambda11.execute+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #254 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #255 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #256 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #257 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #258 pc 000000000049caec  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #259 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #260 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #261 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #262 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #263 pc 000000000049b99c  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.NodeTestTask.executeRecursively+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #264 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #265 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #266 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #267 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #268 pc 000000000049b8b0  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.NodeTestTask.execute+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #269 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #270 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #271 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #272 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #273 pc 000000000049c6e0  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.SameThreadHierarchicalTestExecutorService$$ExternalSyntheticLambda0.accept+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #274 pc 000000000034d5a8  /apex/com.android.art/lib64/libart.so (artQuickToInterpreterBridge+1932) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #275 pc 0000000000379098  /apex/com.android.art/lib64/libart.so (art_quick_to_interpreter_bridge+88) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #276 pc 0000000000164bbc  [anon_shmem:dalvik-jit-code-cache] (offset 0x2000000) (java.util.ArrayList.forEach+220)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #277 pc 0000000000362774  /apex/com.android.art/lib64/libart.so (art_quick_invoke_stub+612) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #278 pc 000000000035bd1c  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+1928) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #279 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #280 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #281 pc 000000000049c720  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.SameThreadHierarchicalTestExecutorService.invokeAll+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #282 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #283 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #284 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #285 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #286 pc 000000000049ba44  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$6$org-junit-platform-engine-support-hierarchical-NodeTestTask+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #287 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #288 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #289 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #290 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #291 pc 000000000049aec4  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.NodeTestTask$$ExternalSyntheticLambda0.execute+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #292 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #293 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #294 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #295 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #296 pc 000000000049caec  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #297 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #298 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #299 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #300 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #301 pc 000000000049bb38  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$8$org-junit-platform-engine-support-hierarchical-NodeTestTask+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #302 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #303 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #304 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #305 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #306 pc 000000000049aefc  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.NodeTestTask$$ExternalSyntheticLambda10.invoke+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #307 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #308 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #309 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #310 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #311 pc 000000000049c3c0  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.Node.around+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #312 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #313 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #314 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #315 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #316 pc 000000000049bb78  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$9$org-junit-platform-engine-support-hierarchical-NodeTestTask+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #317 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #318 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #319 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #320 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #321 pc 000000000049af34  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.NodeTestTask$$ExternalSyntheticLambda11.execute+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #322 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #323 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #324 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #325 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #326 pc 000000000049caec  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #327 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #328 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #329 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #330 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #331 pc 000000000049b99c  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.NodeTestTask.executeRecursively+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #332 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #333 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #334 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #335 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #336 pc 000000000049b8b0  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.NodeTestTask.execute+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #337 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #338 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #339 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #340 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #341 pc 000000000049c6e0  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.SameThreadHierarchicalTestExecutorService$$ExternalSyntheticLambda0.accept+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #342 pc 000000000034d5a8  /apex/com.android.art/lib64/libart.so (artQuickToInterpreterBridge+1932) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #343 pc 0000000000379098  /apex/com.android.art/lib64/libart.so (art_quick_to_interpreter_bridge+88) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #344 pc 0000000000164bbc  [anon_shmem:dalvik-jit-code-cache] (offset 0x2000000) (java.util.ArrayList.forEach+220)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #345 pc 0000000000362774  /apex/com.android.art/lib64/libart.so (art_quick_invoke_stub+612) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #346 pc 000000000035bd1c  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+1928) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #347 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #348 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #349 pc 000000000049c720  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.SameThreadHierarchicalTestExecutorService.invokeAll+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #350 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #351 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #352 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #353 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #354 pc 000000000049ba44  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$6$org-junit-platform-engine-support-hierarchical-NodeTestTask+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #355 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #356 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #357 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #358 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #359 pc 000000000049aec4  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.NodeTestTask$$ExternalSyntheticLambda0.execute+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #360 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #361 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #362 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #363 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #364 pc 000000000049caec  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #365 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #366 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #367 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #368 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #369 pc 000000000049bb38  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$8$org-junit-platform-engine-support-hierarchical-NodeTestTask+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #370 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #371 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #372 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #373 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #374 pc 000000000049aefc  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.NodeTestTask$$ExternalSyntheticLambda10.invoke+0)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #375 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #376 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #377 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.976 27612 27612 F DEBUG   :       #378 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #379 pc 000000000049c3c0  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.Node.around+0)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #380 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #381 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #382 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #383 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #384 pc 000000000049bb78  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$9$org-junit-platform-engine-support-hierarchical-NodeTestTask+0)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #385 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #386 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #387 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #388 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #389 pc 000000000049af34  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.NodeTestTask$$ExternalSyntheticLambda11.execute+0)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #390 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #391 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #392 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #393 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #394 pc 000000000049caec  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute+0)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #395 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #396 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #397 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #398 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #399 pc 000000000049b99c  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.NodeTestTask.executeRecursively+0)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #400 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #401 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #402 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #403 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #404 pc 000000000049b8b0  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.NodeTestTask.execute+0)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #405 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #406 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #407 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #408 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #409 pc 000000000049c6fc  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.SameThreadHierarchicalTestExecutorService.submit+0)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #410 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #411 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #412 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #413 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #414 pc 000000000049a6ac  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.HierarchicalTestExecutor.execute+0)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #415 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #416 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #417 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #418 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #419 pc 000000000049a5d4  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.engine.support.hierarchical.HierarchicalTestEngine.execute+0)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #420 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #421 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #422 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #423 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #424 pc 00000000004a2520  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.launcher.core.EngineExecutionOrchestrator.execute+0)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #425 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #426 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #427 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #428 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #429 pc 00000000004a2628  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.launcher.core.EngineExecutionOrchestrator.execute+0)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #430 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #431 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #432 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #433 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #434 pc 00000000004a2594  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.launcher.core.EngineExecutionOrchestrator.execute+0)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #435 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #436 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #437 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #438 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #439 pc 00000000004a2704  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.launcher.core.EngineExecutionOrchestrator.lambda$execute$0$org-junit-platform-launcher-core-EngineExecutionOrchestrator+0)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #440 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #441 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #442 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #443 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #444 pc 00000000004a2318  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.launcher.core.EngineExecutionOrchestrator$$ExternalSyntheticLambda3.accept+0)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #445 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #446 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #447 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #448 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #449 pc 00000000004a2720  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.launcher.core.EngineExecutionOrchestrator.withInterceptedStreams+0)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #450 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #451 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #452 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #453 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #454 pc 00000000004a25f4  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.launcher.core.EngineExecutionOrchestrator.execute+0)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #455 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #456 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #457 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #458 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #459 pc 00000000004a129c  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.launcher.core.DefaultLauncher.execute+0)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #460 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #461 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #462 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #463 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #464 pc 00000000004a1250  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.launcher.core.DefaultLauncher.execute+0)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #465 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #466 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #467 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #468 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #469 pc 00000000004a13f8  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.launcher.core.DelegatingLauncher.execute+0)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #470 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #471 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #472 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #473 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #474 pc 00000000004a5e04  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.platform.launcher.core.SessionPerRequestLauncher.execute+0)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #475 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #476 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #477 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #478 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #479 pc 00000000002bd5f0  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (de.mannodermaus.junit5.internal.runners.AndroidJUnit5.run+0)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #480 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #481 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #482 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #483 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #484 pc 00000000004b47d0  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.runners.Suite.runChild+0)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #485 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #486 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #487 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #488 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #489 pc 00000000004b47b0  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.runners.Suite.runChild+0)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #490 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #491 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #492 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #493 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #494 pc 00000000004b371c  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.runners.ParentRunner$4.run+0)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #495 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #496 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #497 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #498 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #499 pc 00000000004b3644  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.runners.ParentRunner$1.schedule+0)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #500 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #501 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #502 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #503 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #504 pc 00000000004b408c  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.runners.ParentRunner.runChildren+0)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #505 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #506 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #507 pc 000000000076da48  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false>(art::interpreter::SwitchImplContext*)+12208) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #508 pc 000000000037b5d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #509 pc 00000000004b3d5c  /data/app/~~mVADCp0l4u2tmGTofEPdqQ==/com.dylibso.chicory.android.aot.test-irCJzP1P57Ywof2WQebRvg==/base.apk (org.junit.runners.ParentRunner.access$100+0)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #510 pc 000000000034e21c  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.__uniq.112435418011751916792819755956732575238.llvm.2845697060370838518)+428) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.977 27612 27612 F DEBUG   :       #511 pc 000000000035c5b0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, bool, art::JValue*)+4124) (BuildId: dcb9fe2b5c99aa3f1a682a6008427d08)
07-11 11:18:25.985   216   216 E tombstoned: Tombstone written to: tombstone_31
07-11 11:18:26.006   378   378 I Zygote  : Process 27566 exited due to signal 11 (Segmentation fault)
07-11 11:18:26.006   434   434 I adbd    : Remote process closed the socket (on MSG_PEEK)
07-11 11:18:26.006   596   660 I libprocessgroup: Removed cgroup /sys/fs/cgroup/uid_10238/pid_27566
07-11 11:18:26.012   596   791 D InetDiagMessage: Destroyed 0 sockets, proto=IPPROTO_TCP, family=AF_INET, states=14
07-11 11:18:26.013   596   791 D InetDiagMessage: Destroyed 0 sockets, proto=IPPROTO_TCP, family=AF_INET6, states=14
07-11 11:18:26.013   596   791 D InetDiagMessage: Destroyed live tcp sockets for uids={10238} in 2ms
07-11 11:18:26.013 27556 27556 I app_process: System.exit called, status: 0
07-11 11:18:26.013 27556 27556 I AndroidRuntime: VM exiting with result code 0.
07-11 11:18:26.013   596   791 D InetDiagMessage: Destroyed 0 sockets, proto=IPPROTO_TCP, family=AF_INET, states=14
07-11 11:18:26.013   596   791 D InetDiagMessage: Destroyed 0 sockets, proto=IPPROTO_TCP, family=AF_INET6, states=14
07-11 11:18:26.013   596   791 D InetDiagMessage: Destroyed live tcp sockets for uids={20238} in 0ms
07-11 11:18:26.027   377   448 I netd    : tetherGetStats() -> {[]} <0.34ms>
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

Finally, we managed to make all the spec tests pass on Android. Great, now let's move onto some real-world Wasm modules! We started with the [mcp.run `fetch` servlet](https://www.mcp.run/dylibso/fetch), which is a Wasm module that fetches a webpage from a URL and returns it as Markdown. I was pretty bummed when ART started throwing a `java.lang.OutOfMemoryError`. Looking more closely, I noticed that the offender was the Chicory compiler itself, which was using more than **200MB** of heap space!

Now, everybody knows that the JVM has limitations on class file size and method count. The Dalvik runtime has similar constraints, with a limit of **65535 methods** per class file. So, on the JVM compiler, we generate multiple class files when the class file grows over 50k methods; if a function is too large, we just fall back to the interpreter. This works well on the JVM, but it is not enough on Android; it is mostly an issue related to the way DexMaker works. DexMaker retains the complete intermediate representation of all the generated methods in memory until the final DEX file is generated. This means that if you generate a lot of methods, the memory usage can grow significantly, especially if the methods are large.

Luckily, the solution is relatively straightforward: split class files at a lower boundary (currently 200 methods per "chunk"). This approach finally allowed us to "make [`fetch`](https://www.mcp.run/dylibso/fetch) happen"!

## Testing the Android backend

Android's development tooling is not designed for this kind of lower-level work. Testing Android applications on device, typically uses the [Android Instrumentation framework](https://developer.android.com/training/testing/). The instrumentation framework is a powerful tool but comes with significant overhead; among other things, it transfers a lot of metadata between the host and the device. This is ok for end-to-end, integration testing, where flows are usually long and complex, but not so much for unit testing, where you want to run a lot of tests quickly.

Moreover, while the instrumentation framework is the default way to run tests on the device, most unit tests are usually run on the host machine on a plain JVM; even when Android-specific APIs are involved, [Roboelectric](https://robolectric.org/) simulates the Android environment, allowing you to run tests without the overhead of the instrumentation framework.

Unfortunately, none of this applies to our case: the JVM can't load DEX bytecode, and the Chicory compiler _must_ be tested on ART.

Some work [has been done to run the test suite against the interpreter](https://github.com/dylibso/chicory/tree/main/android-tests). Even though there is some overhead, it is manageable, because the interpreter can be tested and debugged on a traditional JVM. On Android we only ensure that there aren't any platform-specific issues.

However, running the entire Wasm spec test suite against the Android compiler with the instrumentation framework takes about **8 minutes** in Android Studio, and about **4 minutes** using Gradle on the command line. This is not terrible, but it is still a lot of time to wait for a test suite to run, especially when you are iterating on the code. Compare this to the plain JVM, where the same test suite takes about **30   ** to run! (_FIXME: the JVM number is made up, need to measure again_)

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


### Running JUnit 5 tests on Android

Now, as an additional degree of complexity, the Chicory test suite uses JUnit 5, while the Android platform only supports JUnit 4 out of the box. There is an unofficial [JUnit 5 Android support library](https://github.com/mannodermaus/android-junit5) that you can use; it also supports the Instrumentation Framework, and this is what we've been using so far.

But if you run from the console, you can just use the [JUnit Platform Console Launcher](https://junit.org/junit5/docs/current/user-guide/#running-tests-console-launcher), and add it to your dependencies; you will also need to ensure that is bundled in the APK: that means that all the dependencies mut be `androidTestImplementation` instead of `androidTestRuntimeOnly`!

```kotlin
    androidTestImplementation("org.junit.jupiter:junit-jupiter-api:5.13.1")
    androidTestImplementation("org.junit.jupiter:junit-jupiter-engine:5.13.1")
    androidTestImplementation("org.junit.platform:junit-platform-console:1.13.1")
    androidTestImplementation("org.junit.platform:junit-platform-suite:1.13.1")
```

It is also advisable to create a Suite and list all the classes you want to run, as classpath scanning will not work. In our case:

```
package com.dylibso.chicory.test.gen;

import org.junit.platform.suite.api.*;

@SelectClasses({
        SpecV1AddressTest.class,
        SpecV1AlignTest.class,
        ...
        SpecV1Utf8ImportModuleTest.class,
        SpecV1Utf8InvalidEncodingTest.class,
})
@Suite
public class SpecV1 {}
```

Then once built and uploaded to `/tmp/main.apk`:

```sh
./gradlew assembleAndroidTest
adb push MY_PROJECT/build/outputs/apk/androidTest/debug/MY_PROJECT-debug-androidTest.apk /tmp/main.apk
adb shell /apex/com.android.art/bin/dalvikvm64 -cp /tmp/main.apk com.example.Main
```


you will finally be able to run the tests with the following command:

```
adb shell /apex/com.android.art/bin/dalvikvm64 -cp /tmp/main.apk org.junit.platform.console.ConsoleLauncher -c com.dylibso.chicory.test.gen.SpecV1
```

The boot might take a little time: you can follow on `logcat` if you are bored, and you'll see that something is indeed happening; eventually you will see the test results in the console!

This finally brings the test time down to about **14 seconds**, which is a lot more manageable than the **8 minutes** we had before!


### The Hard Way: Running Tests with a Custom Launcher

Can we bring this down further? After all, the spec test suite is generated code, we know exactly what it does, and we could run it without the JUnit framework. [With this custom launcher](#FIXME), we can write a simple `Main` class that runs the tests directly, without the JUnit framework:

```java
    public static void main(String... args) throws Exception {
        var tests = Class[] {
            SpecV1AddressTest.class,
            SpecV1AlignTest.class,
            // ... all the other test classes ...
            SpecV1Utf8ImportModuleTest.class,
            SpecV1Utf8InvalidEncodingTest.class,
        };
        var runner = new SpecTestRunnerMain(8 * 1024 * 1024, tests);
        runner.runSuiteOnThread();
    }
```

The entire suite runs now in about **6 seconds** on the Android device.


### Debugging the Android backend

It is also possible to attach a debugger to the Android process running the tests. This is useful to debug the Chicory compiler and the generated Dalvik bytecode. In this case, the instructions are even more hard to find, but with some trial and error, especially following the `logcat` output I was able to figure out that you can attach a debugger to the Android process running the tests by using the following command:

```sh
/apex/com.android.art/bin/dalvikvm64 -Xplugin:libopenjdkjvmti.so -agentpath:libjdwp.so=server=y,suspend=y,transport=dt_socket,address=0.0.0.0:8000 -cp /tmp/main.apk com.example.Main
```

Then you will be able to attach a remote debugging session using your favorite IDE (e.g. IntelliJ IDEA or Android Studio) to the port `8000` on the Android device.

The critical part is `-Xplugin:libopenjdkjvmti.so`, otherwise the `-agentpath` flag has no effect.

## Conclusion

The Android backend of the Chicory compiler is still a work in progress, but it is showing promising results. If you are interested in lower-level Android development, or if you want to run Wasm modules on Android, I hope this article has given you some insights into the challenges and solutions we have found so far. If you want to try it out follow the instruction on the [https://github.com/dylibso/chicory-compiler-android][chicory-compiler-android] repository, and feel free to open issues if you find any problems or have suggestions for improvements.


[chicory-android]: https://docs.mcp.run/blog/2024/12/27/running-tools-on-android/
[chicory-compiler]: https://chicory.dev/blog/chicory-1.4.0
[sqlite4j]: https://github.com/roastedroot/sqlite4j
[chicory-sdk]: https://github.com/extism/chicory-sdk
[mcpx4j]: https://github.com/dylibso/mcpx4j
[mcp-run]: https://www.mcp.run