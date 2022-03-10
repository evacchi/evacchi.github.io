---
title:  'AWK in Java with JBang!'
subtitle: ''
categories: [AWK, JBang]
date:   2022-03-01
---

A few days ago I learned about [`pz`](https://github.com/CZ-NIC/pz). A Python library that exposes a few simple one-letter shorthands for line-based editing of pipes at the command-line. I immediately thought there could be potential.

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">This is simple and clever. The default `s` variable holds the contents of stdin. <a href="https://twitter.com/jbangdev?ref_src=twsrc%5Etfw">@jbangdev</a> idea? 🤔<a href="https://t.co/pBBcIfEIJb">https://t.co/pBBcIfEIJb</a></p>&mdash; Edoardo Vacchi (@evacchi) <a href="https://twitter.com/evacchi/status/1492559555292766219?ref_src=twsrc%5Etfw">February 12, 2022</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

I liked the idea so I pestered [Max Andersen](https://twitter.com/maxandersen): what if [JBang](https://jbang.dev) supported that kind of shorthand syntax? It turned out Max was already working on something:

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">This might be possible sooner than I thought it would be.<a href="https://t.co/nO4CHgQenV">pic.twitter.com/nO4CHgQenV</a></p>&mdash; Max Rydahl Andersen (@maxandersen) <a href="https://twitter.com/maxandersen/status/1489214081127043075?ref_src=twsrc%5Etfw">February 3, 2022</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>


This is already available since [JBang 0.90.0](https://github.com/jbangdev/jbang/releases/tag/v0.90.0) (but you should use [v0.90.1](https://github.com/jbangdev/jbang/releases/tag/v0.90.1)).


I was pretty sure that this new feature could be used to implement AWK-like scripting using Java:

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">This+data frame lib = jawk !</p>&mdash; Edoardo Vacchi (@evacchi) <a href="https://twitter.com/evacchi/status/1494603353468383234?ref_src=twsrc%5Etfw">February 18, 2022</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

For the following days I have been playing around with a short `Prelude` to enhance these kind of one-liners on JBang. But today I came across a [blog post about Prig](https://benhoyt.com/writings/prig/) on [Hacker News](https://news.ycombinator.com/item?id=30498735). Prig is [«like AWK, but uses Go for “scripting”»](https://benhoyt.com/writings/prig/).

Well, that did it. I had to show I could do the same with JBang. I give you: [`Prelude.jsh`](https://gist.githubusercontent.com/evacchi/7fb37056d92f72ae88157adcbb2f6bea/raw/8cdef074fe8184e8ed964c177c5cb835c863d1d5/Prelude.jsh)

It is a very short collections of utilities. The main idea is that the class `Line` may be used to split the fields of a stdin line into an AWK-like "record". I didn't call it `record` not to confuse it with Java 16+'s `record` feature.

```java
class Line {
    private final String line;
    private final String pattern;
    private final String[] fields; 
    public final int nf;

    Line(String line_, String pattern) {
        line = line_; fields = line.split(pattern); 
        nf = fields.length == 1 ? 0 : fields.length;
    }
    Line(String line) { this(line, "\\s+"); }

    public String s(int n) { return (n == 0)? line : fields[n-1]; }
    public int i(int n) { return Integer.parseInt(s(n)); }
    public double d(int n) { return Double.parseDouble(s(n)); }
    public String toString() { return line; }
}
```

All it does is splitting a line into whitespace-separated fields. You can access a field with
`Line#s`. At index `0` you'll find the entire line; at 1..n you'll find the first..n-th field.

`Line#d`, `Line#i`, are just shorthands to convert the n-th field to a double or an integer.

`Line#nf` gives you the number of fields, just like `AWK`'s `$NF`.

There you go. Now suppose you want to print the second field for each line in `logs.txt`

```
$ cat logs.txt
GET /robots.txt HTTP/1.1
HEAD /README.md HTTP/1.1
GET /wp-admin/ HTTP/1.0
```

You would write:

```sh
$ cat logs.txt | jbang -s Prelude.jsh -c \
    'lines().map(Line::new).map(l -> l.s(2)).forEach(s -> println(s))'
```

of course, you'll need to first download `Prelude.jsh`:

```sh
$ curl -L https://bit.ly/prelude-jsh -o Prelude.jsh
```

oh, by the way, since JBang is awesome, you can also write:

```sh
$ cat logs.txt | jbang -s https://bit.ly/prelude-jsh -c \
    'lines().map(Line::new).map(l -> l.s(2)).forEach(s -> println(s))'
```

> 🚨 **Update:** JBang v0.91.0 has become even *awesomer*: you can now skip the download and use the [catalog I posted here](https://github.com/evacchi/jbang-catalog)
> 
> ```sh
> $ cat logs.txt | jbang -s prelude@evacchi -c \
>     'lines().map(Line::new).map(l -> l.s(2)).forEach(s -> println(s))'
> ```

Now, because creating a `Line` object, then mapping it and then printing each result is so frequent, I also defined a few shorthands for you:

```java
Stream<Line> $lines() { return lines().map(Line::new); }
void $$(Function<Line, Object> f) { $lines().map(f).forEach(o -> println(o)); }
```

There you go, now you can write:

```sh
$ cat logs.txt | jbang -s Prelude.jsh -c '$$(l -> l.s(2))'
```

and of course, now we can implement the example found in the [Prig blog post](https://benhoyt.com/writings/prig/)

```sh
$ cat logs.txt | jbang -s Prelude.jsh -c '$$(l -> "https://example.com" + l.s(2))'
```

But let's see how we may implement the other examples as well.

The average of the third column in `average.txt`:

```
$ cat average.txt
a b 400
c d 200
e f 200
g h 200
```

would be:

```sh
cat average.txt | jbang -s Prelude.jsh -c \
    '$lines().mapToInt(l -> l.i(l.nf)).average().ifPresent(d -> println(d))'
```

Format into millis the third row in `millis.txt`

```
$ cat millis.txt
1 GET 3.14159
2 HEAD 4.0
3 GET 1.0
```

is just:

```sh
$ cat millis.txt | jbang -s Prelude.jsh -c \
    '$lines().filter(l -> l.s(0).matches(".*(GET|HEAD).*"))
        .forEach(l -> printf("%.0fms\n", l.d(3)*1000))'
```

This is only slightly more cumbersome because `String#matches` matches against the entire line; hence requiring the leading `.*(` and the trailing `).*` in the pattern. You may easily add a shorthand to `Line` to decorate the pattern and avoid the noise.

e.g.:

```
boolean matches(int n, String pattern) { return s(n).matches(".*" + pattern + ".*"); }
```

Finally, counting word frequency in `words.txt`

```
$ cat words.txt 
The foo barfs
foo the the the
```

In fact, this does not even require the `Prelude` !

```sh
$ cat words.txt | jbang -c \
    'println(lines().flatMap(s -> Stream.of(s.split("\s+")))
        .collect(Collectors.groupingBy(Function.identity(), Collectors.counting())))'
```

The JBang line-editing feature does not stop here. You have all JBang's power at your fingertips: you can declare dependencies, extend the prelude further... have fun!

Thanks to [Ben Hoyt](https://benhoyt.com/writings/prig/) for nerd-sniping me!
