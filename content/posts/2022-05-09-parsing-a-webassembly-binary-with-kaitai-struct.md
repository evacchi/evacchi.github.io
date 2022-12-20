---
tags: WASM Parser Kaitai
date: "2022-05-09T00:00:00Z"
tags: [wasm, parser, kaitai]
title: Parsing A WebAssembly Binary With Kaitai Struct
aliases:
  - '/wasm/parser/kaitai/2022/05/09/parsing-a-webassembly-binary-with-kaitai-struct.html'
---

Traditionally Java libraries come with everything but the kitchen sink. In the past, people have given a lot of crap to the JavaScript ecosystem, but if there is something I envy, is how tiny some libraries are. People are still joking about [left-pad][left-pad] to this day; in reality, the JavaScript ecosystem has come a long way ever since, and I believe we have all something to learn.

Long story short, I was looking for a Java library to parse [WebAssembly binaries][wasm-bin], and what I found was either a full-blown interpreter or something outdated. Then I remembered about [Kaitai Struct][kaitai], a parser generator for binary structures that supports a number of languages. I have read great things, but I never had a reason to try it personally; until *now*. 

![Kaitai Home](/assets/kaitai/kaitai-home.png)

[Kaitai Struct][kaitai] comes with [a rich library of supported binary formats][kaitai-bin]. Because it is a parser generator with multiple language backends, by writing one good grammar you will get support for all those languages for "free". If a `wasm` grammar was there, my search would have ended. Unfortunately, WebAssembly [is not present officially][kaitai-wasm-pr]. 

But nothing was lost: [there is a Sophos Labs repository][sophos-wasm] with a [fairly complete Kaitai grammar for Wasm binaries][sophos-wasm-kaitai]. This grammar is working in most cases except it does not support [Data Count Sections][wasm-data-count-section] and it does not parse [Custom Sections][wasm-custom-section] correctly.

Custom sections are in fact fairly important for compilers. For instance, [LLVM's WebAssembly backend stores extra metadata in a `linking` custom section][wasm-linking]. When such a section is present, the `wasm` file is an object file, and it is not expected to be directly executable. So, I decided to [fix the grammar myself][wasm-pull]; this is when I learned that [Kaitai comes with a fantastic Web-based IDE][kaitai-ide], complete with a live preview of the parse-tree and an hex-editor that syncs with the parse tree!

<video controls width="100%">
    <source src="/assets/kaitai/kaitai-sync.webm"
            type="video/webm">
    <img src=""/assets/kaitai/kaitai-ide.png" alt="Kaitai IDE">
</video>

That is pretty cool! I have [opened a PR to the original repository][wasm-pull] to fix the initial issues I found, but I am working on extending the grammar to add some support to the [custom `linking` section][wasm-linking]. Hopefully, this will make it easier to support reading a `wasm` file from multiple languages, without having to go through the entire spec every time.


[left-pad]: https://qz.com/646467/how-one-programmer-broke-the-internet-by-deleting-a-tiny-piece-of-code/
[wasm-bin]: https://webassembly.github.io/spec/core/binary/index.html
[kaitai]: https://kaitai.io
[kaitai-bin]: https://formats.kaitai.io
[kaitai-wasm-pr]: https://github.com/kaitai-io/kaitai_struct_formats/issues/158
[sophos-wasm]: https://github.com/sophoslabs/WebAssembly
[sophos-wasm-kaitai]: https://github.com/sophoslabs/WebAssembly/tree/master/Tools/Kaitai-Wasm
[wasm-data-count-section]: https://webassembly.github.io/spec/core/binary/modules.html#data-count-section
[wasm-custom-section]: https://webassembly.github.io/spec/core/binary/modules.html#binary-customsec
[wasm-pull]: https://github.com/sophoslabs/WebAssembly/pull/4
[wasm-linking]: https://github.com/WebAssembly/tool-conventions/blob/main/Linking.md
[kaitai-ide]: https://ide.kaitai.io