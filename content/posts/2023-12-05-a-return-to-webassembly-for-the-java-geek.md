---
date: "2023-12-05T00:00:00Z"
tags: 
  - wasm
  - java
title: A Return to WebAssembly for the Java Geek
---

[![](https://i0.wp.com/www.javaadvent.com/content/uploads/2023/11/java-geek-wasm-2.png?resize=600%2C369&ssl=1)](https://i0.wp.com/www.javaadvent.com/content/uploads/2023/11/java-geek-wasm-2.png?ssl=1)

<div style="padding: 1em; background: #eee">
<b>Update (Jun 20, 2025)</b>. This post was originally published at <a href="https://www.javaadvent.com/2023/12/a-return-to-webassembly-for-the-java-geek.html">JVM Advent</a>.
</div>


Welcome back, my dear **Java Geek!**

[Last year we compared WebAssembly and discussed in what ways it differs from the JVM](https://www.javaadvent.com/2022/12/webassembly-for-the-java-geek.html). A lot of things have happened in the meantime. If you want to dive deeper into that kind of detail, I warmly suggest reading [this beautiful blog series by Chris Dickinson](https://www.neversaw.us/2023/05/10/understanding-wasm/part1/virtualization/). 

For this Java Advent, I wanted to get back to the topic **from a different angle**. Last year we saw that, because the JVM and WebAssembly are only shallowly similar, there is friction when it comes to putting the two together. 

But this is not just a matter of taste; a Wasm VM is generally **simpler, smaller, and easier to embed than a full, modern JVM**. This on the one hand, might **allow JVM languages to run in spaces that they normally would not fit** (for instance, in a plug-in system for a native executable); and, on the other hand, it might **allow the JVM to host languages that normally would not be supported** (by implementing Wasm support *on top* of a JVM).

[Last year we listed a few projects](https://www.javaadvent.com/2022/12/webassembly-for-the-java-geek.html#java-support-on-webassembly) that were starting to approach the space of compiling JVM bytecode to Wasm bytecode, and a few others that were addressing hosting a Wasm binary on top of a JVM.

In this post, we will see **what things have changed** since last time, and we will **revisit** some of those projects. Has the ecosystem matured? And **in what ways should you, the Java Geek, care?**

## **What’s With All The Fuss?**

[![](https://i0.wp.com/www.javaadvent.com/content/uploads/2023/11/jdk-jre-wasm.png?resize=600%2C439&ssl=1)](https://i0.wp.com/www.javaadvent.com/content/uploads/2023/11/jdk-jre-wasm.png?ssl=1)

Original: [Jave SE Platform at a Glance](https://www.oracle.com/java/technologies/platform-glance.html)

In essence, you can think of a Wasm VM as a **JDK without the class library,** with no built-in way to interact with its surroundings, except **importing and exporting functions.** If some function you import is able to perform **I/O,** then **great!** you can actually do something useful, otherwise, you have got yourself a fancy calculator.

Then, if it is so *useless,* why are we interested in it *at all?* The Wasm VM is *small*, so it is relatively easy to implement, and thus, it is also easy to port to many platforms. An unmodified Wasm VM is able to run *in a browser* as well as *outside a browser*. And, if you remember, ***this* is the thing I am personally more excited about**.

Now, in the last year, you might have heard that Wasm VMs are being proposed as an alternative to **containers** because you can produce a self-contained binary that will run in a *sandboxed* environment: this is an ambitious goal, and people are making some progress there. 

However, there is also another, much *less ambitious* use case, where I feel a greater potential lies: because Wasm VMs are relatively small and easy to *embed*, they are great for implementing **sandboxed plug-in and extension systems.**

When a Wasm VM is embedded to provide a plug-in system, the Wasm VM is extended by providing a collection of functions that the Wasm binaries can invoke. A proposed standard called [**WASI**](https://wasi.dev/) provides a set of functions that, if you squint enough, could be considered some sort of POSIX compatibility layer, or, more generally, a set of OS-like primitives to access things like the console, a file system and, in some cases, even the network. A subset of these APIs is often expected to be available, usually to be able to provide simple input and output capabilities (such as logging). Each language usually rebuilds its own standard library on top of these APIs, with the aim of making a port easier for the end-users.

Several projects have adopted Wasm as an extension language; for instance, the [**Envoy proxy**](https://www.envoyproxy.io/) allows writing middleware through a Wasm interface; [**Redpanda**](https://redpanda.com/) is a streaming platform implementing the Kafka protocol, that allows writing data transforms in-core using Wasm. The list could go on and it is always growing.

The [**Extism**](https://extism.org/) project by [**Dylibso**](https://dylibso.com/) is proposing a unified API to provide cross-platform extensions and plug-ins, regardless of the language: they provide a battery-included system to experiment with Wasm plug-ins. This includes the JVM, where the underlying VM is currently [**Wasmtime**](https://wasmtime.dev/), and Go, where the VM is [**wazero**](https://wazero.io/).

The JVM has always provided ways to load code dynamically. I think we could even dare to say that **class loading is in fact a *defining feature* of a JVM,** so, it is pretty easy to write a plug-in system that loads class files or jar files. However, as with any language VM, the JVM has always been limited to the languages that in fact support it.

As the Wasm landscape expands, more and more languages are deciding to target it. For instance, there is support for C, C++, Rust, Go, Python, Ruby just to name a few. Even the [.NET ecosystem has added support to Wasm in its toolchain](https://www.youtube.com/watch?v=jkve_v1Xxak). [Fermyon keeps a list of languages with their degree of support](https://www.fermyon.com/wasm-languages/webassembly-language-support). 

So for a **Java Geek** there are two opportunities here: 

1. **JVM languages can support Wasm as a compilation target** so that they can be used to write Wasm binaries that will run outside a traditional JVM and inside a Wasm VM. This could be used to write software for the browser, as well as plug-ins for other software.
2. **JVM may run Wasm binaries to support languages** that traditionally would not be available on a plain JVM.

## **What Has Changed Since Last Time?**

Quite a lot! First of all, let me talk just a little about my favorite topic in the whole world, i.e. ***myself***. I am no longer at Red Hat, and I have joined [Tetrate](https://tetrate.io/) to join the team of, guess what, [an open-source WebAssembly **runtime** that I mentioned earlier **called wazero.**](https://wazero.io/) In the process, I also had to switch to a different language in my day-to-day; that is, Go, since that’s wazero’s language.

***Burn in hell you, traitor! —*** I hear you say. But, in my defense: first, I still hold the JVM dear to [my achy-breaky heart](https://www.youtube.com/watch?v=byQIPdHMpjc), and, second, I think getting exposed to a different ecosystem helps you to see the bigger picture. And in fact, there are a few things that all these ecosystems (JVM, Go, Wasm) could learn from each other.

But enough about me! While I was busy learning Go, the JVM space has also started to get more involved in Wasm. And, at the same time, Wasm **has gained a few features that, in the future, might make it easier for JVM languages to target it**. For instance, in the previous post, we mentioned the lack of support for **threading, exception handling, and garbage collection**. All these features **made progress**, and they are slowly starting to get experimental support in some languages.

## **Caffeinated Gophers**

[![Duke, the Java Mascot is next to a Gopher (the Go mascot) holding a cup of coffee which resembles the Java logo.](https://i0.wp.com/www.javaadvent.com/content/uploads/2023/11/gojava.png?resize=600%2C395&ssl=1)](https://i0.wp.com/www.javaadvent.com/content/uploads/2023/11/gojava.png?ssl=1)

As surprising as it may sound, while working on **wazero** I have learned that **there are in fact similarities between the Go and the Java runtime** (at least in principle); for instance:

- **compiling Go and Java to a Wasm binary** requires some ***massaging***. But such a *massage* is not that different from building a native executable. We will see in what ways in the next section
- **evaluating a Wasm binary on the JVM and the Go runtime can be** achieved by implementing a Wasm runtime on top of them or depending on an existing runtime. Both the JVM and Go have similar caveats (we will see them later) when it comes to depending on a native library, so writing a Wasm VM specific to that language might be more convenient.

Let us see both aspects in detail.

## **Compiling JVM Software into Wasm**

[![](https://i0.wp.com/www.javaadvent.com/content/uploads/2023/12/java-on-wasm.png?resize=600%2C450&ssl=1)](https://i0.wp.com/www.javaadvent.com/content/uploads/2023/12/java-on-wasm.png?ssl=1)We already mentioned targeting the Wasm bytecode in its current version of the spec (2.0) is **more similar to targeting a native platform** than a high-level language VM like the JVM. 

It is not by chance that **many of the languages that support Wasm as a compilation target are based on the** [**LLVM toolchain**](https://llvm.org/)**.** Because the LLVM toolchain **supports Wasm among its native targets, it is *relatively easy* to add support for it**.

Thus, it is no surprise that C and C++ support Wasm via Clang/LLVM, that Rust supports Wasm, Zig supports Wasm, and that the [**TinyGo**](https://tinygo.org/) flavor of the Go language gained support for Wasm relatively early on. Guess what common trait they all share? That’s right, they all leverage LLVM. 

### **Wasm as a Native Target**

But in what ways does Wasm behave as a native target, rather than, more intuitively, as JavaScript? 

Well, for instance, Wasm 2.0 does not provide primitives to manipulate **structured data**. It only deals with **numbers** – integers and floats (and 128-bit vectors, but that does not make much difference). 

So, what if you want to deal with **fancy** data structures such as — *gasp!* — **arrays**? Well, fear not: Wasm 2.0 provides to *each module* its own, *isolated* **linear memory space;** which is a cool way to call **an array of unsigned bytes** that your program will treat as if it were real operating system memory.  Thus, if you need to allocate an **array**, then you just reserve a *slice* of that larger array. 

And what if you need a ***fancier*** structure such as ***strings***? Well, you just reserve a *slice* of that array for the characters, and then maybe some extra meta-data for the size (depending on how your language represents strings). 

And what if you need an even ***fancier*** data structure such as **Point(int x, int y)**? Well, **you get the idea**.

Now, as long as you only have to ***allocate***, things will work out pretty well. You can just keep track of the last allocation you did; that is, *effectively* keeping an *index* into the array. Which is another way to say, err, **a pointer**. And every time you allocate more, you can just update that *pointer* I mean *index*. 

However, at some point, **that memory space will finish**. And obviously, we don’t want that to happen, so you also want to keep track of things you no longer need, and ***free*** that space. And you want to keep things neat and avoid fragmentation. And there you go: you got yourself a ***memory manager*.** Or, as some runtimes call it, a **garbage collector.**

**Note.** It is worth mentioning that for all intents and purposes, the *linear memory space* can be thought about as real operating system memory: however, there is one big caveat that makes it quite different; such a memory space is *not shared across Wasm programs,* even within the same VM, and even across modules. Each module gets its own, **isolated** memory space, and the only way to pass structured data across modules is essentially to copy it over. This is a big difference that is often highlighted as one of the benefits of Wasm binaries over traditional, native binaries.

All of this might evolve in the future because [the WasmGC spec has moved further](https://v8.dev/blog/wasm-gc-porting) and the multiple memories and threading proposals have moved too. The WasmGC spec deals exactly with allocating and deallocating structured data and delegating memory management to the underlying runtime. The flat memory space will still be available, but a compiler may pick the allocation strategy that suits the language best. The “multiple memories” proposal allows modules to define different, isolated memory spaces, while the threading proposal includes a way to declare a memory space as *shared.* However, at the time of writing the WasmGC proposal is only supported in browsers, and the other two are not widely available yet; until all these proposals gain wider adoption, a bare-bone Wasm 2.0 runtime only provides a flat, linear memory space as described above.

So, from the JVM perspective, there are **two strategies** to target Wasm: 

- compiling a **JVM** into an executable and then letting it load and evaluate class files
- compiling a JVM application into a self-contained executable, from **bytecode** similar to a **native image**
- compiling a JVM application into a self-contained executable, from **source code** with a **language-specific** compiler.

### **Compiling a JVM into Wasm**

In the first case, we are effectively **porting and compiling *a JVM* into Wasm**, and then we are *evaluating JVM bytecode inside such a JVM that runs inside a Wasm VM.* If this gives you a headache, that’s alright. It is a little mind-bending. However, this is a perfectly valid approach, and it is also the approach dynamic languages such as Ruby and Python are adopting. But *is this the *best* approach?* As usual, *it depends on your use case and your performance requirements.* This approach potentially allows for the largest degree of compatibility, with fewer limitations. 

There is at least one project that is doing exactly *that*: the fine people at [Leaning Technologies](https://leaningtech.com/) are developing [a JVM that runs in-browser (CheerpJ)](https://cheerpj.com/) that is especially well-suited to modernize legacy software that would require, say, an applet runtime (they also provide a [browser extension that does exactly that](https://chromewebstore.google.com/detail/cheerpj-applet-runner/bbmolahhldcbngedljfadjlognfaaein?pli=1)). 

However, a modern **JVM tends to be large**; as such, a Wasm binary of this kind might not be well suited to write *tiny executables such as plug-ins and extensions.* 

### **Compiling Java Bytecode into Wasm**

This is the most general approach. If you are able to compile Wasm into bytecode, then potentially all Java software can be compiled into Wasm. This is similar to the approach that GraalVM Native Image takes to produce a native executable. In fact, Native Image would be the most natural candidate, to the point that this was mentioned as a possibility [in the post about RISC-V support which, guess what, leveraged LLVM](https://medium.com/graalvm/graalvm-native-image-meets-risc-v-899be38eddd9).

Because it is the most general approach, just like the Native Image Builder: it should deal with all the worst cases, and cannot make any assumptions about the program that will be run.

- In order to preserve the semantics of your program, you will have to emulate most of the features of the JVM, including, in some form, reflection, and class loading (even if with some limitations, such as the infamous “closed world assumption”).
- **you want to reduce the program surface as much as possible,** just like GraalVM’s Native Image Builder does when it produces a native executable: this however may impose limitations on reflection and class loading (the infamous “closed world assumption”).
- Then, just like the Native Image Builder, you will still have to **ship a full-blown garbage collector.**
- Finally, to keep your boot time low, you might want to move some computation at build time.

At the time of writing, there are at least **two projects** that are able to compile class files into Wasm [**JWebAssembly**](https://github.com/i-net-software/JWebAssembly) and [**TeaVM**](https://teavm.org/)**.** However, if you want to produce a self-contained Wasm executable that runs **outside the browser, TeaVM** is the most promising project so far. 

If you are lazy like me, I found that in order to get started with **TeaVM,** the most effective way is to [**clone the repository**](https://github.com/konsoletyper/teavm), build with ./gradlew publishToMavenLocal, and then try out the example under samples/pi which is already configured for **WASI** support. The program computes the first **N** digits of π a given **N,**  supplied via command line argument, then prints them with the elapsed time., and, if the build was **successful** you will find a pre-built Wasm binary under samples/pi/build/libs/wasi/pi.wasm

You can test it out with your favorite Wasm runtime, that is, obviously **wazero.** Just kidding, obviously, the choice is relevant, the output will be always the same; that’s the point after all!

```
❯ wazero run build/libs/wasi/pi.wasm 3
314        :3
Time in millis: 0

❯ wazero run build/libs/wasi/pi.wasm 5
31415      :5
Time in millis: 0


❯ wazero run build/libs/wasi/pi.wasm 10
3141592653 :10
Time in millis: 0

❯ wazero run build/libs/wasi/pi.wasm 100
3141592653 :10
5897932384 :20
6264338327 :30
9502884197 :40
1693993751 :50
0582097494 :60
4592307816 :70
4062862089 :80
9862803482 :90
5342117067 :100
Time in millis: 7
```

#### *Appendix: Does It Really Boot Fast?*

It is often claimed that **Wasm VMs are super-fast to boot**. This is not false, but the reason is **kind of underwhelming**; there is no secret sauce: **they start fast because they don’t need to do much anyway**. In a typical Java program, a [JVM might need to load and initialize thousands of classes before it reaches a steady state](https://www.youtube.com/watch?v=pGHN8a3VAeU). All these initializers add up, and that is the reason why the Native Image Builder makes the pragmatic choice of moving some of that computation at build time, taking a snapshot of that heap, and then restoring it at boot time to get reasonable startup performance. 

Even Wasm modules may define a startup function to perform initialization. Guess what you might need to do in Wasm too if you want to keep those precious milliseconds down? 

It is interesting how the Wasm community has already produced [a tool to perform build-time initialization called **wizer**](https://github.com/bytecodealliance/wizer). Instead of producing a native binary, **wizer** produces a new Wasm binary, that is, what [Project Leyden would call a **condenser**](https://openjdk.org/projects/leyden/notes/03-toward-condensers)**.**

### **Compiling Source Code into Wasm**

Some JVM languages have supported compiling to a different target for a long time. However, in my research, I have found that *in general* the primary target for these compilers is execution *in the browser*. I will still give a brief overview of these alternative compilers for completeness.

The **Scala** compiler gained soon support for targeting JavaScript with [Scala,js](https://www.scala-js.org/). And, later, the [Scala Native project](https://scala-native.org/en/stable/) started to explore the space of native compilation through an **LLVM-based backend**. More recently this Native backend has experimented with [adding support to target Wasm](https://www.youtube.com/watch?v=mrqKw1PeshI). However, this experiment needs support for features such as automated garbage collection and exception handling, that are emulated via JavaScript shims (this is generated automatically by the **Emscripten** toolchain, that you can think of as an *extension to LLVM*).

The **Kotlin** compiler was born with multi-platform in mind: Kotlin supported a JavaScript output for front-end development since its very first versions. It is only natural to support Wasm with the same goal. The Kotlin compiler for Wasm (Kotlin/Wasm) originally was born as an extension to the Kotlin/Native backend (based on LLVM). The most recent version, however, targets Wasm *directly*, and, in particular, it leverages the WasmGC proposal, which, at the time of writing, has been enabled in Chrome *and* Firefox. [Node, being based on V8 like Chrome, it supports it behind a flag](https://webassembly.org/roadmap/), and [Deno has been reported to run Kotlin fine](https://twitter.com/bashorov/status/1728199701588304069).

The **GWT compiler** for Java evolved into the [**J2CL**](https://github.com/google/j2cl) **compiler** in recent years. Originally targeting Java source-to-JavaScript compilation, it has also become [a test bed for experimenting with the WasmGC spec](https://github.com/google/j2cl/blob/master/docs/getting-started-j2wasm.md).

## **Evaluating Wasm Binaries on the JVM**

[![](https://i0.wp.com/www.javaadvent.com/content/uploads/2023/12/wasm-on-java.png?resize=600%2C450&ssl=1)](https://i0.wp.com/www.javaadvent.com/content/uploads/2023/12/wasm-on-java.png?ssl=1)

In the introduction, we briefly mentioned that Wasm might bring support to languages that otherwise would not be available on the JVM. But we have JRuby, we had Jython, and with Truffle we have other, state-of-the-art dynamic language implementations, including JavaScript: while these get best-in-class performance only when run on a GraalVM JDK, they are still portable and work on any JVM. So the question might be: why would you need Wasm, if you already have GraalVM?

For different reasons, [we briefly discussed GraalVM in the previous post](https://www.javaadvent.com/2022/12/webassembly-for-the-java-geek.html#graalvm-one-vm-to-rule-them-all). While Truffle/GraalVM supports a number of languages with great performance it still requires implementing such language-specific support from scratch. There is one Python implementation, one Ruby implementation, one R implementation, one JavaScript implementation… you get the idea.

But, as we have seen earlier, Wasm was designed as a **compilation target**, and a lot of compiler toolchains already support it. This means that with relatively few changes, it is often possible to bring Wasm support to **first-party language implementations**. For instance, the Python and the Ruby runtimes that run on Wasm are the traditional [CPython](https://en.wikipedia.org/wiki/CPython) and [CRuby (Ruby MRI)](https://en.wikipedia.org/wiki/Ruby_MRI) runtimes, with obvious compatibility benefits.

### **Picking a Wasm Runtime for the JVM**

Assuming that we have now bought into Wasm as a way to host end-user extensions in our Java environment, the most complete and battle-tested implementations of a Wasm VM are written in languages such as C/C++ and Rust. These are native libraries that will require some form of **integration**.

Now, while Java is improving the developer experience **over JNI** with [**Project Panama (**finally being released with JDK 22)](https://openjdk.org/jeps/454) linking against a native library still imposes a **number of restrictions**.

[![](https://i0.wp.com/www.javaadvent.com/content/uploads/2023/11/gojni.png?resize=600%2C465&ssl=1)](https://i0.wp.com/www.javaadvent.com/content/uploads/2023/11/gojni.png?ssl=1)

Interestingly enough, **this is another place where the Go runtime oddly behaves like a Java Runtime.** While the developer experience for Go developers is probably better than JNI, linking against a non-Go, native binary requires a Go developer to reach for the **Cgo** system. This is completely transparent from a development perspective, as it is just a matter of importing the right library. But as you opt-in to CGo, under the hoods, the compilation pipeline changes dramatically: it requires your system to provide a C compiler, and cross-platform build capabilities that usually work out-of-the-box, will require much more work.

The restrictions imposed by both JNI/Panama and Cgo are essentially the same: 

- **There are portability concerns**, because you will have to compile the native library for all the platforms you want to support, and this obviously hinders the portability of your code
- **There is overhead crossing the boundary** to and from the managed environment to “native code”
- **There are security and safety implications** because the native library has access to the entire space of the process memory
- **There are runtime concerns** because every native call hogs an operating system thread: this will not play nicely with **virtual threads** (i.e., in the case of Go, **goroutines**).

This means that, while it is perfectly possible for a native Wasm runtime to be imported into a Java or Go application, this comes with costs that have to be carefully evaluated, and that may ultimately lead to avoiding adopting it altogether.

This is the reason for the **wazero** project: it is a **zero-dependency WebAssembly runtime for Go,** where zero means literally *no dependencies,* but *in particular*, zero **Cgo** dependencies. So, depending on it and using it, for a Go developer is a no-brainer: there is essentially no overhead, and you keep all the benefits of your Go runtime. So, what about Java? Is there anything similar? 

Indeed, there are. The **GraalVM** project already proved that it is possible to run a lower-level compilation target *on top of Truffle*: this is called Sulong, and it is an implementation of a runtime for the **LLVM IR**, that is, the ***I*ntermediate *R*epresentation** that a compiler based on LLVM uses internally, that then, in the final stages of the compilation pipeline, gets translated into the target architecture.

So, there is an experimental [GraalVM Wasm language implementation for Truffle](https://www.graalvm.org/latest/reference-manual/wasm/). Obviously, besides this being experimental, it is also worth noticing that, as it is for all the Truffle-based language implementations, you will need a GraalVM JDK to get the best performance out of it.

I also wanted to mention another project that is being developed by some friends, and I’m keeping myself involved in it from afar, called [**Chicory**](https://github.com/dylibso/chicory). **Chicory** is a Wasm VM implementation that aims to support the entire spec. It currently does not aim for best-in-class performance, but focuses on correctness, by implementing a **Wasm interpreter** [validated against the Wasm test suite](https://github.com/WebAssembly/testsuite). Nonetheless, the people involved are already considering adding support for a bytecode translation layer, which potentially could provide reasonable performance. Chicory was initiated by one of the founders at **Extism** (the Wasm plug-in system), so one of the goals will be to rebase the Extism SDK for Java on top of it, once it is mature enough.

## **Conclusions**

I could go on and on about Wasm and this article has reached a considerable length already. The space is always evolving and for a newcomer, it might be intimidating to get started. I mentioned all of the ways Wasm could be useful from the perspective of the Java Geek, but I also **overlooked some important limitations** that will need to be addressed before Wasm can expect to gain a wider adoption, beyond early adopters and enthusiasts. 

For instance, to this day, there are **few options for debugging** (especially dire is the landscape when it comes to stepwise debugging, where tooling is still dramatically limited — [see for instance this recent talk by Ashwin Kumar Uppala and Shivay Lamba](https://www.youtube.com/watch?v=JqvwzkRatOg)).

The work to stabilize the **WASI** set of APIs is also ongoing. 

Finally, there is a lot of buzz around the so-called [**Component Model**](https://component-model.bytecodealliance.org/). The component model aims to provide a polyglot system to define APIs and compose libraries together, while retaining the safe, isolated architecture of the Wasm VM (remember: memory is not shared by default). These are however early days and the work here is still in flux.

I still hope that this new article has caught your attention; in the meantime, enjoy your ***panettone*** and have a sip of ***spumante***, and see you at some conference in **2024**!

