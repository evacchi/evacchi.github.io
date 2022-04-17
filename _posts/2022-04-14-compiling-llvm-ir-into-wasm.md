---
title:  'Compiling LLVM IR into WebAssembly with WASI support'
subtitle: ''
categories: [LLVM, WASM, WASI]
date:   2022-04-14
---

I have been experimenting a little bit with LLVM and its WASM backend. Unfortunately, although it is easy to find examples on how to compile a "hello world" using C, resources are a bit scattered when it comes to doing something a little more specific like **compiling LLVM IR into WASM** with [WASI support][wasi].

I found [Surma's article on how to compile C into WASM from scratch][surma] to be a very valuable resource, and [Frank Denis's article on Compiling C to WebAssembly using clang/LLVM and WASI][frank-denis] was especially useful, even though it is now a bit outdated.   

So here is an easy-to-follow, up-to-date, step-by-step guide. 

## Setup

1. Make sure you have  installed an `llvm` version that supports `WASM`. For instance, on macOS, system `clang` does not support `wasm` compilation. I installed `llvm` 13 using `brew`.

        ❯ llc --version
        Homebrew LLVM version 13.0.1
        Optimized build.
        Default target: x86_64-apple-darwin21.4.0
        Host CPU: icelake-client

        Registered Targets:
            ...
            wasm32     - WebAssembly 32-bit
            wasm64     - WebAssembly 64-bit
            ...

2. Download the [WASI SYSROOT][wasi-sysroot] matching your version of LLVM. For instance, in my case, version 13.

        curl -LO https://github.com/WebAssembly/wasi-sdk/releases/download/wasi-sdk-13/wasi-sysroot-13.0.tar.gz
        tar xzvf wasi-sysroot-13.0.tar.gz

    You have now extracted the `wasi-sysroot` in your current directory. For convenience, you can 

        export WASI_SYSROOT=$PWD/wasi-sysroot

    This will be useful later.

3. Download `libclang` for `wasm32` matching your version of LLVM.


        curl -LO https://github.com/WebAssembly/wasi-sdk/releases/download/wasi-sdk-13/libclang_rt.builtins-wasm32-wasi-13.0.tar.gz
        tar xzvf libclang_rt.builtins-wasm32-wasi-13.0.tar.gz

    This will extract `lib/wasi/libclang_rt.builtins-wasm32.a` to your current directory.

## Compiling LLVM IR into WASM

We are are now ready to go. Let us create a simple program `wasi.ll`
that prints "Hello\n" to standard output using LLVM IR.

```
target triple = "wasm32-unknown-wasi"

define i32 @main() #0 {
  call i32 @putchar(i32 72)
  call i32 @putchar(i32 101)
  call i32 @putchar(i32 108)
  call i32 @putchar(i32 108)
  call i32 @putchar(i32 111)
  call i32 @putchar(i32 10)
  ret i32 0
}

declare i32 @putchar(i32) #1
```

**Update 2022-04-16**: Shortly after I published this blog post 
[I have learned from Dan Gohman (@Sunfishcode)][sunfishcode] that the easiest way 
to compile LLVM IR sources (`*.ll`) with WASI support is to rely on `clang`. 

In this case, this is the proper command line:

    clang wasi.ll --target=wasm32-unknown-wasi --sysroot=$WASI_SYSROOT -lc \
        $PWD/lib/wasi/libclang_rt.builtins-wasm32.a -o wasi.wasm

Unless you are doing something very specific and you know what you are doing, you should not invoke
`wasi-ld` directly, as the interface will be subject to changes in the future.

If you understand the risks of invoking the tools manually:

1. compile `wasi.ll` into an object file (`wasi.o`) using `llc`:

        llc -march=wasm32 -filetype=obj wasi.ll

2. produce the final binary from `wasi.o` using `wasm-ld`:

        wasm-ld -m wasm32 -L$WASI_SYSROOT/lib/wasm32-wasi \
            $WASI_SYSROOT/lib/wasm32-wasi/crt1.o wasi.o -lc \
            $PWD/lib/wasi/libclang_rt.builtins-wasm32.a -o wasi.wasm


Regardless how you generated `wasi.wasm`, that's it! Now you can run our boring program with your favorite `wasm` runtime:

    wasmtime wasi.wasm
    Hello

Have fun!

[wasi]: https://wasi.dev/
[surma]:  https://surma.dev/things/c-to-webassembly/ 
[frank-denis]: https://00f.net/2019/04/07/compiling-to-webassembly-with-llvm-and-clang/
[wasi-sysroot]: https://github.com/WebAssembly/wasi-sdk/releases/
[sunfishcode]: https://twitter.com/Sunfishcode/status/1514818972251688960




