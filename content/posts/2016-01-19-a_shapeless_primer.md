---
author: Edoardo Vacchi
tags:
- Scala
- Type-level Programming
comments: true
date: "2016-01-19T12:00:00Z"
header-img: img/2016-01-19-a_shapeless_primer/water-drop.jpg
originally:
  rnduja.github.io: http://rnduja.github.io/2016/01/19/a_shapeless_primer/
subtitle: A Shapeless Primer
title: Be Like Water
aliases:
  - '/scala/type-level programming/2016/01/19/a_shapeless_primer.html'
---

Shapeless is a Scala library for [generic programming](https://en.wikipedia.org/wiki/Generic_programming). The name “Shapeless” comes from a famous Bruce Lee quote:

> Don't get set into one form, adapt it and build your own, and let it grow, be like water. Empty your mind, be formless, **shapeless** — like water. Now you put water in a cup, it becomes the cup; You put water into a bottle it becomes the bottle; You put it in a teapot it becomes the teapot. Now water can flow or it can crash. Be water, my friend.

There have been many blog posts
devoted to explaining the very basics of Shapeless, but, in my opinion, fewer posts show
how to truly *understand* how to *use* Shapeless. In this post I am trying to give an overview,
starting from a concrete situation. Hopefully, in the end, you will
have a better picture of how to read a piece of code that makes extensive use
of Shapeless and a Shapeless-like coding-style.


#### Table of Contents


- [Running Example: Stream Processing](#running-example-stream-processing)
- [Abstracting over Arity](#abstracting-over-arity)
- [HLists and Product Types](#hlists-and-product-types)
- [From Tuples to HLists and Back Again](#from-tuples-to-hlists-and-back-again)
- [The `Generic[T]` object](#the-generic-t-object)
- [The `FnToProduct[F]` object](#the-fntoproduct-f-object)
- [Implicit Value Resolution](#implicit-value-resolution)
- [A Short Prolog Digression](a-short-prolog-digression)
- [GrandChild In Scala](#grandchild-in-scala)
- [Final Remarks](#final-remarks)
- [Understanding `applyProduct` evidences and typeclasses](#understanding-applyproduct-evidences-and-typeclasses)
- [The Aux Pattern](#the-aux-pattern)
- [Bonus: `applyProduct` encoding in Prolog](#bonus-applyproduct-encoding-in-prolog)
- [Conclusions and Reference](#conclusions-and-references)


## Running Example: Stream Processing

[Reactive stream](http://reactive-streams.org) frameworks are a fancy new way to do [Dataflow programming](https://en.wikipedia.org/wiki/Dataflow_programming), where
a computation is modeled after the flow of data through the nodes of a *pipeline*, or, more generally, of a *flow graph*.

Each **node** of this graph computes a function on the data that comes in, and returns a new value that will be written onto an output stream. In the simplest case of a pipeline,
each node has exactly one input and one output. That is, it computes a function `T => R`.

The initial and final nodes are an exception:

 - the initial node *produces* data without an actual input (a function `() => R`);
 - the final node *consumes* input without producing output (a function `T => Unit`);

Beside the initial and final nodes, which can be special-cased, the node of a pipeline may be generally described by the case class:

~~~scala
case class Node[T, R](f: T => R)
~~~

But, in a stream processing framework, the more interesting case is that of *flow-graph*
rather than that of a simple pipeline. In this case, each node would compute a function
of `N>=1` parameters, where each parameter would be an in-edge into the node.
In pseudo-code:


~~~scala
case class Node[T1, T2, ..., TK, R](f: (T1, T2, ..., TK) => R)
~~~

The challenge now would be supporting K-ary function with arbitrary K, without writing K
different implementations. As you may know, tuples and functions in Scala
are implemented by explicitly defining 22 variants, with
up to 22 distinct type parameters, like so:

~~~scala
Function1[T,R]                   
Function2[T1,T2,R]              Tuple2[T1,T2]
...                             ...
Function22[T1,T2,...,T22,R]     Tuple22[T1,T2,...,T22]
~~~

The limit is known to be somewhat arbitrary, but we are not debating
this now. The point is, although the code for this could be
auto-generated, we would like to write this once and for all, and

   1. without resorting to meta-programming (macros and the like)
   2. without needless code repetition


In other words, we would like to be able to *abstract* over the arity (number of arguments) of a function.

## Abstracting over Arity

The [Shapeless wiki](https://github.com/milessabin/shapeless/wiki/Feature-overview:-shapeless-2.0.0#facilities-for-abstracting-over-arity) includes a single, interesting example that actually solves our problem.

> Conversions between tuples and HList's, and between ordinary Scala functions of
> arbitrary arity and functions which take a single corresponding HList argument allow
> higher order functions to abstract over the arity of the functions and values they are
> passed

The following incantation does the trick:

~~~scala
import syntax.std.function._
import ops.function._

def applyProduct[P <: Product, F, L <: HList, R](p: P)(f: F)
  (implicit gen: Generic.Aux[P, L], fp: FnToProduct.Aux[F, L => R]) =
    f.toProduct(gen.to(p))

scala> applyProduct(1, 2)((_: Int)+(_: Int))
res0: Int = 3

scala> applyProduct(1, 2, 3)((_: Int)*(_: Int)*(_: Int))
res1: Int = 6
~~~

Now, we *may* copy and paste the piece of code above and call it a day.
Instead, this entire post is devoted to achieve a more deep understanding of this short, but very dense snippet. First of all, let us rule out the **type parameters**. What is an HList?


## HLists and Product Types

Many blog posts have been devoted to describe the ins and the outs of HLists. This is *not* one such post. The point is, HLists are not that hard a construct to understand and even to implement. The neat part about Shapeless is that many operations on HLists are provided *out of the box*.

I want to refer you to the many posts that describe how HLists can be implemented in Scala to find more detail: see the [References](#references) section at the bottom of this post for a list. In this section I will instead explain how Shapeless HLists can be used as a substitute for tuples. In order to understand what will follow, the simple mental model that equates HLists to an alternative tuple implementation should suffice.

HLists (short for *Heterogeneous List*) are lists of objects of arbitrary types, where the type information for each object is kept. In fact, in Scala, we may do:

~~~scala
import shapeless._
val l = 10 :: "string" :: 1.0 :: Nil
~~~

but the type of `l` then would be `List[Any]`, because the common super type of the elements there would be, in fact, only `Any`.
A `HList` is declared similarly to a `List`:


~~~scala
val hl = 10 :: "string" :: 1.0 :: HNil
~~~

except for the terminator `HNil`. But the type of `hl` is actually `Int :: String :: Double :: HNil`.
As you can see, no type information is lost.

You can probably think of another family of types in vanilla Scala that act as a container for values of different types; these
types retain type information for each of their component, and the order in which they are declared: *tuples*.

HLists *can be seen* as an alternative implementation of the concept of Tuple. Or, more generally, of the concept of Product.
It is not by accident that Scala tuples all extend the `Product` trait (in fact, so do case classes; more on this later).
A `Product` is really just this: for some types `T1`, `T2`, ... , `TN` (N>=1) the product type is the n-tuple `(v1, v2, ..., vN)` (with `v1` an instance of type `T1`, etc.). When each field is *named* then the tuple is called a *record* (in vanilla Scala, case classes could be seen as some sort of record).


You already know that Scala tuples are a first-class concept; tuples are constructed through convenient literal syntax `(v1, v2, ..., vN)`; using Shapeless, HLists are constructed using the syntax `v1 :: v2 :: ... :: vN :: HNil`.

The benefit of using `HLists` instead of tuples is that they can be used in all those situations where a tuple would work just as well, **but** *without* the 22-elements limit.

~~~scala
val t  = (1,   "String",   2.0)
val hl =  1 :: "String" :: 2.0 :: HNil

val (a,   b,   c)        = t
val  x :: y :: z :: HNil = hl

t match {
  case (1, s, _) => ...
}

hl match {
  case 1 :: s :: _ :: HNil => ...
}
~~~

Moreover, Shapeless `HLists` provide many of the operations that can be usually applied to `Lists` such as `map`, `flatMap` etc. But, as a simple substitute for tuples, HLists are already quite valuable; for instance, it is not necessary to know the size of an HList before being able to match against it, rendering possible to do things like:

~~~scala
hl match {
  case 1 :: rest => ... // matches if first element 1, regardless the size of the list
                        // and binds the tail to `rest`
  case x :: y :: HNil => ... // matches when it is a pair
  // etc.                      
}
~~~


## From Tuples to HLists and Back Again

Because Tuples and HLists are essentially the implementation of a similar concept,
they are also *isomorphic* to eachother, that is, there is a *morphism* (a function)
to go back and forth between them.

The way we go **from HLists to Tuples** is through the `tupled` method.

~~~scala
import syntax.std.product._
val hl  = 1 :: "String" :: 2.0 :: HNil
val  t  = hl.tupled
~~~

The way we go **from Tuples to HLists** is through the `productElements` method.
In fact, if `hl` is the HList, and `t` is the equivalent tuple, then it is always true that:

~~~scala
hl == t.productElements
~~~

Of course, a simple consequence is that you are able to abstract over the arity of **tuples** by *transforming them into HLists*

~~~scala
aTuple.productElements match {
  case a :: b :: HNil => ...
}
~~~

Whatsmore, because all tuples implement the `Product` trait
and **case classes** implement the `Product` trait, many of the operations
working for tuples also work for case classes.

~~~scala
case class Person(name: String, age: Int)
Person("John", 40).productElements match {
  case name :: age :: HNil => ...
}
~~~

It is even possible to create instances of a case class from an HList.
Let us see how.

## The `Generic[T]` object

A `Generic[T]` object implements the methods `to(T)` and `from(HList)` for a given product type `T` (usually, a case class or a tuple). For instance, the `Generic` object to convert back and forth between a `Person` and an `HList` is created and used as follows:

~~~scala
val gp = Generic[Person]
val john = Person("John", 40)
val hl: String :: Int :: HNil = gp.to(john)
val p: Person = gp.from(hl)

assert( john == p )
~~~

This is likewise true for tuples.

~~~scala
val gp = Generic[Tuple2[String,Int]]
val johnTuple = ("John", 40)
val hl: String :: Int :: HNil = gp.to(johnTuple)
val tp: Tuple2[String,Int] = gp.from(hl)

assert( johnTuple == tp )
~~~

In fact, the `t.productElements` method is really short-hand syntax for the code snippet above.

## The `FnToProduct[F]` object

Suppose you have function `f` that takes `K` arguments, and that you want to conflate
those arguments into one single product-type argument.

Scala provides `f.tupled` which turns a K-ary function into a unary function of one K-tuple argument. For instance:

~~~scala
val f = (s: String, i: Int) => s"$s $i"
val ft: Tuple2[String, Int] => String = f.tupled
~~~

But then, again, as we have seen already, you are not really able to *abstract* over the arity of Scala tuples. However, we can import `FnToProduct[F]` to turn a K-ary function into a function of K-sized HList of the same argument types.

~~~scala
import ops.function._
val fp = FnToProduct[(String, Int) => String]
val fhl: String::Int::HNil => String = fp.apply(f) // or, equivalently, fp(f)
~~~
in fact, there is even syntactic sugar for this:

~~~scala
import syntax.std.function._
val fhl: String::Int::HNil => String = f.toProduct
~~~

So now we have already sufficient elements to understand the following line of the Shapeless wiki.

~~~scala
f.toProduct(gen.to(p)) // could have been written as f.toProduct.apply(gen.to(p))
~~~

First of all, let us consider a special case;
imagine `f` is the function we have just defined above, and let `p = Person("John", 40)`.
Now, the line above is a bit terse. Let us expand it a little bit:

~~~scala
val gen = Generic[Person]
val fhl: String :: Int :: HNil => String = f.toProduct
val hl: String :: Int :: HNil = gen.to(p)
fhl(hl)
~~~
 - we get `gen`, a `Generic[Person]` instance
 - we convert `f` to an HList-accepting function (`f.toProduct`);
 - we convert `p` to the HList `hl`  by applying the `to(Person)` method of `gen`

There is still something missing, though. In the expanded version we have explicitly requested the
`Generic[Person]` instanced to be created.
The wiki uses *implicit parameters*. How do these *implicits* work?
What is their relation to `Generic` and `FnToProduct`? And, do we really need them there?

## Implicit Value Resolution

Implicits are a controversial feature of the Scala programming language.
They are hard to grasp for beginners, and they can be cause of headaches
even to the more experienced users. The choice of the word *implicit* is arguable. It
probably contributed to add to Scala's *black magic* fame.

You are probably already aware that *implicits* are values that, when *in scope*,
are automatically inserted as required by the compiler.
When one such *implicit value* is in scope, then the implicit arguments
can be completely omitted by who is writing the code. For instance:

~~~scala
implicit val implicitJohn = Person("John", 40)
def somePersonString(implicit p: Person) = p.toString

somePersonString // returns "Person(John, 40)"
~~~


Implicits can be brought into scope by explicitly declaring them, as in the example
above, by *importing* them from a library.
The most simple use case for implicits is to provide *fallback values*,
but they can be exploited for other advanced use, such as validating nontrivial
constraints. An example of this is the encoding of
[*dependent types*](http://rnduja.github.io/2015/10/07/scala-dependent-types/).


`implicit def`s can be used to *generate* implicit values as needed.
The most simple use for implicit defs is to provide *implicit type conversions*
that are often used to define [extension methods](https://en.wikipedia.org/wiki/Extension_method).
Implicit defs are also used when implicit parameters of a function are being resolved
to see if a matching value can be *generated* on-the-fly.

For instance, consider this example:

~~~scala
implicit def personProvider = Person("Random Guy", (math.random * 100).toInt)
def somePersonString(implicit p: Person) = p.toString

> somePersonString // Person("Random Guy", and a random integer between 0 and 100)
~~~


Let me repeat this once again: implicit parameter **resolution** is a compile-time
procedure that does not affects run-time performance. Implicit values
that are generated by implicit defs, on the other hand, may obviously affect
run-time since they are instances of classes which may have arbitrary code in
their constructors; on the other hand, the JVM is pretty good
at optimizing out final classes, singletons and the like,
but be advised that a matching implicit def is code that
*will actually execute* at run time!

~~~scala
implicit def aSlowPersonProvider = {
	Thread.sleep(3000) // faking a slow computation here
	Person("Random Guy", (math.random * 100).toInt)
}
def somePersonString(implicit p: Person) = p.toString

> somePersonString // sits 3 secs, then returns the Person instance
~~~

Implicit parameter resolution can be twisted
to enforce stronger constraints than simple type-checking. Parameterized types
such as `Generic.Aux[P,L]` in the Shapeless wiki
can be used to encode *predicates* about types, and *relations* between them.
Your compiler checks for conformity to type signatures when
you invoke methods or create class instances.
In the case of implicits, though, the compiler
*attempts* to fill in the voids *automatically*.
The voids will be filled *if and only if* these relations hold.

In the previous example, we were asking the compiler to
execute method `somePersonString` **if and only if** a value
of type `Person` could be brought into scope. To put it in another way,
we wanted the method invocation to compile **if and only if**
the compiler could *prove* that a matching value (a value of that type)
*may exist*.

When I saw this, I had one of those "aha!"-moments. The key to
understanding implicit resolution in Scala, to me, was finding the parallel that exist
with the evaluation of a program in a logic programming language such as *Prolog*.

### A Short Prolog Digression

In the Prolog programming language you may state *facts* and provide *rules*
to derive new facts. These facts and rules are kept in a space called a *knowledge base* (or *KB*),
a database of all the facts and rules that have been defined.

The knowledge base, can be *queried*, like you would do on a SQL database, although
with a different query language. And, like in any other query language,
the execution of a Prolog program consists in solving the constraints
that are found in the query.


For instance, let us say that John, Carl and Tom are all persons. We do this by writing the Prolog listing:

~~~prolog
person(john).
person(carl).
person(tom).
~~~

These are all *facts* that we are stating about the atomic values john, carl, and tom (notice that they are all in small-letters; capitals are reserved to variables).
You can now check whether `john` is a person with the query

~~~prolog
?- person(john).
true
~~~

You can also get all the persons in the KB:

~~~prolog
?- person(X).
~~~

Where `X` is a *free variable* (Prolog use capitals to denote variables), that is, a *fresh* variable that is not bound to any value.
In this case we are asking the interpreter to find a *concrete proof* (or *evidence* or *witness*)
that the condition holds in the KB.  In other words, we want to find *at least* one binding of `X` for which `person(X)` is true;
By hitting `;` we can request the next binding.

~~~prolog
?- person(X).
X = john ;
X = carl ;
X = tom
~~~

Let us now provide relations; that is K-ary facts about these persons.

~~~prolog
child(john, carl).
child(carl, tom).
~~~


we can now instruct Prolog on how to *derive* further relations between these persons using *rules*. For instance, let us define the `grandchild` rule.

~~~prolog
grandchild(A, B) :-
	child(A, X),
	child(X, B).
~~~

That is, `A` and `B` are in a grandchild relation if `A` is child of some `X`
and `X` is a child of `B`. A rule describes an implication relation, in fact, this, in logic may be written as:

~~~
∀ A, B, X: child(A,X) ∧ child(X,B) → grandchild(A,B)
~~~

Notice that the implication is written in reverse, compared to the Prolog version.
The Prolog version results more readable in code, because it makes more
prominent the fact that the implication will derive.
The interpreter can be queried in a number of ways. Again, small letters
denote atoms, while capitals indicate free variables, we can choose any combination of the two
to produce different results.

~~~prolog
% find the pair(s) X,Y for which the grandchild relation holds
?- grandchild(X, Y).
X = john
Y = tom

% find the john's grandchildren
?- grandchild(john, Y).
Y = tom

% find tom's grandparents
?- grandchild(X, tom).
X = john
~~~

The system will also fail in *impossible* cases, that is, when we try to find
something that is not derivable in the knowledge base.
For instance, looking for someone who is his/her own grandchild

~~~prolog
?- grandchild(X, X)
no
~~~

or looking for someone who is grandchild of tom (who has no grandchildren)

~~~prolog
?- grandchild(tom, X)
no
~~~

or looking for the granparent of john, which is unknown in this knowledge base

~~~prolog
?- grandchild(X, john)
no
~~~

### GrandChild In Scala

In order to better understand the relation between Prolog and Scala, let us first
introduce a dialect of Scala that we will call *Logic Scala*.

*Logic Scala* is a superset of Scala where two new keywords are introduced:

* `fact`
* `rule`

Let us see how an implementation of the `grandchild` example may look in Logic Scala.
First, we have to introduce facts about children. The relation may be declared
as a parameterized type:

~~~scala
trait Child[A,B] // notice that a class would work as well
~~~

where A,B will be substituted by atoms. Atoms are, again, types, so we may declare them as follows

~~~scala
trait John
trait Carl
trait Tom
~~~

We can now declare *facts* (remember, this is *Logic Scala*):

~~~scala
fact johncarl = new Child[John,Carl]{} // an instance of type Child
fact carltom  = new Child[Carl,Tom ]{}
~~~

The rule for `grandchild` is made of two parts. The type declaration,

~~~scala
trait GrandChild[A,B]
~~~

and the semantics of the rule, which is given using the `rule` construct.
Just like in Prolog, our imaginary Scala dialect would describe how to derive
new instances of the type `GrandChild[A,B]` from facts that are found (or that can
be derived) in the current knowledge base. A `rule` is written similarly to
a `def`:

~~~scala
rule grandChild[A,B,X](  
	facts
		xy: Child[A,X],
		yz: Child[X,B]
	) = new GrandChild[A,B] {}
~~~

Notice that, because a `rule` declaration is syntactically similar to a `def` declaration, it is written in the same order of the abstract logic declaration, while the Prolog version is written in reverse. However, if we remove the syntactic noise, this is really stating the same:

~~~scala
rule[A,B,X](Child[A,X], Child[X,B]): GrandChild[A,B]
~~~

that is, for some A, B, X, *if* there exist `Child[A,X]`, `Child[X,B]`, *then*
a `GrandChild[A,B]` can be derived.

we can now write a query. The query will compile if and only if the
compiler is able to satisfy the contraints using the facts and the rules in the knowledge base.

~~~scala

> query[ GrandChild[John, Tom] ]
// (compiles; returns the fact instance)

> query[ GrandChild[John, Carl] ]
Compilation Failed
~~~

You should now be more convinced that there is a strong correspondence between
the Logic Scala version and the Prolog version. Truth is, the only difference between Logic Scala and real-world Scala is that we renamed a few keywords. The code and machinery are actually the same!

~~~scala
trait John
trait Carl
trait Tom
trait Child[T,U]
implicit val john_carl = new Child[John,Carl]{}
implicit val carl_tom  = new Child[Carl,Tom ]{}

trait GrandChild[T,U]
implicit def grandChild[X,Y,Z](
	implicit
		xy: Child[X,Y],
		yz: Child[Y,Z]
	) = new GrandChild[X,Z] {}


> implicitly[ GrandChild[John, Tom] ]
// (compiles; returns the fact instance)

> implicitly[ GrandChild[John, Carl] ]
Compilation Failed
~~~


You should now be convinced that there is a correspondence between the way logic
programs are evaluated and the way implicits are resolved at compile-time.
In fact,

* **implicit vals** can be seen as **facts**
* **implicit defs** can be seen as **rules**
* **type parameters** in a **def** correspond to **variables** in a rule definition
* **implicit parameter lists** are bodies of a rule


### Final Remarks

Please recall that in Prolog we could even write

~~~prolog
?- grandchild(john, Y).
~~~

The query above is asking for *evidence* that there exist a granchild of John in the KB.
The interpreter would return the `Y` binding for which the condition holds; i.e., `Y=tom`.
Something similar can be achieved in Scala through *existential types*.
The query above could be encoded as:

~~~prolog
> implicitly [ GrandChild[John, Y forSome { type Y }] ]
~~~
this encodes *exactly* the same query: find *evidence* (an object instance)
that there exist `Y` such that `GrandChild[John, Y]` is a valid statement (a concrete type).

This can be also more concisely expressed as:

~~~prolog
> implicitly [ GrandChild[John, _] ]
~~~

There are caveats, though.

First of all, Scala will not return the actual binding for which the condition holds.
In fact, this does not make sense, because, in general, when you write `GrandChild[John, _]`
you are *representing the type* of something that is in a `GrandChild` relation with John. It may be
an instance of `GrandChild[John, Tom]`, but there might be others at some point in time. You do not
want `GrandChild[John, _]` to *always* be an alias for `GrandChild[John, Tom]`

Second, the symmetric query `GrandChild[_, Tom]` will not work in this case. The deep reasons for which
are a bit technical and would go beyond the scope of this essay. In short, Scala evaluates
the implicit parameters in the `def` signature in the order they are defined; thus, when you write `GrandChild[_, Tom]`
the system looks for an implicit instance of `Child[_,Y]`, but this cannot be derived, because `Y` is still
unknown at that time. In fact, if we define another `implicit def` where the implicit values are re-ordered:

~~~scala
implicit def grandChildReordered[X,Y,Z](
	implicit
	    yz: Child[Y,Z],
	    xy: Child[X,Y]
	) = new GrandChild[X,Z] {}


> implicitly[ GrandChild[_, Tom] ]
// compiles

> implicitly[ GrandChild[John, _] ]
// compiles
~~~

`GrandChild[_,_]` will only work if you explicitly "assert" it:

~~~scala
implicit val fallback: GrandChild[_,_] = new GrandChild[Nothing,Nothing]{}
> implicitly[ GrandChild[_, _] ]
~~~

We are now ready to understand how type parameter inference and implicit resolution work in `applyProduct`.

## Understanding `applyProduct`: evidences and typeclasses

We already saw what the *body* of the `applyProduct` function did; we left for later the description of the
implicit parameters. Because now we have found the parallel between Scala and Prolog, we can easily describe
what is happening in the function:


~~~scala
def applyProduct[P <: Product, F, L <: HList, R](p: P)(f: F)
  (implicit
  	 gen: Generic.Aux[P, L],
  	 fp: FnToProduct.Aux[F, L => R]
  ) = f.toProduct(gen.to(p))
~~~

* Let be `P` a `Product`, that is, a tuple or a case class
* Let be `F` an unconstrained type parameter
* Let be `L` an `HList`
* Let be `R` an unconstrained type parameter

then,

* For a given product of type `P`
* For a given object of type `F` (the function)

we can invoke `applyProduct` if the following relations hold:

* `Generic.Aux[P, L]`; this is the built-in “predicate” that Shapeless provides
  to encode the relation between a product type `P` and an HList `L`. It holds
  when it is possible to derive a `Generic[P]`  instance that converts `P` into `L`
* `FnToProduct.Aux[F, L => R]`; is he built-in “predicate” that Shapeless provides
  to encode the relation that holds when `F` can be converted into a function from HList `L` to return type `R`;
  it holds when it is possible to derive an `FnToProduct[F]` instance called that converts `F` into `L => R`


Congratulations, you have just learned how to use **typeclasses** !
In fact, the values `gen` and `fp` are *instances* of the `Generic[P]` and `FnToProduct[F]` *typeclasses*.

As we saw in the previous section, the value of an implicit parameter can be seen as *evidence* that a *predicate* holds.
In Scala, we implement predicates through *types*, and *evidences* are object instances. If these object instances
provide methods, then they are called *typeclasses*.

For instance, we saw before that `Generic[P]` provides the methods `to(P)` and `from(HList)`.


### The Aux Pattern

We have left only one little detail. Why are the predicates called `Generic.Aux` and `FnToProduct.Aux`
but the type classes are just `Generic` and `FnToProduct` ?
Again, this is a bit technical.


`Generic.Aux` and `FnToProduct.Aux` are actually *type aliases*. You may know that scala includes a
*type alias* feature. Type aliases are also called *type members* because, just like properties
and methods, they occur as *members* of another type. For instance, the `Generic[P]` trait:

~~~scala
trait Generic[P] {
  type Out
}
~~~

The type alias `Generic.Aux` is defined in the companion object of `Generic`

~~~scala
object Generic {
  type Aux[P,L] = Generic[P] { type Out = L }  
  ...
}
~~~
and it reads as follows:
`Generic.Aux[P,L]` is the type of `Generic[P]` when the type alias `Generic.Out` within it equals to `L`.
Again, this is a *predicate*. When we write `(implicit gen: Generic.Aux[P,L])`
or equivalently `(implicit gen: Generic[P] { type Out = L })`; we want the compiler to *prove*
that there exists (or that it can be derived) in current scope an *evidence* that
`Generic[P] { type Out = L }`; in particular, in the case `Generic.Aux[Person, L]`
we want proof that there exist some HList `L` such that `Person` can be converted into `L`.

But then, you may ask, why can we just write `Generic[P,L]`, and be done with it ?
Recall that in one of our first examples we wanted to convert a `Person(String,Int)` into the corresponding HList:

~~~scala
val gen = Generic[Person]
val hl = gen.to(p)
~~~

If `L` were part of the type signature of `Generic` then we would need to write:

~~~scala
val gen = Generic[Person, L]
~~~

but then `L` would be a fresh type parameter, which cannot occur in that position!

~~~scala
scala> val gen = Generic[Person, L]
<console>:19: error: not found: type L
       val gen = Generic[Person, L]
                                 ^
~~~

The type member trick is useful to “hide” the type parameter in those situations where it is not only unnecessary (the type `L` of the HList is **univocally** determined by `Person`) but also a problem, because we would now be required to explicitly give the HList type
for all the `Generic` instances we wanted to create!

This is what we usually mean when we say that *type members* can be used to *hold the result* of a *type-level computation*;
in this case type member `Generic.Out` contained the result of *automaticaly deriving* `L` from `Person`.

Visavis, `Generic.Aux[P,L]` is sometimes called a *type extractor* because it *extracts* the type member value `Generic.Out`
by raising it into the signature of type `Generic.Aux`.



### Bonus: `applyProduct` encoding in Prolog

We will show how the `applyProduct` works by encoding it in Prolog.

The type-level constraints of our running example can be re-implemented in Prolog.
A tuple can be defined just like in Scala:

~~~prolog
(arg_type1, arg_type2, ..., arg_typeN)
~~~

Since we know that in Scala we can always transform a `FunctionN` into a `Function1` of a `TupleN` for all `N>1`, we can represent a function type as:

~~~prolog
fn((arg_type1, arg_type2, ..., arg_typeN), ret_type)
~~~
and, when there is only one arg, simply:

~~~prolog
fn(arg_type1, ret_type)
~~~

for instance, the type signature for function `def stringify(s: Any): String`
could be represented by `fn(any, string)`


Now, let us try and translate the constraints in the signature of `applyProduct` into Prolog predicates.

~~~scala
def applyProduct[P <: Product, F, L <: HList, R](p: P)(f: F)
  (implicit
  	 gen: Generic.Aux[P, L],
  	 fp: FnToProduct.Aux[F, L => R]
  ) = ...
~~~

In the previous section we saw that type parameters in a `def` can be seen as the variables in a Prolog rule,
while the rule body can be expressed using type parameters.

We may define a `can_apply_product(P,F,L,R)` predicate as follows:



* `P` must be a `Product`. So, let us define the `product` predicate which holds
  when `P` is a Product.

	For instance, in the case of a pair:

  ~~~prolog
  product(X):- X = (_,_).
  ~~~

	Or, in short:

  ~~~prolog
  product((_,_)).
  product((_,_,_)).
  ... % for >3 tuples
  ~~~

* `F` is unconstrained (in the sense that there is no unary predicate on F)

* `L` must be an `HList`, which in Prolog we can represent as a list.
    We can define the predicate `hlist`, which holds when `L` is a list.

  ~~~prolog
  hlist(X):- X = [_|_]. % [Head | Tail]
  product([_|_]). % in short.
  ~~~

* `R` is unconstrained (in the sense that there is no unary predicate on F)

* `Generic.Aux[P,L]` puts in relation product `P` with HList `L`. In Prolog, this would a predicate. Let us see it for the case of a pair `P`:

  ~~~prolog
  generic(P, L) :-
      product(P),
  	(X, Y) = P,  % deconstruct P into X and Y (the correct term is «unify»)
  	hlist(L),
  	[X, Y] = L.  % deconstruct L into X and Y
  ~~~

	But, really, in Prolog we can drop the `product` and `hlist` constraints, because
	they are really implied by the syntax `(X,Y)` and `[X,Y]`, respectively. So this can be further shortened:

  ~~~prolog
  generic((X,Y), [X,Y]).
  ~~~

* `FnToProduct.Aux[F, L => R]` puts in relation the type parameter `F` with the function type `L => R`. In Prolog, this could be described as the predicate

  ~~~prolog
  fn_to_product(F, fn(L, R)):
  ~~~

 in fact, we decided to represent any `T => R` function type using `fn(T,R)`.
 The `fn_to_product` predicate shall hold when

 1. `F` is a function
 2. the tuple of the types of the arguments of `F` can be converted into a (h)list

 This can be written as follows:

~~~prolog
fn_to_product(F, fn(L, R)) :-
   fn(Args,R) = F,
   generic(Args, L).    
~~~

  or, in short:

~~~prolog
fn_to_product(fn(Args,R), L, R) :-
   generic(Args, L).    
~~~

The `can_apply_product` predicate, which encodes all of the type constraints
in the `applyProduct` Scala function, puts in relation:

* the product `P`
* the function `F`
* the list `L` of the arg types of `F`
* the return type `R` of `F`

Let's see Prolog and Scala side-by-side:

<table>
<tr><td><code>
<pre>
can_apply_product(P, F, L, R) :-

    generic(P,L),
	fn_to_product(F, fn(L, R)).
</pre></code>
</td><td><code><pre>
def applyProduct[P <: Product, F, L <: HList, R](p: P)(f: F)
  (implicit
  	 gen: Generic.Aux[P, L],
  	 fp: FnToProduct.Aux[F, L => R])
</pre></code>
</td></tr>
</table>

So, let's see it as a whole:


~~~prolog
generic((X,Y), [X,Y]).

fn_to_product(fn(Args,R), L, R) :-
    generic(Args, L).

can_apply_product(P, F, L, R) :-
    generic(P,L),
	fn_to_product(F, fn(L, R)).
~~~

## Conclusions and References

This long blog post has only scratched the surface of Shapeless. If this has whetted your appetite, then I suggest you to delve further into Shapeless's inner guts. I have compiled for you a short list of resources that you may want to have a look at.

### Longer Essays
* [Our own Andrea Ferretti's step-by step introduction to dependent types](http://rnduja.github.io/2015/10/07/scala-dependent-types/) encoded in Scala's type system
* [Mark Harrah's series](https://apocalisp.wordpress.com/2010/06/08/type-level-programming-in-scala/) on type-level programming on *ApocaLisp* blog

### Slides and Courses

* [Stefan Zeiger's slides on Type-level Computations](http://slick.typesafe.com/talks/scalaio2014/Type-Level_Computations.pdf)
* [Sam Halliday's Shapeless For Mortals](https://github.com/fommil/shapeless-for-mortals)  workshop material on GitHub (slides, videos, sources)
* [Miles Sabin's introductory talks](https://www.youtube.com/watch?v=GDbNxL8bqkY) Video on YouTube


### Misc. Resources

* [Why is the Aux technique required for type-level computations?](http://stackoverflow.com/questions/34544660/why-is-the-aux-technique-required-for-type-level-computations) on StackOverflow
* [Shapeless Wiki](https://github.com/milessabin/shapeless/wiki/Feature-overview:-shapeless-2.0.0)
* [Shapeless Examples](https://github.com/milessabin/shapeless/tree/master/examples/src/main/scala/shapeless/examples)
* [More on typeclasses in Eugene Yokota's “Herding Cats”](http://eed3si9n.com/herding-cats/)
