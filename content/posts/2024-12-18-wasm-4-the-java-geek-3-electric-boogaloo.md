---
title: 'Wasm 4 the Java Geek 3: Electric Boogaloo'
author: 'Edoardo Vacchi'
date: 2024-12-18
---


![](https://i0.wp.com/www.javaadvent.com/content/uploads/2024/12/wasm-4-java-geek-3.jpg?w=1292&ssl=1)


<div style="padding: 1em; background: #eee">
<b>Update (Jun 20, 2025)</b>. This post was originally published at <a href="https://www.javaadvent.com/2024/12/wasm-4-the-java-geek-3-electric-boogaloo.html">JVM Advent</a>.
</div>


And here we are again. For the third time in a row, we are back to the Java Advent, eager to discover what’s new with WebAssembly from a Java developer perspective.

Incidentally, since, as you know, I have a [favorite topic](https://www.javaadvent.com/2023/12/a-return-to-webassembly-for-the-java-geek.html#:~:text=WHAT%20HAS%20CHANGED%20SINCE%20LAST%20TIME%3F) (after programming languages and compilers, of course), it is also the **third time in a row** I have worked at a different company.

The [first time](https://www.javaadvent.com/2022/12/webassembly-for-the-java-geek.html) I was at Red Hat, [last year](https://www.javaadvent.com/2023/12/a-return-to-webassembly-for-the-java-geek.html) I was at Tetrate, and this year I joined [Dylibso](https://dylibso.com). Maybe [**Dylibso**](https://dylibso.com) does *jingle* a bell (see what I just did there? Come on, it’s Java *Advent*), or maybe it does not. But if you have followed this award-winning series of blog posts, you might remember Dylibso [for the Extism framework](https://extism.org). Extism is an easy-to-use cross-platform framework to write plugins for several languages using the same API across languages.

To that end, we use and contribute to quite a few runtimes; for instance, we use [wazero](https://wazero.io) (the runtime [I have been contributing to for the last year and a half at Tetrate](https://www.javaadvent.com/2023/12/a-return-to-webassembly-for-the-java-geek.html#:~:text=WHAT%20HAS%20CHANGED%20SINCE%20LAST%20TIME%3F)) for our [Go SDK](https://github.com/extism/go-sdk). Dylibso started and have been sponsoring the development of the pure-Java [Chicory runtime](https://chicory.dev) which has recently hit its [first 1.0 milestone release](https://chicory.dev/blog/chicory-1.0.0-M1). We’ll talk more about that later!

Finally, we have also just released a cool new [product called **XTP** that builds and extends Extism](https://getxtp.com) with type-safe bindings, codegen and a plug-in delivery system! More on that later too!

## Is This 🫴🦋 About Front-End?

Well, yes, and no. Have you been living under a rock or is this your first Java Advent? Just kidding.

<div style="width:375px;float:left">

[![The "is this a pigeon meme", but the guy looks a too frowning.](https://i0.wp.com/www.javaadvent.com/content/uploads/2024/12/is-this-a-pigeon-srsly.png?resize=365%2C273&ssl=1)](https://i0.wp.com/www.javaadvent.com/content/uploads/2024/12/is-this-a-pigeon-srsly.png?ssl=1)

Dude, seriously?
</div>


Sure, Wasm was originally created to bring a safe, sandboxed execution environment to the browser, leveraging the existing JS runtime. But the Wasm spec is relatively smaller than the bulk of a full JS implementation, and, as such, a constellation of pure-Wasm runtimes was born: these smaller, lighter-weight runtimes can be used as stand-alone language VMs (similarly to a JVM), or as embededded language hosts, as you would usually do for a scripting language, except that they are language-agnostic.

In fact, Wasm is a **language-agnostic compilation target** and any compiler is free to generate it: that makes it a great binary format to distribute **cross-platform software extensions** (i.e. plug-ins) that can be hosted in *any* “host” application.

In the last few years, we have seen the CNCF cloud-native landscape extend, and promote this new technology, to the point that [Wasm itself has now its own section in the CNCF software landscape](https://landscape.cncf.io/?group=wasm). Wasm is now for all intents and purposes considered one of the CNCF official technologies for “Cloud-Native development”: this extends well-beyond the browser!

But how does this affect you, as a Java developer? Similarly to the other years, in this blog post, I will explore two main topics:

- **front-end development**, compiling a JVM language into Wasm
- **back-end development** running Wasm on top of a JVM

## The Front-End

Wasm was always meant to make the Web platform more efficient. That’s one reason why browsers and state-of-the-art JavaScript runtimes such as V8, JavaScriptCore and Spidermonkey, will probably *always* be at the forefront, when it comes to implementing practical, but bleeding-edge features. After all, for better or worse, that’s how Web development goes.

This is also why often these runtimes **adopt experimental features of the Wasm spec**, way before other, smaller runtimes. This is expected, considering also the workforce that is often beyond major web browsers.

[Last year](https://www.javaadvent.com/2023/12/a-return-to-webassembly-for-the-java-geek.html) we explained how Wasm behaves more like a native target, than a JVM. This is, to some extent, still true: most WebAssembly runtimes do not support much beyond the so-called “linear memory”, threading support is still hit and miss, and the exception handling spec was revised a few times, finally hitting the last stage at the end of last year.

However, today all major browsers now support garbage-collected references, exception handling, and, through some shims, even threaded execution. All of these features are particularly important when it comes to supporting higher-level, managed, garbage-collected languages such as those hosted on the Java platform.

It is indeed feasible to support these features even in a native-like compilation target: after the GraalVM Native Image builder does just that. And Go *is* a higher-level, managed, garbage-collected, multi-threaded language and both TinyGo and “Big” Go can now be compiled to Wasm that runs perfectly in self-contained Wasm runtimes such as [Wasmtime](https://wasmtime.dev), [wazero](https://wazero.io) or [chicory](https://chicory.dev).

However, this comes with the cost of inflating the resulting binary, with baggage such as a garbage collector, emulating exception handlers, emulating multi-threading.

I believe this situation will continue for a while; that is, smaller runtimes, especially if they want to keep their footprint small, and especially if their *development team* is small, will generally be very conservative when it comes to implementing new, experimental part of the specs.

But it’s good that browsers are the front-runners: they have better resourcing, and hence can afford to support these experiments earlier. And besides, it makes sense for the Web to try to keep binary sizes down. For other use cases, we will see, this is less of a concern.

In the following I will show 3 compiler toolchains that generate Wasm, but still mainly target the browser.

Feeling [fatigued](https://medium.com/@ericclemmons/javascript-fatigue-48d4011b6fc4#.prcj59904) already? We’re just getting started!

### Kotlin

<div style="width:375px;float:left">

![](https://lh4.googleusercontent.com/proxy/Dy9akvPnvjj7g9HZX2HabbfJZU5VyCuQkHAtnutrZOIpCmyNJ8Zj6X8DELrxBwGxz-TpU_150Gjk6ax2wUKXPf0LLII4CQwXnDY)

Pic related: A Kotlin jar

</div>


Kotlin was one of the first JVM languages that targeted the browser with their JavaScript backend. Over the years, they gained a native compiler, and also, starting off as a flavor of their native compiler, a Wasm compiler.

The Kotlin team has recently switched their Wasm compiler backend to WasmGC and the Exception Handling proposal, resulting in smaller file sizes. However, this required a major overhaul of the compiler architecture, because a Wasm binary that targets the linear memory [behaves more like a native target](https://www.javaadvent.com/2023/12/a-return-to-webassembly-for-the-java-geek.html), while targeting WasmGC is a bit closer to how JVM class files are represented.

Luckily, with their familiarity with targeting both the JVM and JavaScript, even though it was no small feat, we now have a functional Kotlin compiler for Wasm that works for [on all runtimes supporting the GC and Exception Handling specs](https://kotlinlang.org/docs/wasm-troubleshooting.html#exception-handling-proposal).

To get started, you can head over to [this Kotlin Wasm template](https://github.com/Kotlin/kotlin-wasm-browser-template).

I believe that the Kotlin toolchain is currently the most impressive and advanced when it comes to practical use cases for JVM language: the teams at Jetbrains have already ported [Compose Multiplatform](https://www.jetbrains.com/compose-multiplatform/) to it. This that, you can already today port complex applications that will run both on mobile platforms and in the browser, with the convenience of using one modern, familiar programming language!

Kudos to the team!

### Scala

Scala was a pioneer in the JVM language ecosystem for many reasons; it helped revive the attention to the Java platform with a new, concise and practical programming language, in a time where the Java language was still very conservative.

It is also one of the first JVM languages to gain [support for a JavaScript compiler](https://www.scala-js.org/).

End users have been asking for a Wasm backend for a while. But the JavaScript backend was pretty practical and battle-tested; so the team deferred this sizable effort until there was reasonable support for some key features, such as, for instance (again!) garbage collected objects.

In the meantime, the alternative [Scala Native backend initially experimented with it](https://github.com/shadaj/scala-native-wasm). Now, the Scala.js team has decided to finally take on this effort, as the feature gap in browsers is closing more and more.

As documented on the [Scala.js website](https://www.scala-js.org/doc/project/webassembly.html) the Wasm backend is still in its infancy, and it comes with some experimental requirements (including garbage collected references, and exception handling support). It is also primarily targeting browsers, and as such, it still emits some chunks of JavaScript.

For some examples, check out [keynmol/scalajs-wasm-game-of-life](https://github.com/keynmol/scalajs-wasm-game-of-life) and [https://github.com/sjrd/funlabyrinthe-scala](https://github.com/sjrd/funlabyrinthe-scala).

### Java

I mentioned [TeaVM](https://www.teavm.org/) multiple times in the past. TeaVM has recently [switched their backend to WasmGC](https://github.com/konsoletyper/teavm/discussions/963).

The [J2CL](https://github.com/google/j2cl) project is the successor to GWT. This project is extensively used at Google, to the point that you are probably already using it even if you don’t realize it. For instance, [Google Sheets](https://web.dev/case-studies/google-sheets-wasmgc) is using Wasm. Check out [WebAssembly at Google by Thomas Steiner &amp; Thomas Nattestadt](https://www.youtube.com/watch?v=2En8cj6xlv4). And since you are there, you might want to check out Thomas Steiner’s podcast, [WasmAssembly](https://youtu.be/gJPYvv1flJY?list=PLNYkxOF6rcIA46I-YCX3ASF4SRb548z8s) there is also an episode interviewing [our very own Steve](https://youtu.be/gJPYvv1flJY?list=PLNYkxOF6rcIA46I-YCX3ASF4SRb548z8s), talking about all things [Extism](https://extism.org), including [Chicory](https://chicory.dev)!

Read more on [how to use J2CL with Wasm here](https://github.com/google/j2cl/blob/master/docs/getting-started-j2wasm.md).

### GraalVM Native Image

All the languages we have mentioned earlier, have something in common: they all generate Wasm code starting from source code. This means that the Kotlin compiler compiles Kotlin code, the Scala compiler compiles Scala code, and the TeaVM and J2CL compilers compile Java code. “Well, duh”, I hear you say.

<div style="width:375px;float:right">

[![](https://i0.wp.com/www.javaadvent.com/content/uploads/2024/12/graalvm-mascot-coffeecup.png?resize=500%2C500&ssl=1)](https://i0.wp.com/www.javaadvent.com/content/uploads/2024/12/graalvm-mascot-coffeecup.png?ssl=1)

The brand new GraalVM Mascot
</div>

On the other hand, GraalVM native image builds a binary starting from class files, making it possible to generate binaries for *any* language that typically runs on a JVM. So, considering the history of Wasm compilers so far, it would be really cool if they also started to build upon the experience they made with the Native Image builder, and implement a new backend for WebAssembly.

Oh boy, do I have news for you! It turns out that the GraalVM team *is* in fact working on a Wasm backend that will target browsers and state-of-the-art JavaScript runtimes such as V8 (at least at the beginning).

The work is being [actively tracked on GitHub](https://github.com/oracle/graal/issues/3391). I had the chance to take a look at an early demo, where a fully functional `javac` ran in the browser. The compiler is targeting WASI too, so, you should also be able to run it with Node, and, when more will catch up, with all the other self-contained Wasm runtimes.

There is some work to do but the direction is extremely exciting!

### Honorable Mentions

I am always fascinated with the excellent work at [LeaningTech](https://leaningtech.com/), so, even though it’s kind of a different stack, I want to mention the herculean work that this team is doing in porting code to the browser. Besides their **very recent relaunch of their x86 in-browser VM**, this time including a GUI, they also provide commercial support to compiling C++ and Flash to Wasm.

I believe one of the crown jewels is their Java runtime stack called [CheerpJ](https://leaningtech.com/what-is-cheerpj/), which they *also* recently relaunched. It is a [sophisticated port of a proper OpenJDK to Wasm that JIT-compiles bytecode into Wasm for optimal performance](https://cheerpj.com/how-cheerpj-works/), and contained code size. They can also run [Java Applets](https://cheerpj.com/cheerpj-applet-runner/) and [JNLP](https://cheerpj.com/cheerpj-jnlp-runner/) without a Java runtime installed on the desktop. That’s pretty neat!

## Backend

Wasm in the browsers is definitely here to stay, but the thing about this ecosystem that interests me the most is Wasm workloads in the backend. Wasm is on the [ThoughtWorks 2024 Tech Radar](https://www.thoughtworks.com/content/dam/thoughtworks/documents/radar/2024/10/tr_technology_radar_vol_31_en.pdf)

This year a lot has happened there too. But how does it impact JVM developers?

Last year, a surprising [Christmas gift for all Java developers was the first global official announcement of the Chicory interpreter](https://www.javaadvent.com/2023/12/chicory-wasm-jvm.html). While at the time the project was in an early development stage, it was already quite capable! Last October [we announced our first milestone release](https://chicory.dev/blog/chicory-1.0.0-M1), including, next to the interpreter, a new experimental bytecode translator (the “AoT compiler”), that turns Wasm bytecode into JVM Wasm for improved performance. Check it out and let us all know!

The GraalVM team also [recently announced](https://medium.com/graalvm/whats-new-in-graal-languages-24-1-b2452c9debae) that their [Wasm support can be now considered stable](https://www.graalvm.org/webassembly/) and ready for production, with the benefit of all their past and present experience with developing languages for the JVM.

This means that it’s a great time to get started with Wasm support on the JVM. But what are the best use cases you can cover with a Wasm runtime that runs on a JVM? Isn’t this redundant? Why would you want to load a foreign bytecode format (Wasm) on a bytecode language runtime (i.e., your beloved JVM)?

There are many use cases, but I want to start from possibly the most niche and counterintuitive, because it’s cursed, and it feels wrong, and yet sometimes it makes so much sense.

### Native Libraries Without JNI

<div style="text-align:center; font-style: italic">

[![A Panama Hat](https://i0.wp.com/www.javaadvent.com/content/uploads/2024/12/hat-1062061_640.jpg?resize=600%2C450&ssl=1)](https://i0.wp.com/www.javaadvent.com/content/uploads/2024/12/hat-1062061_640.jpg?ssl=1)

Did you know that “JNI” is supposed to be read as “Genie”? No? That’s because I just made that up (image by [Natasha G](https://pixabay.com/users/natashag-363234/?utm_source=link-attribution&utm_medium=referral&utm_campaign=image&utm_content=1062061) from [Pixabay](https://pixabay.com//?utm_source=link-attribution&utm_medium=referral&utm_campaign=image&utm_content=1062061))

</div>

“LOL look at this one,”JNI”: we’ve got [**Panama**](https://openjdk.org/projects/panama/), now, you boomer!” Right. Of course, **Panama** will finally bring a saner developer experience to interfacing with a native library.

However, you still have to deal with all of the limitations of interfacing with native code; essentially,

- the garbage collector and the threads (virtual and native!) may not play along well with it;
- plus, a memory corruption bug in your native code could tear down your entire JVM.
- finally, your build is now tied to the specific CPU architectures that your native library provides support for (this may not depend on you!)

Now, it would be most excellent if you could just bring a native library to JVM land, and be done with it. Some projects attempted this before:

- [NestedVM](http://nestedvm.ibex.org/) was a wonderfully cursed project that translated binaries for the MIPS architecture to Java bytecode. Unfortunately it hasn’t released a new version since 2009.
- [GraalVM’s Sulong](https://github.com/oracle/graal/tree/master/sulong) is a high-performance LLVM bitcode runtime; i.e. it evaluates LLVM’s intermediate binary representation. One downside is that LLVM bitcode is [known to be unstable](https://dl.acm.org/doi/10.1145/3062341.3062363).

The key difference is that, this time

1. 1a lot of compilers are including first-party support for Wasm, so it’s easier to cross-compile to this target and check whether it’s working.
2. the Web itself is huge, and a lot of developers are interested in porting well-written, performance-conscious libraries to use in their Web application

As a consequence, it’s becoming easier to find a port of a traditionally “native” library to Wasm, and if that port exists, it is ([in my limited experience with ancient code](https://tetrate.io/blog/introducing-wazero-from-tetrate/#:~:text=Or%2C%20maybe%20you%20would%20rather%20play%20the%201977%20Infocom%20text%20adventure%20Zork.)) safe to assume that the binary will run.

So how would this help you? Granted, this is a niche use case, but there are situations where the one library you need is written in C or Rust, and you just want to be able to use that bit of functionality; maybe you just want to bootstrap your project (then in the future you will rewrite it), or maybe it’s just enough for your use case.

<div style="width:210px;float:right">

[![JRuby Logo](https://i0.wp.com/www.javaadvent.com/content/uploads/2024/12/jruby.png?resize=200%2C200&ssl=1)](https://i0.wp.com/www.javaadvent.com/content/uploads/2024/12/jruby.png?ssl=1)

</div>

One example is JRuby’s team [port of the Prism Ruby parser](https://blog.jruby.org/2024/02/jruby-prism-parser#web-assembly-chicory), where they successfully used Chicory as a fallback on platforms where the native binary may not be available.

There are many other examples in the Go space, because the wazero project is much more mature than Chicory, and the Go ecosystem has already employed it successfully in many cases: in fact the Go runtime has a surprising number of similarities when it comes to limitations with interfacing with native code. My favorite must be [Xe Iaso’s “Carcinization of Go Programs”](https://xeiaso.net/blog/carcinization-golang/), because everyone should run a Rust library from a Go program to parse Mastodon’s toots. But there is also the large collection of [wasilibs](https://github.com/wasilibs), native tools re-built on top of wazero (including things like several plugins for **protoc**).

Another great example is [Nuno Cruces’ Go SQLite Wasm port](https://github.com/ncruces/go-sqlite3), built on top of wazero, [is competitive in many ways](https://github.com/cvilsmeier/go-sqlite-bench) with other alternatives. Indeed, there is also an experimental SQLite port of SQLite to Chicory called [sqlite-zero](https://github.com/dylibso/sqlite-zero).

The GraalVM team has also shared on their brand-new [GraalWasm landing page](https://www.graalvm.org/latest/reference-manual/wasm/) [an example of embedding C code running as a Wasm binary](https://github.com/graalvm/graal-languages-demos/tree/main/graalwasm/graalwasm-embed-c-code-guide/).

And, of course, you can run Doom in Swing using both [GraalWasm](https://github.com/stepstone-tech/doom-graalvm) and [Chicory](https://github.com/andreaTP/doom-chicory).

### Bringing Computations In-Core

Wasm has caught the attention of many as a *compact alternative to containers*.

Indeed, serverless workloads are an excellent candidate for Wasm: you get a cross-platform, efficient, executable binary representation that can be compiled into fast native code. And, of course, the grumpy Java developer in you might mumble something about application servers, but hopefully [we have already addressed](https://www.javaadvent.com/2022/12/webassembly-for-the-java-geek.html) that Wasm is not just “exactly the same” as Java Bytecode. Companies such as [Fermyon](https://fermyon.com) and [Cosmonic](https://cosmonic.com) are building software platforms of this kind. CDN companies such as [Fastly](https://fastly.com) or [Cloudflare](https://cloudflare.com) are putting Wasm-computations at the edge.

But besides serverless containers, there is another space where cloud deployments could be replaced by Wasm “functions”. This is when a service plays the role of a logic “extension” to another service.

In this sense, such a service, be it a *sidecar*, be it a *Webhook* (or whatever fancy words you kids call them these days) can be often seen as a glorified “plug-in”.

[Shopify with their “functions”](https://shopify.dev/docs/apps/build/functions) were among the first mainstream platforms to adopt Wasm to that end.

There are a few reasons why Wasm is a good candidate for plugin embedding. First of all, it was *explicitly designed* to work in concert with JavaScript, which, nowadays, is the “extension language” for excellence! It was designed with the necessities of the web platform in mind: namely, security and sandboxing; in fact, it cannot access any feature of the host language without being explicitly given access to them (through what are called “host functions”). And obviously being cross-platform, efficient and compact.

<div style="width:320px;float:right">

[![Extism logo](https://i0.wp.com/www.javaadvent.com/content/uploads/2024/12/extism.png?resize=300%2C320&ssl=1)](https://i0.wp.com/www.javaadvent.com/content/uploads/2024/12/extism.png?ssl=1)

</div>

[Dylibso](https://dylibso.org) believes that all software should be “squishy”: that is, malleable, adaptable, and extensible; so they developed [Extism](https://extism.org), an open-source framework to build your own “function-as-a-service” layer *inside* your application. Or, to put it more simply, to easily embed your own plug-in system. At the time of writing Extism supports 16 “host” runtime platforms and over 10 “guest” languages to write your plugin (and counting!). Notably, for the Go host, Extism uses [wazero](https://wazero.io), and for the Java platform, we are finalizing our [Chicory SDK](https://github.com/extism/chicory-sdk) (Java is [already supported through a native extension](https://github.com/extism/java-sdk) though).


<div style="width:220px;float:left">

[![XTP Logo](https://i0.wp.com/www.javaadvent.com/content/uploads/2024/12/xtp-logo.png?resize=200%2C150&ssl=1)](https://i0.wp.com/www.javaadvent.com/content/uploads/2024/12/xtp-logo.png?ssl=1)

</div>

Dylibso has also recently released an integrated platform for plugin delivery and management called [XTP](https://getxtp.com), building on top of Extism and OpenAPI. Essentially, if you have an API manifest for your service, you should feel at home bringing it to Extism. The codegen and testing toolset is all available free of charge. For instance, we have an example of [how to port the API for the Twenty CRM and scaffold your own plug-in system](https://www.getxtp.com/blog/adding-extensibility-to-twenty-a-modern-crm) and another example of how to write a [programmable Discord bot](https://www.getxtp.com/blog/extending-discord-with-wasm).

What makes Wasm interesting as a JDK extension language is that it supports multiple languages, and the guest can be preemptively terminated, **preventing it from hogging the CPU**. Moreover, user-provided Wasm code can be safely loaded and unloaded in a controlled environment, making it an excellent choice for hosting [user-defined functions in a database](https://dylibso.com/blog/pg-extism/).

The GraalVM team has also released a set of [compelling use cases for GraalWasm](https://www.graalvm.org/webassembly/#:~:text=GraalWasm%20Quick%20Start), including integrations with [Micronaut](https://github.com/graalvm/graal-languages-demos/tree/main/graalwasm/graalwasm-starter) and [Spring Boot](https://github.com/graalvm/graal-languages-demos/tree/main/graalwasm/graalwasm-spring-boot-photon)!

#### More Complex Use Cases

I have recently written a few use cases for Extism, Chicory and XTP. Both the following examples support compilation into a native binary, while retaining dynamic code loading capabilities.

##### Kafka Data Transforms

I have recently written an article on how to plug Chicory, Extism and XTP inside a **native Quarkus application** to build a Kafka data transform service. In this article I am giving a detailed explanation of how to use Extism, the XTP codegen tooling, and the XTP service to manage a data transform pipeline. [Sounds intriguing? Check it out](https://www.getxtp.com/blog/pluggable-stream-processing-with-xtp-and-kafka)

[![](https://i0.wp.com/cdn.loom.com/sessions/thumbnails/80dca9fa50bd46688ebca87b0fd58f6f-537f741734826116-full-play.gif?w=600&ssl=1)](https://www.getxtp.com/blog/pluggable-stream-processing-with-xtp-and-kafka)

There is also a follow-up in the pipeline, where I’ll explore **extending the Kafka broker** to the same effect. Stay tuned!

##### Quarkus Wasm Extension Demo

In the [Quarkus Wasm Extension](https://github.com/evacchi/extism-quarkus-greeting-app) demo, I have implemented a middleware-style HTTP filter that you would usually run in a proxy or an API gateway like Envoy or NGINX using some API like [Proxy-Wasm](https://github.com/proxy-wasm). Through the Quarkus extension and Chicory, you can compile the executable to a native image and load and reload extensions dynamically. [![](https://i0.wp.com/www.javaadvent.com/content/uploads/2024/11/request-proxy.png?resize=600%2C390&ssl=1)](https://i0.wp.com/www.javaadvent.com/content/uploads/2024/11/request-proxy.png?ssl=1)

Essentially, a proxy such as Envoy implements a chain of HTTP request filters, it intercepts your requests, and then passes them through the filter. However, this requires you to deploy a fleet of proxies as sidecars alongside your application. On the one hand, makes it easier to manage policies from a central control plane; on the other hand, it might raise operational costs, because now you have even *more services* to deal with! Instead, you can turn some of these policies into interceptors inside your Quarkus application!

[![](https://i0.wp.com/www.javaadvent.com/content/uploads/2024/11/request-embedded-1024x720.png?resize=600%2C422&ssl=1)](https://i0.wp.com/www.javaadvent.com/content/uploads/2024/11/request-embedded.png?ssl=1)

By cutting on the middleman, you are turning a complex multi-container deployment into a single application deployment! Pair that with XTP, and then you also get the missing control plane for your plugin policies!

### Native DYNAMIC Software Extensions

Did I just mention **dynamic code loading** with Native Image? Indeed, I just did. One of the original limitations for an executable built into a native image, was the infamous “closed world assumption”, essentially, you have to instruct the native image builder about all the classes you expect to need at run-time, especially if you load them dynamically through run-time code reflection.

Limitations with reflections are however nowadays largely solved, because a lot more tooling is available to automatically inspect your record and report it to the compiler; there are [embedded JSON manifests as well as automatic detection mechanisms](https://www.graalvm.org/jdk21/reference-manual/native-image/dynamic-features/Reflection/); besides frameworks like [Quarkus](https://quarkus.io) provide also sophisticated code-generation mechanisms that extend GraalVM’s built-in routines.

There have been also ways to dynamically load code: you can [link against native libraries](https://www.graalvm.org/latest/reference-manual/native-image/dynamic-features/JNI/) or you can use the [Espresso Truffle runtime](https://www.graalvm.org/latest/reference-manual/espresso/).

[Espresso is particularly interesting because it’s another example of a VM-in-a-VM:](https://www.graalvm.org/latest/reference-manual/espresso/) it’s literally an implementation of the Java VM Specification that runs *on top* of another JVM. While, at a very first glance, this could feel like an intellectual experiment (you could probably run an Espresso instance on top of another Espresso instance, and then… it’s Espresso all the way down), it makes a lot of sense if you build one inside a native image! Then you can have your fast-to-boot, tiny, efficient, self-contained executable, and yet be able to load Java code in a sandboxed environment! You can literally have your cake and eat it too.

Now, why would you want to use Wasm instead? Well, because, why limit yourself to JVM languages? A Wasm runtime will allow you to dynamically load *any* Wasm code from your users!

If you are interested in this use case, read the details in the [Extism+XTP Kafka demo](https://www.getxtp.com/blog/pluggable-stream-processing-with-xtp-and-kafka), and the [Quarkus Wasm Extension Demo](https://github.com/evacchi/extism-quarkus-greeting-app).

## Conclusions

It’s been a wonderful year for Wasm in the Java space. I hope this article sparked your interest too!

If you want to learn more about [Extism](https://extism.org) and [XTP](https://getxtp.com), you can join us on our [Discord](https://discord.gg/RWUN5SytPs?event=1303763128370200666), where we host [weekly office hours on Wednesdays](https://discord.gg/RWUN5SytPs?event=1303763128370200666): ask all your questions about Wasm, Extism, XTP.

If you are especially interested in [Chicory](https://chicory.dev), you can also join our [Zulip](https://chicory.zulipchat.com/join/g4gqsxoys6orfxlrk6hn4cyp/); and if you are the odd Gopher reading a Java blog, you might want to join the [#wazero channel too](https://gophers.slack.com/archives/C040AKTNTE0).

Finally, if [Extism](https://extism.org) and [XTP](https://getxtp.com) sparked your interest and you’d like to have some fun with extending [Claude.ai](https://claude.ai/) (and other LLMs supporting the [Model Context Protocol](https://modelcontextprotocol.io/docs/)) you might want to check out our brand-new project: [**mcp.run**](https://www.mcp.run/).

 
