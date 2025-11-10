---
title: 'The Return of Language-Oriented Programming'
author: 'Edoardo Vacchi'
date: 2025-11-09
tags: [dsl, language-oriented-programming]
cover: 
  image: /assets/lop/cover.jpg
---

<img src="/assets/lop/cover.jpg" alt="Spoof of 'The Return of the Pink Panther' with a dragon in place of the panther, and the silhouette of the knight instead of Inspector Clouseau" title="Spoof of 'The Return of the Pink Panther' with a dragon in place of the panther, and the silhouette of the knight instead of Inspector Clouseau" width="100%"/>

I've been wondering **what LLMs mean for language design and implementation**. Some believe that, because language models are obviously trained on existing content, they are inherently less capable of assisting users with new programming languages. Intuitively this makes sense. However:

- [Simon Wilson has a "hunch"](https://simonwillison.net/2025/Nov/7/llms-for-new-programming-languages/#atom-everything) that LLMs actually make it _easier_ to build a new programming language;
- [Richard Felman has argued that this might be the "best time to create new programming languages"](https://www.youtube.com/watch?v=ZsBHc-J9f8o); 
- [Maxime Chevalier has been developing her own experimental programming language](https://x.com/Love2Code/status/1950622166166241767?ref_src=twsrc%5Etfw) called [Plush](https://github.com/maximecb/plush), porting ["many example programs with the help of LLMs"](https://x.com/Love2Code/status/1986900389631811723). 

If anything, LLMs might be shifting the cost of programming language development economics, making it possibly *even simpler* to build your own. As a self-professed ["programming-language nerd" and compiler-enthusiast](https://x.com/evacchi/), that just makes me excited.

**Disclaimer**: you should _not_ think of this as of a fully-fleshed out essay, rather it is a collection of ideas that I just want share in the hope to spark some interesting conversation! Some code examples have been generated using Claude and a bit of Python. If you find mistakes, [let](https://twitter.com/evacchi) [me](https://bsky.app/profile/evacchi.dev) [know](https://mastodon.social/@evacchi)! 

## Domain-Specific Languages and Language-Oriented Programming&nbsp;

One of my favorite articles about domain-specific languages is a classic from 1994 where M.C. Ward introduces the idea of [Language-Oriented Programming][lop]. In essence, LOP extends the idea of designing a large software system in layers, with one layer being a language definition.

<div style="text-align:center"><img src=/assets/lop/dsl-middle-out.png width=100% alt="The 'middle-out' diagram found in the paper next to the poster for the Silicon Valley series, where 'middle-out compression' is discovered over a decidedly NSFW argument" title="The 'middle-out' diagram found in the paper next to the poster for the Silicon Valley series, where 'middle-out compression' is discovered over a decidedly NSFW argument" />
<span style="font-style: italic; font-size: small; line-height: .5">The 'middle-out' diagram found in the paper next to the poster for the Silicon Valley series, where 'middle-out compression' is discovered over a decidedly NSFW argument</span>
</div>

Rather than traditional top-down or bottom-up development, LOP proposes a "middle-out" approach:

1. Design a domain-oriented language suited for the specific problem domain
2. Split development into two independent parallel tracks:
    - Implement the system using this middle-level language
    - Implement a compiler/interpreter for the language

[Domain-Specific Languages](https://martinfowler.com/books/dsl.html) are small languages designed to focus on a specific aspect of a software system. We deal with DSLs every day: SQL can be considered a DSL, LaTeX is a DSL, AWK is a DSL, Kubernetes' YAMLs are a DSL. They are "domain-specific" because they are used to write code for a given "subdomain" of the software system. In this sense, they have been also described as a means of communcation between a developer and a "domain-expert". The holy grail of computing for many years was to let such "domain experts" write the code themselves, while developers would only validate it and deploy it in the large system.

Now it so happens that coding agents are pretty good at generating code, to the point that [Andrey Karpathy claimed](https://x.com/karpathy/status/1617979122625712128) that "The hottest new programming language is English"!

So, I've been thinking: what if instead of just generating code for the languages that LLMs already have in their training set, **we instead let _them_ generate the _implementation_ of a domain-specific language**, and _then_ **use _that_ throughout the rest of a coding session**?

But then, what would one such language look like? [Sergei Egorov shared a thread with his thoughts on a similar matter](https://x.com/bsideup/status/1955412035052900587). I myself I've been wondering if a programming language for LLMs would be some kind of mixture between a high-level and a low-level language: in that we want the language to be terse, but at the same time low-level enough to implement some kind of VM for it. 

Then again, what would "terseness" mean in this context?

## Detour: Token-Efficiency in Programming Languages

Traditional programming languages optimize for human readability, not for token efficiency within an LLM's "context window"[^1]. 

From a language design perspective, there's a **fundamental mismatch in how "tokens" are defined**: while programming language tokenizers split text at whitespace and symbols (which are then usually dropped after parsing), language models treat tokens very differently.

For instance, in a JS tokenizer, the control-flow structure:

```js
for (let i = 0; i <= 10; i++) {
    print(i);
}
```

would be tokenized as follows (notice how similar types of tokens are color-coded the same)

<div>


<style>
    .token-container {
        line-height: 1.8;
        font-family: monospace;
        margin: 20px 0;
    }
    
    .token {
        padding: 2px 1px;
        border: 1px solid #ccc;
        margin: 0;
    }
    
    
    .bg-0 { background-color: #ffcdd2; }
    .bg-1 { background-color: #f8bbd0; }
    .bg-2 { background-color: #e1bee7; }
    .bg-3 { background-color: #d1c4e9; }
    .bg-4 { background-color: #c5cae9; }
    .bg-5 { background-color: #bbdefb; }
    .bg-6 { background-color: #b3e5fc; }
    .bg-7 { background-color: #b2dfdb; }
    .bg-8 { background-color: #c8e6c9; }
    .bg-9 { background-color: #dcedc8; }
    .bg-10 { background-color: #f0f4c3; }
    .bg-11 { background-color: #fff9c4; }
    .bg-12 { background-color: #ffecb3; }
    .bg-13 { background-color: #ffe0b2; }
    .bg-14 { background-color: #ffccbc; }
</style>

<pre class="token-container"><span class="token bg-0" title="keyword">for</span><span class="token bg-5" title="punctuation">(</span><span class="token bg-0" title="keyword">let</span><span class="token bg-3" title="identifier">·i</span><span class="token bg-5" title="operator">·=</span><span class="token bg-11" title="number">·0</span><span class="token bg-5" title="punctuation">;</span><span class="token bg-3" title="identifier">·i</span><span class="token bg-5" title="operator">·&lt;=</span><span class="token bg-11" title="number">·10</span><span class="token bg-5" title="punctuation">;</span><span class="token bg-3" title="identifier">·i</span><span class="token bg-5" title="operator">++</span><span class="token bg-5" title="punctuation">)</span><span class="token bg-5" title="punctuation">·{</span><span class="token bg-3" title="identifier">·print</span><span class="token bg-5" title="punctuation">(</span><span class="token bg-3" title="identifier">i</span><span class="token bg-5" title="punctuation">,</span><span class="token bg-3" title="identifier">·j</span><span class="token bg-5" title="punctuation">);</span><span class="token bg-5" title="punctuation">·}</span></pre>
</div>

this in turn, would correspond to a "parse tree" of the form:


```
ForStatement
├── Init: VariableDeclaration (let)
│   └── VariableDeclarator
│       ├── Identifier: i
│       └── Literal: 0
├── Test: BinaryExpression (<=)
│   ├── Identifier: i
│   └── Literal: 10
├── Update: UpdateExpression (++)
│   └── Identifier: i
└── Body: BlockStatement
    └── ExpressionStatement
        └── CallExpression
            ├── Identifier: print
            └── Arguments
                └── Identifier: i
```

But, for the tokenizer of an LLM, no further structure is detected before inference; the token stream would look something like the following[^2], where there is no longer any relation between colors and content (in fact, in this representation matching colors do not qualify the kind of token)

<div>

<pre class="token-container"><span class="token bg-0" title="Token 0: 'for'">for</span><span class="token bg-1" title="Token 1: ' ('">·(</span><span class="token bg-2" title="Token 2: 'let'">let</span><span class="token bg-3" title="Token 3: ' i'">·i</span><span class="token bg-4" title="Token 4: ' ='">·=</span><span class="token bg-5" title="Token 5: ' '">·</span><span class="token bg-6" title="Token 6: '0'">0</span><span class="token bg-7" title="Token 7: ';'">;</span><span class="token bg-8" title="Token 8: ' i'">·i</span><span class="token bg-9" title="Token 9: ' &lt;='">·&lt;=</span><span class="token bg-10" title="Token 10: ' '">·</span><span class="token bg-11" title="Token 11: '10'">10</span><span class="token bg-12" title="Token 12: ';'">;</span><span class="token bg-13" title="Token 13: ' i'">·i</span><span class="token bg-14" title="Token 14: '++)'">++)</span><span class="token bg-0" title="Token 15: ' {\n'">·{
</span><span class="token bg-1" title="Token 16: ' '">·</span><span class="token bg-2" title="Token 17: ' console'">print</span><span class="token bg-4" title="Token 19: '(i'">(i</span><span class="token bg-7" title="Token 22: ');\n'">);
</span><span class="token bg-8" title="Token 23: '}'">}</span></pre>
</div>

In other words, because the tokenizer of an LLM is trained on a vast amount of varied text, it is not specifically optimized for code; **semantically equivalent code can have wildly different token counts** based purely on formatting choices, identifier naming, or even the presence of comments. 

For instance, complete words like `user`, `authentication`, and `token` tend to be learned as single units; rare abbreviations (`Tkn`) may be split into multiple character-level tokens (`T`, `k`, `n`); symbols usually count as single tokens. It follows that:

- a loop written with short variable names like `i` and `j` does not necessarily consume fewer tokens than one calling them, respectively `outer` and `inner`; 
- in general, verbose but clear variable names using common English words (`userAuthenticationToken`) may be more token-efficient than abbreviations (`uat` or `usrAuthTkn`); 
- code that might look compact can still be token-heavy for a language model; comments, symbols and whitespace are often often transformed, abstracted or even dropped during parsing; however, in LLM tokenization, these syntactic construct will usually persist.

Let's consider a few examples.

### Example 1: JavaScript vs Python

When it comes to token-efficiency, a language like Python might compare more favorably to Javascript. Even in code  where complexity is comparable, Python uses fewer symbolic delimiters, favoring whitespace and full words instead. Consider:

<div>

<div class="stream output-id-3">
    <div class="output_subarea output_text">
        <h4>JavaScript (short vars): 43 tokens</h4>
    </div>
</div>


<div class="output-section">
    <pre class="token-container"><span class="token bg-0" title="Token 0: 'for'">for</span><span class="token bg-1" title="Token 1: ' ('">(</span><span class="token bg-2" title="Token 2: 'let'">let</span><span class="token bg-3" title="Token 3: ' i'"> i</span><span class="token bg-4" title="Token 4: ' ='"> =</span><span class="token bg-5" title="Token 5: ' '"> </span><span class="token bg-6" title="Token 6: '0'">0</span><span class="token bg-7" title="Token 7: ';'">;</span><span class="token bg-8" title="Token 8: ' i'"> i</span><span class="token bg-9" title="Token 9: ' &lt;='"> &lt;=</span><span class="token bg-10" title="Token 10: ' '"> </span><span class="token bg-11" title="Token 11: '10'">10</span><span class="token bg-12" title="Token 12: ';'">;</span><span class="token bg-13" title="Token 13: ' i'"> i</span><span class="token bg-14" title="Token 14: '++)'">++)</span><span class="token bg-0" title="Token 15: ' {\n'"> {
</span><span class="token bg-1" title="Token 16: ' '"> </span><span class="token bg-2" title="Token 17: ' for'"> for</span><span class="token bg-3" title="Token 18: ' ('">(</span><span class="token bg-4" title="Token 19: 'let'">let</span><span class="token bg-5" title="Token 20: ' j'"> j</span><span class="token bg-6" title="Token 21: ' ='"> =</span><span class="token bg-7" title="Token 22: ' '"> </span><span class="token bg-8" title="Token 23: '0'">0</span><span class="token bg-9" title="Token 24: ';'">;</span><span class="token bg-10" title="Token 25: ' j'"> j</span><span class="token bg-11" title="Token 26: ' &lt;='"> &lt;=</span><span class="token bg-12" title="Token 27: ' '"> </span><span class="token bg-13" title="Token 28: '5'">5</span><span class="token bg-14" title="Token 29: ';'">;</span><span class="token bg-0" title="Token 30: ' j'"> j</span><span class="token bg-1" title="Token 31: '++)'">++)</span><span class="token bg-2" title="Token 32: ' {\n'"> {
</span><span class="token bg-3" title="Token 33: '   '">   </span><span class="token bg-4" title="Token 34: ' console'"> console</span><span class="token bg-5" title="Token 35: '.log'">.log</span><span class="token bg-6" title="Token 36: '(i'">(i</span><span class="token bg-7" title="Token 37: ','">,</span><span class="token bg-8" title="Token 38: ' j'"> j</span><span class="token bg-9" title="Token 39: ');\n'">);
</span><span class="token bg-10" title="Token 40: ' '"> </span><span class="token bg-11" title="Token 41: ' }\n'"> }
</span><span class="token bg-12" title="Token 42: '}'">}</span></pre>
</div>

<div class="output-section">
    <h4>Python (short variable names): 21 tokens (51% vs JS)</h4>
    <pre class="token-container"><span class="token bg-0" title="Token 0: 'for'">for</span><span class="token bg-1" title="Token 1: ' i'"> i</span><span class="token bg-2" title="Token 2: ' in'"> in</span><span class="token bg-3" title="Token 3: ' range'"> range</span><span class="token bg-4" title="Token 4: '('">(</span><span class="token bg-5" title="Token 5: '11'">11</span><span class="token bg-6" title="Token 6: '):\n'">):
</span><span class="token bg-7" title="Token 7: ' '"> </span><span class="token bg-8" title="Token 8: ' for'"> for</span><span class="token bg-9" title="Token 9: ' j'"> j</span><span class="token bg-10" title="Token 10: ' in'"> in</span><span class="token bg-11" title="Token 11: ' range'"> range</span><span class="token bg-12" title="Token 12: '('">(</span><span class="token bg-13" title="Token 13: '6'">6</span><span class="token bg-14" title="Token 14: '):\n'">):
</span><span class="token bg-0" title="Token 15: '   '">   </span><span class="token bg-1" title="Token 16: ' print'"> print</span><span class="token bg-2" title="Token 17: '(i'">(i</span><span class="token bg-3" title="Token 18: ','">,</span><span class="token bg-4" title="Token 19: ' j'"> j</span><span class="token bg-5" title="Token 20: ')'">)</span></pre>
</div>

<div class="output-section">
    <h4>Python (readable names): 22 tokens (49% vs JS)</h4>
    <pre class="token-container"><span class="token bg-0" title="Token 0: 'for'">for</span><span class="token bg-1" title="Token 1: ' outer'"> outer</span><span class="token bg-2" title="Token 2: ' in'"> in</span><span class="token bg-3" title="Token 3: ' range'"> range</span><span class="token bg-4" title="Token 4: '('">(</span><span class="token bg-5" title="Token 5: '11'">11</span><span class="token bg-6" title="Token 6: '):\n'">):
</span><span class="token bg-7" title="Token 7: ' '"> </span><span class="token bg-8" title="Token 8: ' for'"> for</span><span class="token bg-9" title="Token 9: ' inner'"> inner</span><span class="token bg-10" title="Token 10: ' in'"> in</span><span class="token bg-11" title="Token 11: ' range'"> range</span><span class="token bg-12" title="Token 12: '('">(</span><span class="token bg-13" title="Token 13: '6'">6</span><span class="token bg-14" title="Token 14: '):\n'">):
</span><span class="token bg-0" title="Token 15: '   '">   </span><span class="token bg-1" title="Token 16: ' print'"> print</span><span class="token bg-2" title="Token 17: '('">(</span><span class="token bg-3" title="Token 18: 'outer'">outer</span><span class="token bg-4" title="Token 19: ','">,</span><span class="token bg-5" title="Token 20: ' inner'"> inner</span><span class="token bg-6" title="Token 21: ')'">)</span></pre>
</div>
</div>


Of course, you might argue that the example above is somewhat contrieved (it's just two nested loops). Let's consider another example.

### Example 2: APL vs Q vs Python

[kdb+](https://en.wikipedia.org/wiki/Kdb%2B) is quantitative finance's sweetheart; a somewhat obscure time-series DB, sporting an [APL](https://en.wikipedia.org/wiki/APL_(programming_language)) dialect called [K](https://en.wikipedia.org/wiki/K_(programming_language)), and a thin, more readable wrapper called [Q](https://en.wikipedia.org/wiki/Q_(programming_language_from_Kx_Systems)), where symbolic identifiers are often traded for more explicit English words. 

How does APL compare to an equivalent Q program, in terms of tokens? And what about Python with numpy and pandas (effectively using Python as an array-oriented DSL)?

**Update Nov 10**: The original version presented a broken 3-period weighted average; thus it also claimed, incorrectly, that the Python version was shorter (token-wise). Thanks to [Adám Brudzewsky](https://bsky.app/profile/abrudz.bsky.social/post/3m5bm3tebys2o)

In the following, we compute a 3-period moving average by symbol, selecting the symbols with average greater than 100. For instance, if we have:

```
data = {
    'sym': ['A', 'A', 'B', 'A', 'B', 'A', 'B', 'C', 'C', 'C'], 
    'price': [100, 101, 95, 102, 98, 108, 105, 96, 98, 100] 
}
```

the result is `['A']`, because `C`'s average will be greater than 100 at the end

In [Dyalog APL](https://tryapl.org/) this is written as follows (assuming two vectors, `sym` and `price`, where each index correspond to a `symbol, price` pair):

```apl
sym ← 'A' 'A' 'B' 'A' 'B' 'A' 'B' 'C' 'C' 'C'
price ← 100 101 95 102 98 108 105 96 98 100
result ← (sym { 100 < (+/3↑⌽⍵) ÷ 3 } ⌸ price) / ∪sym
result            ⍝ // prints 'A'
```

Now, notice how the usage of Unicode characters **explodes** into several non-printable tokens:

<div> <!-- APL -->

<div class="output-section">
    <h4>Dyalog APL: 33 tokens</h4>
<pre><span style="background-color: #ffcdd2; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 0: 'result'">result</span><span style="background-color: #f8bbd0; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 1: ' ←'">·←</span><span style="background-color: #e1bee7; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 2: ' ('">·(</span><span style="background-color: #d1c4e9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 3: 'sym'">sym</span><span style="background-color: #c5cae9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 4: ' {'">·{</span><span style="background-color: #bbdefb; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 5: ' '">·</span><span style="background-color: #b3e5fc; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 6: '100'">100</span><span style="background-color: #b2dfdb; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 7: ' &lt;'">·&lt;</span><span style="background-color: #c8e6c9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 8: ' (+'">·(+</span><span style="background-color: #dcedc8; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 9: '/'">/</span><span style="background-color: #f0f4c3; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 10: '3'">3</span><span style="background-color: #fff9c4; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 11: '↑'">↑</span><span style="background-color: #ffecb3; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 12: '�'">�</span><span style="background-color: #ffe0b2; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 13: '�'">�</span><span style="background-color: #ffccbc; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 14: '�'">�</span><span style="background-color: #ffcdd2; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 15: '�'">�</span><span style="background-color: #f8bbd0; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 16: '�'">�</span><span style="background-color: #e1bee7; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 17: '�'">�</span><span style="background-color: #d1c4e9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 18: ')'">)</span><span style="background-color: #c5cae9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 19: ' �'">·�</span><span style="background-color: #bbdefb; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 20: '�'">�</span><span style="background-color: #b3e5fc; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 21: ' '">·</span><span style="background-color: #b2dfdb; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 22: '3'">3</span><span style="background-color: #c8e6c9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 23: ' }'">·}</span><span style="background-color: #dcedc8; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 24: ' �'">·�</span><span style="background-color: #f0f4c3; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 25: '�'">�</span><span style="background-color: #fff9c4; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 26: '�'">�</span><span style="background-color: #ffecb3; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 27: ' price'">·price</span><span style="background-color: #ffe0b2; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 28: ')'">)</span><span style="background-color: #ffccbc; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 29: ' /'">·/</span><span style="background-color: #ffcdd2; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 30: ' �'">·�</span><span style="background-color: #f8bbd0; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 31: '�'">�</span><span style="background-color: #e1bee7; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 32: 'sym'">sym</span></pre>
</div>

Somehow surprisingly, the same code in Q, even if it's more readable and, some might argue, more verbose, has actually a slightly lower token count:  


<!--
result:·select·from·(
··update·wavg:·(1·2·3)·wavg·3·mcount·price·by·sym·from·prices
)·where·wavg·>·100
-->

<div class="output-section">
    <h4>Q (kdb+): 26 tokens</h4>
<pre><span style="background-color: #ffcdd2; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 0: 'select'">select</span><span style="background-color: #f8bbd0; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 1: ' sym'">·sym</span><span style="background-color: #e1bee7; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 2: ' from'">·from</span><span style="background-color: #d1c4e9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 3: ' (\n'">·(
</span><span style="background-color: #c5cae9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 4: ' '">·</span><span style="background-color: #bbdefb; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 5: ' select'">·select</span><span style="background-color: #b3e5fc; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 6: ' sm'">·sm</span><span style="background-color: #b2dfdb; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 7: 'a'">a</span><span style="background-color: #c8e6c9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 8: ':'">:</span><span style="background-color: #dcedc8; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 9: ' avg'">·avg</span><span style="background-color: #f0f4c3; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 10: ' '">·</span><span style="background-color: #fff9c4; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 11: '3'">3</span><span style="background-color: #ffecb3; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 12: '#'">#</span><span style="background-color: #ffe0b2; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 13: 'price'">price</span><span style="background-color: #ffccbc; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 14: ' by'">·by</span><span style="background-color: #ffcdd2; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 15: ' sym'">·sym</span><span style="background-color: #f8bbd0; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 16: ' from'">·from</span><span style="background-color: #e1bee7; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 17: ' prices'">·prices</span><span style="background-color: #d1c4e9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 18: '\n'">
</span><span style="background-color: #c5cae9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 19: ')'">)</span><span style="background-color: #bbdefb; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 20: ' where'">·where</span><span style="background-color: #b3e5fc; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 21: ' sm'">·sm</span><span style="background-color: #b2dfdb; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 22: 'a'">a</span><span style="background-color: #c8e6c9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 23: ' &gt;'">·&gt;</span><span style="background-color: #dcedc8; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 24: ' '">·</span><span style="background-color: #f0f4c3; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 25: '100'">100</span></pre>
</div>


Unsurprisingly, the Python version does have a higher token count; but, considering how terse APL and Q are, it is by a relatively small margin (only about 20-25 tokens!)

<!--
result·=·(prices
····.groupby('sym')['price']
····.apply(lambda·g:·g.tail(3).mean())
····.to_frame(name='avg')·
····.query('acg·>·100').index.tolist())
-->

<div class="output-section">
    <h4>Python (numpy, pandas): 49 tokens</h4>

<pre><span style="background-color: #ffcdd2; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 0: 'result'">result</span><span style="background-color: #f8bbd0; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 1: ' ='">·=</span><span style="background-color: #e1bee7; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 2: ' ('">·(</span><span style="background-color: #d1c4e9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 3: 'prices'">prices</span><span style="background-color: #c5cae9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 4: '\n'">
</span><span style="background-color: #bbdefb; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 5: '   '">···</span><span style="background-color: #b3e5fc; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 6: ' .'">·.</span><span style="background-color: #b2dfdb; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 7: 'group'">group</span><span style="background-color: #c8e6c9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 8: 'by'">by</span><span style="background-color: #dcedc8; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 9: " ('""="">('</span><span style="background-color: #f0f4c3; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 10: 'sym'">sym</span><span style="background-color: #fff9c4; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 11: " ')['""="">')['</span><span style="background-color: #ffecb3; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 12: 'price'">price</span><span style="background-color: #ffe0b2; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 13: " ']\n""="">']
</span><span style="background-color: #ffccbc; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 14: '   '">···</span><span style="background-color: #ffcdd2; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 15: ' .'">·.</span><span style="background-color: #f8bbd0; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 16: 'apply'">apply</span><span style="background-color: #e1bee7; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 17: '(lambda'">(lambda</span><span style="background-color: #d1c4e9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 18: ' g'">·g</span><span style="background-color: #c5cae9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 19: ':'">:</span><span style="background-color: #bbdefb; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 20: ' g'">·g</span><span style="background-color: #b3e5fc; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 21: '.tail'">.tail</span><span style="background-color: #b2dfdb; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 22: '('">(</span><span style="background-color: #c8e6c9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 23: '3'">3</span><span style="background-color: #dcedc8; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 24: ').'">).</span><span style="background-color: #f0f4c3; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 25: 'mean'">mean</span><span style="background-color: #fff9c4; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 26: '())\n'">())
</span><span style="background-color: #ffecb3; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 27: '   '">···</span><span style="background-color: #ffe0b2; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 28: ' .'">·.</span><span style="background-color: #ffccbc; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 29: 'to'">to</span><span style="background-color: #ffcdd2; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 30: '_frame'">_frame</span><span style="background-color: #f8bbd0; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 31: '(name'">(name</span><span style="background-color: #e1bee7; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 32: " ='""="">='</span><span style="background-color: #d1c4e9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 33: 'avg'">avg</span><span style="background-color: #c5cae9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 34: " ')""="">')</span><span style="background-color: #bbdefb; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 35: ' \n'">·
</span><span style="background-color: #b3e5fc; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 36: '   '">···</span><span style="background-color: #b2dfdb; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 37: ' .'">·.</span><span style="background-color: #c8e6c9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 38: 'query'">query</span><span style="background-color: #dcedc8; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 39: " ('""="">('</span><span style="background-color: #f0f4c3; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 40: 'ac'">ac</span><span style="background-color: #fff9c4; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 41: 'g'">g</span><span style="background-color: #ffecb3; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 42: ' &gt;'">·&gt;</span><span style="background-color: #ffe0b2; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 43: ' '">·</span><span style="background-color: #ffccbc; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 44: '100'">100</span><span style="background-color: #ffcdd2; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 45: " ').""="">').</span><span style="background-color: #f8bbd0; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 46: 'index'">index</span><span style="background-color: #e1bee7; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 47: '.tolist'">.tolist</span><span style="background-color: #d1c4e9; padding: 2px 1px; border: 1px solid #ccc; margin: 0;" title="Token 48: '())'">())</span></pre>

</div>
</div> <!-- END APL -->

### Example 3: Token-Oriented Object Notation (TOON)

I like this example, because I did not come up with it. [Johann Schopplich](https://x.com/jschopplich) has proposed the ["Token-Oriented Object Notation" (TOON)](https://github.com/toon-format/toon#readme), a more compact alternative to JSON. The authors claim "typically 30-60% fewer tokens on large uniform arrays vs formatted JSON". In the words of its readme:

> AI is becoming cheaper and more accessible, but larger context windows allow for larger data inputs as well. **LLM tokens still cost money** – and standard JSON is verbose and token-expensive:
> 
> ```json
> {
>   "users": [
>     { "id": 1, "name": "Alice", "role": "admin" },
>     { "id": 2, "name": "Bob", "role": "user" }
>   ]
> }
> ```
> 
> TOON conveys the same information with **fewer tokens**:
> 
> ```
> users[2]{id,name,role}:
>   1,Alice,admin
>   2,Bob,user
> ```

TOON is a perfect example of a token-efficient DSL.

## The Return of Language-Oriented Programming

A domain-specific language is by definition smaller in scope than a general-purpose language, so it should be easier to design and implement; moreover, if the language is designed well, it should lead to a more efficient usage of the context window.

If we can abstract away parts of our domain into a higher-level language, we can effectively use the LLM to 

1. generate the implementation of a DSL
2. generate **documentation** and **examples** for such our DSL
3. point the LLM to docs and examples and prompt it to generate more code **using our DSL**

So, instead of trying to come up with a general-purpose language for LLMs, we define a tiny DSL for each specific subsystem we mean to realize. The domain-specific Language is now not only a means of communication between a domain-expert and the developer, but also a means of communication between the developer, the domain-expert and the language model. 

I'm going to show a couple of examples

### Example 1: Piano DSL

Days ago, I stumbled upon [Alexander Kaminski' blog post](https://xlii.space/eng/haskell-feels-easy/) about "microdiagram DSLs":

> The core concept is quite straightforward: instead of having one language for all diagrams, have multiple languages for various purposes. For example, this diagram is designed to help learn piano by illustrating the relationship between different keys.
> <div style="text-align:center"><img src="/assets/lop/dsl-microdiag-piano.png" width=50% /></div>

I immediately wondered if I could generate something similar using Claude:

> design a DSL to design piano diagrams like these:
> 
> https://xlii.space/eng/haskell-feels-easy/
> 
> then implement it in JS and show me some examples with their rendering

[This is the result](https://claude.site/public/artifacts/1660b34e-939e-4ccf-a403-4c29cbad48e8/embed?utm_source=embedded_artifact&utm_medium=iframe&utm_campaign=artifact_frame):

<a href=https://claude.site/public/artifacts/ade8dd7f-1883-4f59-9372-be464834b6d9/embed target=_blank><img src=/assets/lop/dsl-piano-gen.png width=100% /></a>

The implementation is fully-functional and interactive

### Example 2: Business Rules

Another classic example of a DSL is Business Rules Languages. To be more precise, BRLs are more of a "framework" to define business rules; then rules _then_ encode the actual domain logic.

> design and implement a business rules language. Then implement it in JS and show me some examples

[Here's the result](https://claude.site/public/artifacts/ade8dd7f-1883-4f59-9372-be464834b6d9/embed?utm_source=embedded_artifact&utm_medium=iframe&utm_campaign=artifact_frame):

<a href=https://claude.site/public/artifacts/1660b34e-939e-4ccf-a403-4c29cbad48e8/embed target=_blank><img src=/assets/lop/dsl-brl-gen.png width=100% /></a>

Now, if you ignore that [this is clearly reminiscent of Drools to the point of plagiarism](https://kie.apache.org/docs/10.0.x/drools/drools/language-reference-traditional/index.html#drl-rules-THEN-con_drl-rules-traditional), and that the implementation is really poor, you still got a functional PoC, with the added benefit that the LLM is fully aware of the syntax and can assist you in iterating over it.

The goal here isn't necessarily to implement a language perfectly; instead we can quickly iterate on a prototype, possibly delivering this to end users, while the implementation is improved. After all, this is exactly how the original [LOP paper][lop] proposed to carry on the work!

## What About Maintainance?

A **common critique** about the cost of maintaining and working with DSLs is that **you now have to maintain not just documentation but also tooling**. With the advent of LLMs, these special-purpose, small languages are much more cost-effective. 

1. docs and examples can be very often generated by the LLM itself.
2. if the main interaction pattern is code-gen via an LLM, there is far fewer pressure to implement comprehensive tooling, such as a full IDE integrations, because a coding agent might be happy enough with a CLI tool or an MCP server. Other, simpler things, like syntax coloring, can be generated as well.

**Another common critique** is about the **cost of defining an external DSL as opposed to an internal DSL**. 

- An **external DSL** is the type of language that we have shown in this blog post, with its own syntax, parser, interpreter/compilation pipeline.
- An **internal** or **embedded DSL** (also _"fluent interface"_) is a style of library design where function or method invocations are chained together to form "sentences". For instance, [Martin Fowler's classic essay mentions JMock](https://martinfowler.com/bliki/FluentInterface.html):

    ```java
    mock.expects(once()).method(“m”).with( or(stringContains(“hello”),
                                            stringContains(“howdy”)) );
    ```
    but I am sure nowadays you'll have seen plenty of those. The more the syntax of your "host" language is flexible, the more your "internal" DSL will look like its own language. For instance, at some point Scala was notorious for [going a bit too wild with operators](https://www.scala-graph.org/guides/1.x/core-initializing.html#EdgeFactories). The benefit of this style is that you don't need to define your own parser and tooling. The downside is that the "host" language is not really aware of your "guest" language, so error messages might be more inscrutable.

Depending on how flexible your language is, the boundaries between internal and external DSLs might blur: for instance [Racket allows you to define your own syntax in term of the core syntax](https://docs.racket-lang.org/guide/languages.html); in fact [Rhombus is a recent addition to the family, with whitespace-delimited, Python-like syntax](https://docs.racket-lang.org/rhombus/index.html).

In general, the way you will implement your language is really a detail, at this point. But because of the way LLMs change language economics, I would argue that the cost of defining your own syntax instead of leaning onto the host language's is now much lower; in fact, I would even dare to say that you might want to _prefer_ flexible syntax, so that you will be able to optimize for token cost.

## Conclusions

Over the years the pendulum has swung back and forth, when it comes to domain-specific languages. In late 2000s and early 2010s there was an explosion of newer programming languages, and there was a lot of excitement around DSLs, including [Debasish Ghosh's classic "DSLs in Action"](https://www.manning.com/books/dsls-in-action)[^3]. 

In recent years, there has been something of a “winter” in DSL design and development due to the high maintenance costs and the tooling expectations from end users. This blog post explored the syntactic dimension of "token-efficiency" in DSL design: I invite you to explore more of this space, including semantics; I, for one, will welcome more crazy DSL implementations!

I hope that with the avalanche of changes AI is bringing to our daily lives, it will also ignite a renewed wave of enthusiasm for language design. 

If it won't, well, I know [I'll find a way](https://x.com/evacchi/status/1971956291913634265).

[^1]: Yes, I hear you: context windows are getting larger; however, research has shown that performance may suffer at much smaller lengths than the theoretical maximum context size. e.g.:
    - Context Is What You Need: The Maximum Effective Context Window for Real World Limits of LLMs (Paulsen, 2025)
    - Context Length Alone Hurts LLM Performance Despite Perfect Retrieval (Du et al., 2025)
    - LLM Effective Context Limits – Why Does the Effective Context Length of LLMs Fall Short? (October 2024)

[^2]: Throughout the rest of the post, we used `tiktoken` with `cl100k_base`, used by GPT-4 and similar to Claude's (`tiktoken.get_encoding("cl100k_base")`)

[^3]: In fact, [Guglielmo Iozzia is currently writing a book about Domain-Specific *Small Language Models*](https://www.manning.com/books/domain-specific-small-language-models). 

[lop]: https://citeseerx.ist.psu.edu/document?repid=rep1&type=pdf&doi=825a90a7eaebd7082d883b198e1a218295e0ed3b

