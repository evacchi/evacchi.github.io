---
title: 'The Return of Language-Oriented Programming'
author: 'Edoardo Vacchi'
date: 2025-11-01
tags: [dsl, language-oriented-programming]
---


I've been wondering what LLMs mean for language design and implementation. Some believe that, because language models are obviously trained on existing content, they are inherently less capable of assisting users with new languages. Intuitively this makes sense. On the other hand, if these models are good at something they are definitely excellent at mimicking patterns, and most programming languages are similar.

[Richard Felman argues that this might be the "best" time to create new programming languages](https://www.youtube.com/watch?v=ZsBHc-J9f8o); [Maxime Chevalier has been developing her own experimental programming language](https://x.com/Love2Code/status/1950622166166241767?ref_src=twsrc%5Etfw) called [Plush](https://github.com/maximecb/plush), porting ["many example programs with the help of LLMs"](https://x.com/Love2Code/status/1986900389631811723).

If anything, LLMs might be shifting the cost of programming language development economics, making it possibly even simpler to build your own. As a self-professed ["programming-language nerd" and compiler-enthusiast](https://x.com/evacchi/status/1971956291913634265), that just makes me excited.

### The Return of Language-Oriented Programming

One of my favorite articles about domain-specific languages is a classic from 1994 where M.C. Ward introduces the idea of [Language-Oriented Programming][lop].

In essence, it extends the idea of designing a large software system in layers, with making one of the layers a language definition.

So, I've been thinking: what if instead of just generating code for the languages that LLMs already have in their training set, we instead let them generate the _implementation_ of a domain-specific language, and _then_ use that throughout the rest of the coding session?

Our programming languages aren't necessarily "optimized" for use in concert with an LLM. For instance, their design is not meant to spare tokens in a relatively limited "context window". In fact, the notion of "token" itself is possibly counterintuitive from a language design perspective: the tokenizer of a programming language is aware of symbols and whole identifiers and drops whitespace in most cases. For the tokenizer of an LLM, semantically equivalent code can have wildly different token counts based purely on formatting choices, identifier naming, or even the presence of comments. 

A loop written with short variable names like `i` and `j` consumes far fewer tokens than one using `currentIndex` and `nestedIterator`, yet both correspond to a single identifier. Traditional languages prioritize human readability, expressiveness, and machine execution efficiency, not token economy.

This mismatch creates interesting tensions: 
- for instance, verbose but clear variable names using common English words (`userAuthenticationToken`) may actually be more token-efficient than cryptic abbreviations (`uat` or `usrAuthTkn`). This is because BPE tokenizers are trained on natural language, where complete words like `user`, `authentication`, and `token` are learned as single units. Meanwhile, rare abbreviations may be split into multiple character-level tokens, resulting in both poor readability and higher token counts: the worst of both worlds. 
- Symbols are often used as delimiters for scope, and they often dropped in the later stages of parsing

It follows that, even code snippets where complexity is comparable, a language like Python might compare more favorably to Javascript when it comes to token-efficiency.

<div class="stream output-id-3">
    <div class="output_subarea output_text">
        <pre>
JavaScript (short vars): 43 tokens
</pre>
    </div>
</div>

<div class="display_data output-id-2">
    <div class="output_subarea output_html rendered_html">
        <pre
            style="line-height: 1.8; font-family: monospace; font-size: 14px;"><span style="background-color: #ffcdd2; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 0: 'for'">for</span><span style="background-color: #f8bbd0; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 1: ' ('">·(</span><span style="background-color: #e1bee7; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 2: 'let'">let</span><span style="background-color: #d1c4e9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 3: ' i'">·i</span><span style="background-color: #c5cae9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 4: ' ='">·=</span><span style="background-color: #bbdefb; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 5: ' '">·</span><span style="background-color: #b3e5fc; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 6: '0'">0</span><span style="background-color: #b2dfdb; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 7: ';'">;</span><span style="background-color: #c8e6c9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 8: ' i'">·i</span><span style="background-color: #dcedc8; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 9: ' &lt;='">·&lt;=</span><span style="background-color: #f0f4c3; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 10: ' '">·</span><span style="background-color: #fff9c4; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 11: '10'">10</span><span style="background-color: #ffecb3; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 12: ';'">;</span><span style="background-color: #ffe0b2; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 13: ' i'">·i</span><span style="background-color: #ffccbc; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 14: '++)'">++)</span><span style="background-color: #ffcdd2; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 15: ' {\n'">·{
</span><span style="background-color: #f8bbd0; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 16: ' '">·</span><span style="background-color: #e1bee7; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 17: ' for'">·for</span><span style="background-color: #d1c4e9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 18: ' ('">·(</span><span style="background-color: #c5cae9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 19: 'let'">let</span><span style="background-color: #bbdefb; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 20: ' j'">·j</span><span style="background-color: #b3e5fc; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 21: ' ='">·=</span><span style="background-color: #b2dfdb; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 22: ' '">·</span><span style="background-color: #c8e6c9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 23: '0'">0</span><span style="background-color: #dcedc8; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 24: ';'">;</span><span style="background-color: #f0f4c3; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 25: ' j'">·j</span><span style="background-color: #fff9c4; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 26: ' &lt;='">·&lt;=</span><span style="background-color: #ffecb3; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 27: ' '">·</span><span style="background-color: #ffe0b2; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 28: '5'">5</span><span style="background-color: #ffccbc; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 29: ';'">;</span><span style="background-color: #ffcdd2; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 30: ' j'">·j</span><span style="background-color: #f8bbd0; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 31: '++)'">++)</span><span style="background-color: #e1bee7; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 32: ' {\n'">·{
</span><span style="background-color: #d1c4e9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 33: '   '">···</span><span style="background-color: #c5cae9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 34: ' console'">·console</span><span style="background-color: #bbdefb; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 35: '.log'">.log</span><span style="background-color: #b3e5fc; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 36: '(i'">(i</span><span style="background-color: #b2dfdb; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 37: ','">,</span><span style="background-color: #c8e6c9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 38: ' j'">·j</span><span style="background-color: #dcedc8; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 39: ');\n'">);
</span><span style="background-color: #f0f4c3; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 40: ' '">·</span><span style="background-color: #fff9c4; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 41: ' }\n'">·}
</span><span style="background-color: #ffecb3; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 42: '}'">}</span></pre>
    </div>
</div>
<div class="stream output-id-3">
    <div class="output_subarea output_text">
        <pre>
Python (cryptic abbrevs): 21 tokens (51% vs JS)
</pre>
    </div>
</div>
<div class="display_data output-id-4">
    <div class="output_subarea output_html rendered_html">
        <pre
            style="line-height: 1.8; font-family: monospace; font-size: 14px;"><span style="background-color: #ffcdd2; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 0: 'for'">for</span><span style="background-color: #f8bbd0; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 1: ' i'">·i</span><span style="background-color: #e1bee7; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 2: ' in'">·in</span><span style="background-color: #d1c4e9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 3: ' range'">·range</span><span style="background-color: #c5cae9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 4: '('">(</span><span style="background-color: #bbdefb; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 5: '11'">11</span><span style="background-color: #b3e5fc; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 6: '):\n'">):
</span><span style="background-color: #b2dfdb; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 7: ' '">·</span><span style="background-color: #c8e6c9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 8: ' for'">·for</span><span style="background-color: #dcedc8; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 9: ' j'">·j</span><span style="background-color: #f0f4c3; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 10: ' in'">·in</span><span style="background-color: #fff9c4; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 11: ' range'">·range</span><span style="background-color: #ffecb3; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 12: '('">(</span><span style="background-color: #ffe0b2; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 13: '6'">6</span><span style="background-color: #ffccbc; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 14: '):\n'">):
</span><span style="background-color: #ffcdd2; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 15: '   '">···</span><span style="background-color: #f8bbd0; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 16: ' print'">·print</span><span style="background-color: #e1bee7; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 17: '(i'">(i</span><span style="background-color: #d1c4e9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 18: ','">,</span><span style="background-color: #c5cae9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 19: ' j'">·j</span><span style="background-color: #bbdefb; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 20: ')'">)</span></pre>
    </div>
</div>
<div class="stream output-id-5">
    <div class="output_subarea output_text">
        <pre>
Python (readable names): 22 tokens (49% vs JS)
</pre>
    </div>
</div>
<div class="display_data output-id-6">
    <div class="output_subarea output_html rendered_html">
        <pre
            style="line-height: 1.8; font-family: monospace; font-size: 14px;"><span style="background-color: #ffcdd2; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 0: 'for'">for</span><span style="background-color: #f8bbd0; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 1: ' outer'">·outer</span><span style="background-color: #e1bee7; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 2: ' in'">·in</span><span style="background-color: #d1c4e9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 3: ' range'">·range</span><span style="background-color: #c5cae9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 4: '('">(</span><span style="background-color: #bbdefb; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 5: '11'">11</span><span style="background-color: #b3e5fc; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 6: '):\n'">):
</span><span style="background-color: #b2dfdb; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 7: ' '">·</span><span style="background-color: #c8e6c9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 8: ' for'">·for</span><span style="background-color: #dcedc8; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 9: ' inner'">·inner</span><span style="background-color: #f0f4c3; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 10: ' in'">·in</span><span style="background-color: #fff9c4; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 11: ' range'">·range</span><span style="background-color: #ffecb3; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 12: '('">(</span><span style="background-color: #ffe0b2; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 13: '6'">6</span><span style="background-color: #ffccbc; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 14: '):\n'">):
</span><span style="background-color: #ffcdd2; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 15: '   '">···</span><span style="background-color: #f8bbd0; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 16: ' print'">·print</span><span style="background-color: #e1bee7; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 17: '('">(</span><span style="background-color: #d1c4e9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 18: 'outer'">outer</span><span style="background-color: #c5cae9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 19: ','">,</span><span style="background-color: #bbdefb; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 20: ' inner'">·inner</span><span style="background-color: #b3e5fc; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 21: ')'">)</span></pre>
    </div>
</div>









Instead, I'd like to focus on the way we use limited resources such as context size: a large context window allows us to "waste" more tokens, but it helps very little with compute costs. Even though programming languages are well-structured relatively terse when compared to natural language, they are not necessarily token-efficient.

For instance, consider the following code:

```
... example C-like program ...
```

as compared to:

```
... example Python ...
```

as compared to:

```
pseudo-code
```

This made me think. The way _we_, as humans, think of code, does not necessarily correspond to an efficient representation for an LLM.

What an LLM-specific programming language would look like?



> "Everything is a compiler if you are obsessed enough"


Now, writing a complete, new programming language might be some heavy lift; but an LLM makes it way easier to do the heavy lifting: we can actually let our imagination roam free and let the robot sketch the parser and handle a good part of the implementation, while we just bikeshed the syntax.

Now, why would we want to do that?

....\

LM-Oriented Programming



The paper describes an approach to developing large software systems
where one key phase is to define a high-level domain-specific programming language.

Rather than traditional top-down or bottom-up development, LOP uses a "middle-out" approach:

1. Design a formally specified, domain-oriented language suited for the specific problem domain

2. Split development into two independent parallel tracks:

    - Implement the system using this middle-level language
    - Implement a compiler/interpreter for the language

Now, some programming languages and some programming language communities are more akin to doing that. For instance, languages in the LISP and Scheme lineage have a long tradition of writing their own embedded "mini-languages"; which is also one of the reasons people say that LISP codebases are hard to maintain [citation needed].





Key Advantages

- High productivity: Systems end up much smaller (order of magnitude fewer lines of code)
- Maintainability: Smaller codebase, localized design decisions
- Portability: Only the language implementation needs porting to new platforms
- Reusability: Domain knowledge captured in the language can be reused across projects
- User-enhanceable: Top-level language can serve as a powerful macro system for users



[lop]: https://citeseerx.ist.psu.edu/document?repid=rep1&type=pdf&doi=825a90a7eaebd7082d883b198e1a218295e0ed3b


As a passionate language nerd, as well as compiler contributor, and scholar the advent of LLMs
kind of scares and intrigues me.

It scares me, because it is changing the way people think about coding.

It intrigues me, because it _is_ changing the way people think about code.

Developing a programming language from scratch, even if small, is always a complex endeavour.

But what if we let the LLM do the boring, heavy-lifting part, like writing the parser?

Moreover, we talk a lot about context management.

LLMs are biased towards writing MORE code, they will write more code before they search for a library.
They work like that coworker who's not scared of writing code: they will write some code and if it won't work they will write EVEN MORE CODE until it works.

What if we used LLMs to write DSLs and then let _them_ write the code for that DSL?

1. we reduce the number of lines of code, smaller context
2. we make it clearer what the language is doing
3. we can tailor the level of the language to the "understanding" of the LLM
4. we can iterate more easily


LLMs change programming language economics 


## more notes

I might be trying Nix, because I know I won't have to waste too much time learning all the quirks of that weird config