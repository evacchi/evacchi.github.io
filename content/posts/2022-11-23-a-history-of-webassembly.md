---
tags: Wasm Compilers History
date: "2022-11-23T00:00:00Z"
tags: 
  - wasm
  - compilers
title: A History of WebAssembly
aliases:
  - '/wasm/compilers/history/2022/11/23/a-history-of-webassembly.html'
---

<img src="https://i.imgur.com/JnJO7HZ.jpg" style="float:right" width=400/>

Lately there have been quite a few announcements around WebAssembly, such as the [Docker+Wasm Technical Preview][docker]. You may have started to wonder whether this technology is something you should care about.

In this blog post, we will lightheartedly explore the history of Wasm. I will not make any claim about correctness: I may have made mistakes; in that case, feel free to contact me! I will try to motivate how we came to defining the WebAssembly standard and VM, and how they are all about providing a multi-platform, portable low-level **compilation target** for multiple programming languages. In fact, a history of WebAssembly really is...

## A History of Running Arbitrary Code in the Browser

And indeed, this finds its root in some of the earliest days of the Internet going mainstream (let's call them the "Geocities days"). Browser had limited extensibility; but you could script your web page using a tiny cute language called JavaScript, that, as we all know, bear *only a superficial similarity to Java* (basically "it has braces" and similar control flow structures, including falling-through `switch` because why not). At the beginning, JavaScript was mostly meant to add a tidbit of interactivity to web pages. 

### The Early Days: Browser Plug-Ins

If you wanted full-blown multimedia capabilities you could use **plug-ins** instead. There were multimedia players such as Real Player, Windows Media Player, Quicktime; but also programmable platforms such as Flash, Shockwave, Java and also, at some point Silverlight. But developers could write their own shared libraries and link them against the browser runtime.

There were downsides: for instance, these libraries had to be shipped for multiple browsers, multiple operating systems and possibly multiple architectures. They had to be periodically updated, and, in general, they ran with the full-permission level of the browser, which in turn made them susceptible to security exploits.

### Rise of the Transpilers

<div style="float:right; margin-left: 1em"><img src="https://i.imgur.com/miQZNW0.png" style="float:right" width=300 /></div>


Plug-ins got the job done, but keeping them up-to-date and secure was a chore. In the meantime, browsers were becoming more and more powerful. For instance. While videos and music needed plug-in support for a long time, at some point browsers gained video and music playing capabilities. Around the same time, Web developers discovered that, even with all its quirks, JavaScript was a decent language after all. Google, with [Chrome and the V8 (2008)][v8] runtime, demonstrated that you could achieve quite a reasonable amount of performance using a well-engineered JIT compiler; and at some point, with Node.js [it even escaped the browser (2009)][nodejs].
 
However, for a long time the JavaScript language could not evolve. In order to support the largest number of browsers and not to "break the web", even if some browsers could support different or better programming languages (e.g. Google initially tried this with [Dart (2011)](https://dart.dev/)) people had to target the lowest common denominator.

This is probably one of the reasons why there was a sudden spike in so-called "Transpilers". If you know me, you will now that I am not in love with the term, because **Transpilers are just Compilers-In-Disguise**. Regardless, it just meant that most of these source-to-source translators (better name, but indeed a bit long) targeted JavaScript. In other words, you could write whitespace-delimited [Coffeescript (2009)](https://coffeescript.org/) (the first of such transpilers to go mainstream) and desugar it to well-formatted JavaScript, minus the pain. 

<div style="float:left; margin-right: 1em"><img src="https://i.imgur.com/Qo4sPje.png" width="300" /></div>

Around the same time, many other transpilers were released (including even pre-processors for CSS and HTML). Even people that did not really know JavaScript could write Web applications using one of these "compile-to-JavaScript" languages. Even the Java scene got excited for a while about the [Google Web Toolkit (GWT, 2006)](https://www.gwtproject.org/), a Java compiler framework that promised to handle the client/server split in an automated way (an evolution of GWT still exist in the [J2CL compiler](https://github.com/google/j2cl)). And I am not even mentioning the number of tools to transpile CSS and, heck, even HTML.

As a result, front-end developers grew accustomed to expect a build step in their regular workflow; even if you just wanted to use JavaScript! In fact, the [Babel.js project (2014)](https://babeljs.io/) allowed (and still allows) to use features in recent releases of the language, desugaring them into syntax for the older releases. And, since you are already waiting for a build stage, you might as well carry some static analysis there. And then [TypeScript (2012)](https://www.typescriptlang.org/) was born. But that's another story.

### A Compilation Target

In the meantime, Google was presenting to the world its sandboxing technology for the browser, called [Native Client (or NaCl (2011)](https://developer.chrome.com/docs/native-client/overview/)). While similar in principle to plug-ins (you had to ship executable code for different architectures), the difference was that Native Client executables were not unrestricted like traditional plugins, but ran in a sandboxed environment; morever, now a web-page could carry its own specific binary payload; while, traditionally plug-ins had to be installed separately. Finally, with Portable Native Client (PNaCl), you could also target an abstract CPU architecture, and the browser would "JIT it" to the actual architecture. The NaCl compiler toolchain was essentially a customized C/C++ toolchain. The main downside was that (P)NaCl was still a Chrome-specific technology.

At this point some people also realized that, since JS was effectively being treated as a compilation target, nothing would prevent us from "transpiling" languages that traditionally targeted bare-metal CPU architectures (such as C/C++) into JavaScript. — Incidentally, this is also where the traditional notion of a "transpiler" falls apart (at least to me): if a Transpiler is «a translator between languages "at the same level of abstraction"», is a translator from C to JS really between languages "at the same level of abstraction"?

And then another thought surfaced: what if we designed a *strict subset* of JS, such that a smart JIT can turn it into efficient native code for the "real" host CPU?

<iframe style="float:right" width="560" height="315" src="https://www.youtube-nocookie.com/embed/BV32Cs_CMqo" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

That's how ASM.JS (2013) was born. The ground-breaking [Unreal demo][unreal] showed native or near-native performance could be achieved by the JavaScript engine of a regular browser, equipped with a JIT-compiler that could recognize a specific subset of JavaScript. Such a subset could be easily translated into native code using the [Emscripten][emscripten] toolchain, making it effectively ["The Assembly Of The Web"](https://www.hanselman.com/blog/javascript-is-assembly-language-for-the-web-sematic-markup-is-dead-clean-vs-machinecoded-html) as people had already started to call JavaScript.

This is an example of how a C function would be translated into an ASM.JS equivalent snippet:

```c
int f(int i) {
  return i + 1;
}

function f(i) {
  i = i|0;
  return (i + 1)|0;
}
```

As you may notice, this is still valid JS, albeit stylized in such a way that a "smart" interpreter could figure out extra details about the code. In this case, the `|0` annotation really is a binary `or` with `0`. Because JS does not define a proper integer type (only IEEE 754 floating points), but the binary `or` operation against a literal `0` is only meaningful when `i` is an integer, this is effectively telling the JIT that `i` *is* an integer value.

After ASM.JS was released, it became clear that JavaScript engines could support a proper compilation target; now, parsing *text* for the purpose of turning into binary instructions is just inconvenient. Let's do away with that. What if we use an efficient, compact binary representation with a lower-level semantics, that can be efficiently interpreted and JIT-compiled into native code for fast execution?

Well, that's WebAssembly.

## Conclusion

This is only the first part of our journey. In the next part, that will be kindly hosted on the [Java Advent Blog](https://www.javaadvent.com/) we will do a deeper comparison between Java and WebAssembly, and we will learn why, in spite of the name, there is more to WebAssembly than just "the Web".

[docker]: https://www.docker.com/blog/docker-wasm-technical-preview/
[nodejs]: https://nodejs.org/en/
[v8]: https://v8.dev/
[unreal]: #
[jrr]: https://dl.acm.org/doi/10.1145/1711506.1711508
[emscripten]: https://emscripten.org
[rationale]: https://github.com/WebAssembly/design/blob/main/Rationale.md 
[graalvm-ann]: #
[abi]: https://docs.oracle.com/javase/specs/jls/se7/html/jls-13.html
[draft-gc]: #