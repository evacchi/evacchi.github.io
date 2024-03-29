---
tags: Wasm Java History
date: "2022-12-23T00:00:00Z"
tags: 
  - wasm
  - java
title: WebAssembly for the Java Geek
---

<div style="padding: 1em; background: #eee">
<b>Update (Aug 3, 2023)</b>. This post was originally published at <a href="https://www.javaadvent.com/2022/12/webassembly-for-the-java-geek.html">JVM Advent</a>. If you want to read a more comprehensive guide to language VMs, I strongly recommend <a href="https://www.neversaw.us/2023/05/10/understanding-wasm/part1/virtualization/">Chris Dickinson's series</a> and especially <a href="https://www.neversaw.us/2023/06/30/understanding-wasm/part2/whence-wasm/">part 2</a>. If you want to see a recorded, more recent version of this content, <a href="https://www.youtube.com/watch?v=PRL05TZtxpM">check out my JNation talk</a>.
</div>


When many Java developers hear the word WebAssembly, the first thing they think is “browser technology”. The second thing: “it’s the JVM all over again”. After all, for a Java developer, in-browser apps are <strong>prehistory</strong>.
<div style="float: left; margin-right: 1em;"><img style="width: 200px;" src="/assets/wasm-java/hipster-kitty.png" /></div>
In the last few weeks, there have been quite a few announcements around WebAssembly, such as the <a href="https://www.docker.com/blog/docker-wasm-technical-preview/">Docker+Wasm Technical Preview</a>. As a Java geek myself, I think we should not dismiss this technology as just a fad.

Indeed, WebAssembly <strong><em>is</em></strong> “a bytecode for the Web” (I mean, that’s the name after all), but the similarities between Java and <strong>Wasm</strong> (lower-cased: it’s a contraction, not an acronym!) really end here.

If you want to know more about how we came to define the WebAssembly standard, <a href="https://evacchi.github.io/wasm/compilers/history/2022/11/23/a-history-of-webassembly.html">you can learn more about its history on my own blog</a>. In the following, I will try to argue that <strong>there is more to WebAssembly than “just the web”.</strong>

First of all, a WebAssembly runtime is only shallowly similar to a JVM. For instance, WebAssembly was always meant to be <em>a proper compilation target</em> for different programming languages, while the JVM was not, at least, not originally.
<h2 id="myth-1-the-jvm-is-a-polyglot-compilation-target">Myth #1: The JVM Is A Polyglot Compilation Target</h2>
Of course, everyone knows the JVM is one of richest, interoperable language ecosystems there is. We don’t have just Java, we also have Scala, Jython, JRuby, Clojure, Groovy, Kotlin and many many others.

<img style="float: right; width: 100px; margin-left: .5em;" src="https://i.imgur.com/LTOaClW.png" />

However, the sad, sad reality is that <strong>Java bytecode was never really meant to be a general-purpose compilation target</strong>. In fact, you can even find <a href="https://dl.acm.org/doi/10.1145/1711506.1711508">literary references</a> that spell that out clearly; in <a href="https://dl.acm.org/doi/10.1145/1711506.1711508">“Bytecodes meet combinators: <code>invokedynamic</code>  on the JVM”, John Rose</a> writes (bold mine):
<blockquote>The Java Virtual Machine (JVM) has been widely adopted in part because of its classfile format, which is portable, compact, modular, verifiable, and reasonably easy to work with. <strong>However, it was designed for just one language—Java—</strong> and so when it is used to express programs in other source languages, there are often “pain points” which retard both development and execution.</blockquote>
The paper describes how and why the <code>invokedynamic</code> opcode was introduced in the JVM; in fact, it was specifically introduced to support dynamic languages targeting the JVM as a runtime. At the time, those were many: JRuby, Jython, Groovy, etc… This opcode was not added <em>because</em> the JVM was <em>supposed</em> to support such languages; but because people were doing it anyway: so, it was better just to acknowledge it!

In other words, the JVM, as it was at the time, was <em>not</em> an adequate compilation target for dynamic languages. We may even argue that the JVM became a compiler target not because it was the <strong>best</strong> compilation target, but because people wanted to interoperate with it because of adoption and support …just like JavaScript!
<h2 id="graalvm-one-vm-to-rule-them-all">GraalVM: One VM to Rule Them All</h2>
The <a href="https://www.graalvm.org/">GraalVM</a> project has recently gone mainstream. This project includes a Just-in-Time compiler targeting regular Java bytecode, an API to build efficient language interpreters, and, recently, a native image compiler.

One of the original goals for GraalVM was to be <a href="https://dl.acm.org/doi/10.1145/2509578.2509581">“One VM to rule them all”</a>, i.e. to be a <em>polyglot runtime</em>.

But Truffle does not define a polyglot <em>compilation target</em>. Instead, the Truffle API allows you to <em>build</em> an efficient, JITting <strong>interpreter</strong> for <em>dynamic programming languages</em> using a very high-level representation (an AST-based interpreter, if you are interested).
<div style="font-size: 80%;">
<p style="padding-left: 40px;"><strong>Note for the nitpicker. </strong>Now, once you enter the programming-language-rabbit-hole everything gets kind of “meta”. Indeed, with Truffle you can write a JITting interpreter for some other “proper” bytecode format.</p>
<p style="padding-left: 40px;">In fact, there is a Truffle-based interpreter for LLVM (Sulong); and, sure, LLVM bitcode is <em>meant</em> to be a multi-platform/multi-target compilation target. So, by the transitive property, you may argue that GraalVM/Truffle do support a multi-platform compilation target.</p>
<p style="padding-left: 40px;">This is technically correct (<a href="https://www.youtube.com/watch?v=hou0lU8WMgo">which is the <em>best</em> kind of correct</a>), but there are many considerations to be made, and there is not enough space here to discuss them all. In short, LLVM bitcode is meant to be a compilation target, but it was not necessarily meant to be a cross-platform <em>runtime language</em> (e.g., there are slight variations in the instructions you may have to use, depending on the CPU/OS you want to target). Moreover, as opposed to WebAssembly, which is a multi-vendor standard, GraalVM and Truffle are, to this day, open source, community-driven, but single-implementation efforts (<a href="https://www.graalvm.org/2022/openjdk-announcement/">work has recently started to bring it to the OpenJDK and possibly to the Java Language Specification</a>).</p>
<p style="padding-left: 40px;">Ultimately, WebAssembly is also another language that GraalVM/Truffle is able to support, so if you want to use GraalVM, you might even target Wasm!</p>

</div>
<h2 id="myth-2-its-just-another-stack-based-language-vm">Myth #2: It’s Just Another Stack-based Language VM</h2>
WebAssembly is defined as a virtual instruction set architecture (ISA) for a <a href="https://github.com/WebAssembly/design/blob/main/Rationale.md"><em>structured</em> stack-based virtual machine</a>.

The word <em>structured</em> here is key, because it is a very significant departure from the way, say, the JVM works. In practice, in a structured stack machine most computations use a stack of values, but control flow is expressed in structured constructs such as blocks, ifs, and loops. Moreover, in the WebAssembly language, some instructions can be represented both as “simple” and as “nested”.

Let’s see an example. In the stack-based Wasm machine the expression:
<pre>( x + 2 ) * 3</pre>
<div id="cb2" class="sourceCode">
<pre class="sourceCode asm"> int exp(int);
    Code:
       0: iload_1
       1: iconst_2
       2: iadd
       3: iconst_3
       4: imul
       5: ireturn</pre>
</div>
Could be translated in the following sequence of instructions:
<div id="cb3" class="sourceCode">
<pre class="sourceCode scheme">(local.get $x) 
(i32.const 2) 
i32.add 
(i32.const 3) 
i32.mul</pre>
</div>
<ul>
 	<li><code>local.get</code> puts the value of the local variable <code>$x</code> on the stack</li>
 	<li>then the <code>i32.const</code> pushes the 32-bit integer (<code>i32</code>) constant <code>2</code> on the stack</li>
 	<li><code>i32.add</code> pops the two values from the stack, and push the result <code>$x+2</code> on the stack</li>
 	<li>we then push the integer constant <code>3</code></li>
 	<li><code>i32.mul</code> pops the two integer values and pushes the <code>i32</code> result of the multiplication (<code>($x+2)*3</code>)</li>
</ul>
You may have noticed how instructions that take at least one argument are parenthesized. The one we just saw is the “linearized” version of WebAssembly. It is the one that is straightforwardly translated into its binary representation in a <code>.wasm</code> file. There is however another, semantically equivalent “nested” representation:
<div id="cb4" class="sourceCode">
<pre class="sourceCode scheme">(i32.mul 
  (i32.add 
    (local.get $x) 
      (i32.const 2)) 
    (i32.const 3))</pre>
</div>
The nested representation is particularly interesting because it shows a peculiar difference with other types of bytecodes (such the JVM’s), i.e. operations nest and read like operations in a more conventional programming language. Well, for some definition of conventional: it reads like Scheme (a language in the family of LISPs), and the convention for parenthesization is a clear homage to it. Of course, this is not by accident; if you know a bit about JavaScript’s evil origin story you’ll definitely know that it was originally written in 10 days; and you may also know that Brendan Eich initially was hired to develop a Scheme dialect.

However, the even more interesting detail (at least to me) is that the nested sequence naturally linearizes to the other version; in fact, if you follow the precedence rule for parenthesized expressions, you have to start at the innermost parentheses:
<div id="cb5" class="sourceCode">
<pre class="sourceCode scheme"> (i32.add 
    (local.get $x) 
    (i32.const 2))</pre>
</div>
so first you get <code>$x</code>, then you evaluate the constant to <code>2</code>, then you sum them; then you continue with the outermost expression:
<div id="cb6" class="sourceCode">
<pre class="sourceCode scheme">(i32.mul 
  (i32.add ...) 
  (i32.const 3))</pre>
</div>
Now you have evaluated the contained <code>i32.add</code>, you evaluate the constant <code>3</code> and you can multiply them. That’s exactly the same order of evaluation of the stack-based version!

We have also mentioned structured control flow. The reason for this choice is, again, safety; but also <a href="https://github.com/WebAssembly/design/blob/main/Rationale.md">simplicity</a>:
<blockquote>The WebAssembly stack machine is restricted to structured control flow and structured use of the stack. This greatly simplifies one-pass verification, avoiding a fixpoint computation like that of other stack machines such as the Java Virtual Machine (prior to stack maps). This also simplifies compilation and manipulation of WebAssembly code by other tools.</blockquote>
Let’s see an example:
<div id="cb7" class="sourceCode">
<pre class="sourceCode java"> void print(boolean x) {
    if (x) {
        System.out.println(1);
    } else {
        System.out.println(0);
    }
}</pre>
</div>
This translates to the bytecode:
<div id="cb8" class="sourceCode">
<pre class="sourceCode asm"> void print(boolean);
 Code:
 0: iload_1
 1: ifeq 14
 4: getstatic #7 // java/lang/System.out:Ljava/io/PrintStream;
 7: iconst_1
 8: invokevirtual #13 // java/io/PrintStream.println:(I)V
11: goto 21
14: getstatic #7 // java/lang/System.out:Ljava/io/PrintStream;
17: iconst_0
18: invokevirtual #13 // java/io/PrintStream.println:(I)V
21: return</pre>
</div>
You will notice the unstructured jump instructions <code>ifeq</code>  and <code>goto</code> which are missing from the equivalent WebAssembly definition, replaced instead by proper <code>if...then...else</code> blocks!
<div id="cb9" class="sourceCode">
<pre class="sourceCode scheme">(module
 ;; import the browser console object, 
 ;; you'll need to pass this in from JavaScript
 (import "console" "log" (func $log (param i32)))

 (func
   ;; change to positive number (true) 
   ;; if you want to run the if block
   (i32.const 0) 
   (call 0))

  (func (param i32)
   local.get 0 
   (if
     (then
       i32.const 1
       call $log ;; should log '1'
     )
     (else
       i32.const 0
       call $log ;; should log '0'
     )))

 (start 1) ;; run the first function automatically
)
</pre>
</div>
You can see and play with the original example on the <a href="https://developer.mozilla.org/en-US/docs/WebAssembly/Reference/Control_flow/if...else">Mozilla Developer Network</a>

Obviously, this also linearizes to a non-nested version:
<div id="cb10" class="sourceCode">
<pre class="sourceCode scheme">(module
  (type (;0;) (func (param i32)))
  (type (;1;) (func))
  (import "console" "log" (func (;0;) (type 0)))
  (func (;1;) (type 1)
    i32.const 1
    call 0)
  (func (;2;) (type 0) (param i32)
    local.get 0
    if  ;; label = @1
      i32.const 1
      call 0
    else
      i32.const 0
      call 0
    end)
  (start 1))
</pre>
</div>
<h2 id="more-differences-memory-management">More Differences: Memory Management</h2>
For better or worse, another area where WebAssembly virtual machines greatly differ from a JVM is <strong>memory management</strong>. As you probably know, Java languages do not require you to allocate and deallocate memory, or care about stack vs. heap allocations; at least in general: you may care about those and there are ways to deal with them explicitly if you really need to. But the reality is that most people won’t.

This is not a language-level feature, it is really also how the VM works. You do not have primitives to deal with memory at the VM-level; in fact, primitives for heap allocation are available, but they are exposed as JDK APIs. There is no way for you to opt out of managed memory: you cannot just say “I don’t care about the garbage collected heap, I am going to do my own memory management”.

At this time, WebAssembly is quite the opposite. It is no coincidence that <em>most</em> languages targeting WebAssembly today really manage their own memory. Some languages do garbage collection; but in those cases, they have to roll their own garbage collection routines, because the VM does not provide such a facility.

Instead, with WebAssembly you get a slice of <em>linear memory,</em> and then you can do whatever you want with it. Allocate, deallocate; even move it around if you’d like. While this is, in a way, more powerful than what the JVM provides, it also comes with caveats.

For instance, the JVM does not require you to specify the memory layout of an object, because it is up to the VM to deal with structure packing, word alignment, etc. In the case of WebAssembly, <em>you</em> deal with those issues.

On the one hand, this makes it perfect as a target for manually-managed programming languages, where a higher degree of control is expected and desired. On the other hand, it could make it harder for such languages to <em>interoperate</em> with each other.

Now, structure and object layout is an <strong>ABI</strong> concern: <a href="https://docs.oracle.com/javase/specs/jls/se7/html/jls-13.html">a thing of the past for JVM developers, except for some very limited and notable exceptions</a>.

Interestingly enough, the <a href="https://github.com/WebAssembly/gc">draft GC spec for WebAssembly has recently moved forwards</a>, and it does not just deal with garbage collection, but it effectively describes how to deal with <strong>structures</strong>, and how to make them interoperate, regardless of the originating language. So, while this is still not ready, things are continuously evolving and multiple concerns are being addressed.
<h2 id="more-than-web">More Than Web</h2>
<a href="https://www.youtube.com/watch?v=UrIiLvg58SY"><img style="float: right;" src="https://i.imgur.com/P6BMCD1.png" width="400" /></a>

Now, in all we have learned so far, you might have noticed that I never mentioned the word Web once.

Indeed, it took me a while to get to the point, but <strong>this is where I tell you, the Java Geek, why you should care</strong>.

Even if you do not care about front-end, you should not dismiss WebAssembly as a purely front-end technology. <strong>There is nothing in the design and specification of WebAssembly that makes it specifically tied to the front-end.</strong> In fact, most mainstream JavaScript runtimes are now able to load and link WebAssembly binaries, even outside the browser; so you can run a Wasm executable in a Node.js runtime, with a thin layer of JS glue code to interact with the rest of the platform.

But there are also many <strong>pure-WebAssembly runtimes</strong>, such as <a href="https://wasmtime.dev/">Wasmtime</a>, <a href="https://wasmedge.org/">WasmEdge</a>, <a href="https://wazero.io/">Wazero</a> that are completely untied from a JavaScript host. These runtimes are usually lighter-weight than a full-blown JavaScript engine, and they are often easy to embed inside larger projects.

In fact, <strong>many projects are starting to embrace WebAssembly as a polyglot platform to host extensions and plug-ins.</strong>

One notable example, for instance, is the<strong> <a href="https://www.envoyproxy.io/">Envoy proxy</a>:</strong> the codebase is mostly C++; it does support plug-ins, but with the same caveats as browser plug-ins: you have to compile them, you have to ship them, they may not run at the right level of privileges, they may even tear down the entire process in case of a fatal fault. Now, you could embed a Lua or a JS interpreter and let your users script their way to success: the interpreter is safer because it is isolated from your main business logic, and it only interacts in a safe way with the host environment; the main downside: you have to pick a language for your users.

Or, you could just embed a WebAssembly runtime, let your users pick their own language and <em>just compile it to Wasm</em>. You will have the same safety guarantees, and happier users.

These pure WebAssembly runtimes are not just for extensions. Many projects are creating thin layers of Wasm-native APIs to provide <strong>stand-alone platforms.</strong>

For instance, <a href="https://wasmcloud.com/">wasmCloud</a> is a distributed platform for writing portable business logic that can run anywhere from the edge to the cloud.

<a href="https://docs.fastly.com/products/compute-at-edge">Fastly</a> has developed a platform for serverless computing at the edge, where the serverless functions are implemented by user-provided WebAssembly executables.

<a href="https://www.fermyon.com/">Fermyon</a> is a startup that is developing a rich ecosystem of tooling and Web-based APIs to write Web apps using only Wasm. One of their latest announcement is their <a href="https://www.fermyon.com/cloud">Fermyon Cloud</a> offering.

These solutions offer custom, ad-hoc APIs for specific use cases; and this is indeed one way to use WebAssembly. But that is not the end of it. In 2019, Docker founder Solomon Hykes wrote:
<blockquote class="twitter-tweet">
<p dir="ltr" lang="en">If WASM+WASI existed in 2008, we wouldn't have needed to created Docker. That's how important it is. Webassembly on the server is the future of computing. A standardized system interface was the missing link. Let's hope WASI is up to the task! <a href="https://t.co/wnXQg4kwa4">https://t.co/wnXQg4kwa4</a></p>
— Solomon Hykes (@solomonstre) <a href="https://twitter.com/solomonstre/status/1111004913222324225?ref_src=twsrc%5Etfw">March 27, 2019</a></blockquote>
<a href="https://platform.twitter.com/widgets.js">https://platform.twitter.com/widgets.js</a>

If you pull this out of context your first question may be “What the hell has Wasm to do with Docker?” and, of course, “What the hell is WASI?”.

WASI is the <em>WebAssembly System Interface</em>. You can think of it as of a collection of (POSIX-like) APIs that allow a Wasm runtime to interact with the operating system. Is this like the JDK Class Library? Not quite. It is a thin layer of capability-oriented APIs for interaction with the operating system. You can read more on the <a href="https://hacks.mozilla.org/2019/03/standardizing-wasi-a-webassembly-system-interface/">Mozilla announcement blog.</a>, but, in short, this is the last piece of the puzzle: WASI allows to define backend applications that directly interact with the operating system without any extra layer, and without ad-hoc APIs. The current effort is to make WASI widely-adopted and, in a way, a standard <em>de facto</em> for backend development.

WASI APIs include things like file system access, networking and even threading APIs. These APIs work hand-in-hand with the lower-level capabilities of the runtime, enabling easier ports to the platform.
<h2 id="porting-java">Porting Java</h2>
With all its challenges, for the first time, we have a technology with the potential to become a truly multi-vendor, multi-platform, safe, polyglot programming platform. I believe that we, as Java geeks, should not lose the occasion to be relevant in this space.

The WebAssembly specification and the WASI effort are still in flux. But all these pieces together are paving the way to allow an easier port of any programming language, not just those with a manual memory management.

Indeed, some garbage collected languages are already available, albeit not all of them take the same approach. For instance, Go can be compiled to Wasm (albeit with some limitations). For instance, the Python port is a port of <em>the interpreter</em>. So they compiled the CPython interpreter to Wasm, and <em>then</em> that is used to evaluate Python scripts, just like in a traditional execution environment.

In fact, memory management is really just part of the story, and only one of the many caveats that at this time would allow to port Java. You can always stick a GC in your executable (indeed, this is how GraalVM Native Image currently work); in my opinion, however it is harder to support other CPU features or system calls that are currently still unstable or not widely supported.

For instance:
- threading support is still lacking or experimental in most stand-alone Wasm runtimes;
- even browser support is experimental, and simulated through WebWorkers.
- there is not a standardized support for socket access: all the services that allow you to write custom HTTP handlers usually provides you with a pre-configured socket, limiting low-level access
- Exception handling is another experimental feature that is harder to simulate, because of the lack of unstructured jumps in the Wasm bytecode: this will likely need proper support in Wasm VMs before it can be adopted.
- each language brings its own constraints on memory layout and object shapes: it is therefore harder for languages to share data across boundaries, hindering compatibility between different languages and thus limiting the suitability of Wasm as a polyglot plaform (this is however being addressed as part of the GC spec itself).

In short, there are many challenges to porting Java to the WebAssembly platform inside and outside the browser.
<h2 id="java-support-on-webassembly">Java Support on WebAssembly</h2>
Currently, multiple projects and library that deal with WebAssembly and Java. I have compiled a list of those that I found around the web. At this time, however, most of these are hobby projects.
<h3 id="running-java-in-the-browser">Running Java in the Browser</h3>
Many projects target Java translation to WebAssembly. Most of them, however, do not emit code that is compatible with leaner Wasm runtimes: in general, they are meant for running in the browser.
<ul>
 	<li><a href="https://github.com/mirkosertic/Bytecoder">Bytecoder</a>, <a href="https://github.com/i-net-software/JWebAssembly">JWebAssembly</a>, and <a href="https://teavm.org/">TeaVM</a> are all translators from Java bytecode into WebAssembly that take a slightly different approach to translating Java bytecode to browser-friendly code. Among the others, <a href="https://teavm.org/">TeaVM</a> seems the most promising, as shown in <a href="https://github.com/fermyon/teavm-wasi">Fermyon’s fork which includes initial support for WASI</a></li>
 	<li><a href="https://leaningtech.com/cheerpj/">CheerpJ</a> is a very promising, albeit proprietary, attempt to support the full extent of Java, including Swing. There is also a <a href="https://chrome.google.com/webstore/detail/cheerpj-applet-runner/bbmolahhldcbngedljfadjlognfaaein">Chrome extension to run good ol’ applets through Web tech</a></li>
</ul>
Here are also some honorable mentions of projects that target browser runtimes (with experimental Wasm support in some cases):
<ul>
 	<li><a href="https://github.com/google/j2cl/tree/master/samples/wasm/src/main/java/com/google/j2cl/samples/wasm">J2CL (successor to GWT)</a> is a source-to-source translator (i.e. a <em>transpiler</em>) from Java to JavaScript, which has recently gained support for Wasm. This compiler has also bleeding-edge support for the GC spec.</li>
 	<li><a href="https://github.com/jtulach/bck2brwsr">Bck2Brwsr</a> is another compiler from bytecode that targets JavaScript and the browser</li>
 	<li><a href="https://www.fermyon.com/wasm-languages/kotlin">Kotlin/Native</a> also supports being compiled to Wasm via LLVM. It comes with all the caveats of Kotlin/Native (e.g. it may not support all of your Java libraries)</li>
 	<li><a href="https://plasma-umass.org/doppio-demo/">DoppioJVM</a> is an interesting project that I wish to mention because it takes a completely different approach, <a href="https://www.fermyon.com/wasm-languages/python">similar to Python’s</a>: instead of compiling bytecode to Wasm, it is instead an in-browser VM (written in JavaScript) that is able to interpret JVM bytecode. Unfortunately, the project is currently unmaintained.</li>
</ul>
<h3 id="running-webassembly-on-the-jvm">Running WebAssembly on the JVM</h3>
We have been talking about running Java programs on a Wasm runtime. But of course, you may want to be able to do the opposite, too. In all fairness, the JVM already provides quite a few programming languages, and the current programming model that most Wasm runtimes offer (with manual memory management) seems kind of off when hosted on a JVM. But I still want to mention these for completeness, and because in general, they may still be interesting.
<ul>
 	<li>The prime candidate is obviously the aforementioned <a href="https://github.com/oracle/graal/tree/master/wasm">GraalVM’s Truffle implementation of a WebAssembly interpreter</a>, which benefits from all the JIT superpowers and polyglot interoperability of the GraalVM/Truffle platform</li>
 	<li><a href="https://github.com/cretz/asmble">asmble</a> is a suite of tools that includes a compiler from Wasm to bytecode and a Wasm interpreter</li>
 	<li><a href="https://github.com/fishjd/HappyNewMoonWithReport">Happy New Moon With Report (JVM)</a> is a WebAssembly runtime for the JVM (that I am including in this list because I just love the silly name!)</li>
 	<li>There are also bindings to native Wasm runtimes, such as <a href="https://github.com/kawamuray/wasmtime-java">kawamuray/wasmtime-java</a></li>
 	<li>The <a href="https://extism.org/">Extism project</a> has recently launched: it provides a unified API across different host languages to interface with a native WebAssembly runtime (Wasmtime)</li>
 	<li><a href="https://github.com/evacchi/kaitai-webassembly">Katai WebAssembly</a> is a Wasm parser written using the <a href="https://kaitai.io/">Katai Struct</a> binary parser generator that I am currently maintaining (PRs welcome!): this is not meant necessarily for <em>running</em> Wasm on the JVM, but it is actually useful when you want to be able to manipulate or query Wasm executables for information. In fact, a Kaitai grammar allows one to generate a binary parser for any supported language, so not just Java, but also Python, Ruby, Go, C++, and many others.</li>
</ul>
<h2 id="conclusion">Conclusion</h2>
I hope that this post sparked some interest in you. It is still early days for Java-on-Wasm, but I invite you to explore this brand-new world with an open mind: it may surprise you!