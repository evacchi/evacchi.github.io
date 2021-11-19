---
title:  'Type You An Actor Runtime For Greater Good! (with Java 17, records, switch expressions and JBang)'
subtitle: ''
categories: [Java, Records, JBang]
date:   2021-11-18
---

The festive season is that period of the year when they tempt you to indulge in those dear sweet, sugary treats. 

<div style="float:right">
<img src="/assets/actor-3/shakespeare.png" alt="Whimsical sketch of Shakespeare" />
</div>

Personally, as an Italian, I do love me some [panettone][panettone]. And as much as I enjoy the bitter taste of Java coffee, I have been enjoying the sugar that has been introduced in the most recent versions. Indeed, I believe that Java 17 really hits the sweet spot, when it comes to treats. So what better time of the year to indulge in Java's sweet, sweet sugar than this December?

In the last couple of months [I published a blog series][first-part] with my take on [Viktor Klang][klang]'s original [tiny Java][actorjava] and [Scala][minscalaactors] actor system, updated for Java 17.

*Untyped* actors in the style of [Akka Classic][akka-classic] used to be clunky to write in Java, because Java used to lack some key goodies: 

1. a concise way to express messages; but now we have **records**
2. a tidy syntax to match against the types of the incoming messages; but now we have **switch expressions** and **pattern matching**  

Another key addition is **sealed type hiearchies**. If you are able to express the upper bound of your type hierarchy, and such a type hierarchy is "sealed", then the compiler will tell you if you are missing a `case` in a `switch` expression (*exhaustiveness check*).

For instance:

```java
sealed interface A {
    record X() implements A{} 
    record Y() implements A{}

    static void f(A a) {
        switch (a) {
            case X x -> {}
        }
    }
}
```

If you put this in `A.java` and run it with `java --enable-preview --source 17 A.java` you'll read:

```
A.java:6: error: the switch statement does not cover all possible input values
        switch (a) {
        ^
```

In my [previous blog posts][first-part] I have detailed how to develop an actor runtime for *untyped actors*; that is, actors that can accept *any kind of message*. In this part we are rewriting that actor runtime from scratch and implement a **typed actor runtime**, and we will see how sealed type hierarchies can improve the code we write!

1. [The Actor Model](#the-actor-model)
  - [Behaviors and Effects](#behaviors-and-effects)
  - [Example 1: A Hello World](#example-1-a-hello-world)
  - [Example 2: Ping Pong](#example-2-ping-pong)
  - [Example 3: A Vending Machine](#example-3-a-vending-machine)
2. [Implementing The Actor System](#implementing-the-actor-system)
3. [Wrapping Up](#wrapping-up)

This is the full listing of our typed actor runtime. You can also find it [at this repository][minjavaactors-typed]:

```java
package io.github.evacchi;

import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.function.Function;
import static java.lang.System.out;

public interface TypedActor {
    interface Effect<T> extends Function<Behavior<T>, Behavior<T>> {}
    interface Behavior<T> extends Function<T, Effect<T>> {}
    interface Address<T> { void tell(T msg); }
    static <T> Effect<T> Become(Behavior<T> next) { return current -> next; }
    static <T> Effect<T> Stay() { return current -> current; }
    static <T> Effect<T> Die() { return Become(msg -> { out.println("Dropping msg [" + msg + "] due to severe case of death."); return Stay(); }); }
    record System(Executor executor) {
        public <T> Address<T> actorOf(Function<Address<T>, Behavior<T>> initial) {
            abstract class AtomicRunnableAddress<T> implements Address<T>, Runnable
                { AtomicInteger on = new AtomicInteger(0); }
            return new AtomicRunnableAddress<T>() {
                // Our awesome little mailbox, free of blocking and evil
                final ConcurrentLinkedQueue<T> mbox = new ConcurrentLinkedQueue<>();
                Behavior<T> behavior = initial.apply(this);
                public void tell(T msg) { mbox.offer(msg); async(); }  // Enqueue the message onto the mailbox and try to schedule for execution
                // Switch ourselves off, and then see if we should be rescheduled for execution
                public void run() {
                    try { if (on.get() == 1) { T m = (T) mbox.poll(); if (m != null) behavior = behavior.apply(m).apply(behavior); }
                    } finally { on.set(0); async(); }
                }
                // If there's something to process, and we're not already scheduled
                void async() {
                    if (!mbox.isEmpty() && on.compareAndSet(0, 1)) {
                        // Schedule to run on the Executor and back out on failure
                        try { executor.execute(this); } catch (Throwable t) { on.set(0); throw t; }
                    }
                }
            };
        }
    }
}
```
{: style="font-size: small"}

But before we get to that, let us learn more about what it does.

## The Actor Model

The Actor Model is a *concurrency model* where the unit of execution is called *an actor*. An actor *receives messages*.
In response to a message, an actor may (e.g. cf. [Wikipedia][wiki-actor]):

- send a message to another actor
- create new actors
- transition to a new state, with a different *behavior*, to handle the next message 

### Behaviors and Effects

The *behavior* of an actor is just a *function* that, applied to a message,
returns another behavior.

Actors usually encapsulate *state*; thus, as a side-effect, the *behavior*
function usually updates the state of the actor; it may send other messages
to other actors, and creates new actors to handle new state.

For instance, you will have noticed how most web platforms allow
you to export the content you have created; but most of them
will start a background process and will notify you later when the archive 
is ready; for instance, by sending a link to your e-mail address. 

When the service receives your "export" request, an actor may be responsible
for acknowledging your request immediately; but it may spawn another actor 
to process the request in the background.

<div>
<img src="/assets/actor-3/diag-spawn.png" alt="Diagram of an HTTP request actor" />
</div>


At its core, an actor is just a routine paired with a message queue. 
But instead of evaluating the routine as soon as a message is sent, 
the system submits a message to the queue of the receiver.
Then, at some point, the system "wakes up" that actor: it takes
one message from the queue, and it applies the routine to that message. 

The routine returns a description of the *next state* of the actor; 
i.e. the routine that should be executed when a new message is evaluated. 

Such a routine is called a *behavior*, and in code, the `Behavior` can be defined 
as a function that takes a message of some type, and it returns a *transition*
between states that we call an `Effect`:

```
Behavior : T ⟶ Effect
```

where `T` is some known type.


An `Effect` describes a *transition* between two 
states of the actor. It can be represented as a *function* 
that takes the current `Behavior` and returns the next `Behavior`:

```
Effect : Behavior ⟶ Behavior
```

In code, we may write them as:

```java
interface Behavior<T> extends Function<T, Effect<T>> {}
interface Effect<T> extends Function<Behavior<T>, Behavior<T>> {}
```

The most basic `Effect`s (state transitions) are `Stay` and `Die`:

- `Stay` means no behavioral change
- `Die` will effectively turn off the actor, making it inactive. 

For instance, this is a valid behavior for an actor that
starts, then waits for one message, then it dies: i.e., 
it will drop and ignore all subsequent messages and/or
the system may decide to collect it and throw it away.

```java
Effect<String> receiveThenDie(String msg) {
    out.println("Got msg: '" + msg + "'; length: " + msg.length());
    return TypedActor.Die();
} 
```

or written differently:

```java
Behavior<String> receiveThenDie = msg -> {
    out.println("Got msg '" + msg + "'; length: " + msg.length());
    return TypedActor.Die();
};
```


### Example 1: A Hello World

> You can run the following example with:
> 
> ```sh
> j! https://github.com/evacchi/min-java-actors/blob/fix-typed/src/main/java/io/github/evacchi/typed/examples/HelloWorld.java
> ```

In this example we will create an actor system, 
then spawn an actor that will process one message and then `Die`.
You will recognize the behavior `receiveThenDie` that we defined above.

```java
// create an actor runtime (an actor "system")
var actorSystem = new Actor.System(Executors.newCachedThreadPool());
// create an actor
Address<String> actor = actorSystem.actorOf(self -> msg -> {
    out.println("self: " + self +"; got msg: '" + msg + "'; length: " + msg.length());
    return Actor.Die();
});
```

The `actorOf` method returns an `Address<T>` which is defined as follows:

```java
interface Address<T> { Address<T> tell(T msg); }
```

allowing us to write:

```java
actor.tell("foo");
actor.tell("bar");
```

or just:

```java
actor.tell("foo").tell("bar");
```

which, when executed, prints the following:

```
self: io.github.evacchi.TypedActor$System$1@24a95c2e; got msg 'foo'; length, 3
Dropping msg [foo] due to severe case of death.
```

because the `"bar"` message was sent to a dead actor.

If we change the lambda to return `stay` instead:

```java
Address<String> actor = actorSystem.actorOf(self -> msg -> {
    out.println("self: " + self +"; got msg: '" + msg + "'; length: " + msg.length());
    return Stay();
});
```

then the output would read:

```
self: io.github.evacchi.TypedActor$System$1@7519a17c; got msg: 'foo'; length: 3
self: io.github.evacchi.TypedActor$System$1@7519a17c; got msg: 'bar'; length: 3
```

You may define `Stay` as:

```java
static <T> Effect<T> Stay() { return current -> current; }
```

that is, a transition from the current behavior to the current behavior (i.e. it *stays*
in the same state.)

`Die` is defined as:

```java
static <T> Effect<T> Die() { 
    return Become(msg -> {
        out.println("Dropping msg [" + msg + "] due to severe case of death."); 
        return Stay(); 
    }); 
}
```

where `Become` is:

```java
static Effect<T> Become(Behavior<T> next) { return current -> next; }
```

i.e. `Become` is a method, that, given a `Behavior` returns an *effect*. 
And that effect is taking the `current` behavior and returning the `next` one.

Thus, `Die` is just an effect that takes the `prev` behavior and returns the behavior to drop
all messages, and then `Stay`s in that state.

With the exception of `Become`, which takes a parameter (`next`), you may be wondering
why `Stay` and `Die` are not constant fields:

```java
static Effect<T> Stay =  return current -> current; 
```

You may be wondering what `self` is.
`self` is a self-reference to the actor. It serves the same purpose as `this` in a class.
Because the behavior  is written as a function, we need to "seed" a reference to
`this` into the function. But there is no `this` until
the actor is actually created by the runtime, so
we provide it in the closure, so that it may be filled lazily.

If this is not too clear, don't worry for now; we'll get to that later.

#### Use of Types

If you are familiar with the [untyped version][first-part], you'll remember that's how
we did it at that time. However, the key here is that little `<T>` up there. We have to use
a method to let the compiler infer the `T`.

Notice how that little `T` in the definition of `Behavior`
makes a huge difference: [an *untyped* actor system][first-part] would be defined as:

```java
interface Behavior extends Function<Object, Effect> {}
interface Effect extends Function<Behavior, Behavior> {}
interface Address { Address tell(Object message); }
```

So the actor itself would be written:

```java
Address actor = actorSystem.actorOf(self -> msg -> {
    out.println("self: " + self +"; got msg: '" + msg + "'; length: " + msg.length());
    return Actor.Die();
};
```

This would not compile, because `msg` has now type `Object` and `length()` is
no longer a known method:

```
  error: cannot find symbol
            out.println("self: " + self +"; got msg '" + msg + "'; length: " + msg.length());
                                                                                  ^
  symbol:   method length()
  location: variable msg of type Object
```

unless you check its type:

```java
Behavior receiveThenDie = msg -> {
    if (msg instanceof String) {
        var s = (String) msg;
        out.println("self: " + self +"; got msg: '" + s + "'; length: " + s.length());
    } else {
        // handle the non-String message
    }
    return Actor.Die();
};
```

or, more concisely:

```java
    if (msg instanceof String s) {
        out.println("self: " + self +"; got msg: '" + s + "'; length: " + s.length());
    ...
```

The concise version uses Pattern Matching for `instanceof`, delivered in JDK 14 ([JEP-305][jep305]).
It allows you to check against a type, and get a typed variable out of it if the check passes, all in one line.


### Example 2: Ping Pong

> You can run the following example with:
> 
> ```sh
> j! https://github.com/evacchi/min-java-actors/blob/fix-typed/src/main/java/io/github/evacchi/typed/examples/PingPong.java
> ```

An actor routine usually accepts more than one type of messages. It is therefore
useful to match against all the accepted subtypes.

This is where [`switch` expressions][jep361],
[records][jep395] and [sealed types][jep409] are useful.

<div style="float:right">
<img src="/assets/actor-3/pingpong.jpg" alt="A drawing of two ping-pong rackets" />
</div>

In a classic actor example, one actor sends a "ping" to another;
the second replies with a "pong", and they go on back and forth.

In order to make this more interesting (and also not to loop indefinitely):
- one of the actors (the `ponger`) will receive `Ping` and reply with `Pong`; 
- it will also count 10 `Ping`s, then `Die`; 
- upon reaching 10 and before it `Die`s, the `pinger` 
  will also send a message (`DeadlyPong`) to the `ponger`
- the `pinger` receives `Ping` and replies with `Pong`
- when it receives a `DeadlyPong` it `Die`s.

In the [untyped version][first-part] of this program, the messages do not need to be 
defined in a hierarchy. But in the typed version, a tiny hierarchy of sealed records
will make the code shorter.

There is only one type of `Ping`:

```java
record Ping(Address<Pong> sender) {}
```

the *sender* of such messages is able to receive `Pong`s. Now, we said that there
are two types of `Pong`s:

```java
record SimplePong(Address<Ping> sender) 
record DeadlyPong(Address<Ping> sender) 
```

And they are in the same hierarchy, so let us define the interface `Pong`:

```java
interface Pong {}
record SimplePong(Address<Ping> sender) implements Pong {}
record DeadlyPong(Address<Ping> sender) implements Pong {}
```

Both messages are *sent* by the actor that is able to *receive* `Ping`s.

```java
void static void main(String... args) {
    var actorSystem = new TypedActor.System(Executors.newCachedThreadPool());
    Address<Ping> ponger = actorSystem.actorOf(self -> msg -> pongerBehavior(self, msg, 0));
    Address<Pong> pinger = actorSystem.actorOf(self -> msg -> pingerBehavior(self, msg));
    ponger.tell(new Ping(pinger));
}
static Effect<Ping> pongerBehavior(Address<Ping> self, Ping msg, int counter) {
    if (counter < 10) {
        out.println("ping! 👉");
        msg.sender().tell(new SimplePong(self));
        return Become(m -> pongerBehavior(self, m, counter + 1));
    } else {
        out.println("ping! 💀");
        msg.sender().tell(new DeadlyPong(self));
        return Die();
    }
}
static Effect<Pong> pingerBehavior(Address<Pong> self, Pong msg) {
    return switch (msg) {
        case SimplePong p -> {
            out.println("pong! 👈");
            p.sender().tell(new Ping(self));
            yield Stay();
        }
        case DeadlyPong p -> {
            out.println("pong! 😵");
            p.sender().tell(new Ping(self));
            yield Die();
        }
    };
}
```

The prints the following:

```
ping! 👉
pong! 👈
ping! 👉
pong! 👈
ping! 👉
pong! 👈
ping! 👉
pong! 👈
ping! 👉
pong! 👈
ping! 👉
pong! 👈
ping! 👉
pong! 👈
ping! 👉
pong! 👈
ping! 👉
pong! 👈
ping! 👉
pong! 👈
ping! 💀
pong! 😵
Dropping msg [Ping[sender=io.github.evacchi.TypedActor$System$1@21198648]] due to severe case of death.
```

#### Use of Types

If you are familiar with the [untyped version][first-part], you'll remember that the `pingerBehavior` needed
a `default` clause: that's because, as we learned previously, the signature for `Behavior` was
`Function<Object,Effect>`: we had to handle and *ignore* messages that were not `Pong`s!

Because the signature is now effectively `Function<Pong, Effect<Pong>>`
the compiler *knows* that only messages from the `Pong` hierarchy may be received; 
thus, we don't need to add a `default` clause!

Likewise `pongerBehavior` defined a `switch` expression too. In the typed version, 
however, the `switch` is made entirely redundant:

```java
static Effect<Ping> pongerBehavior(Address<Ping> self, Ping msg, int counter) {
    return switch (msg) {
        case Ping p && counter < 10 -> {
            out.println("ping! 👉");
            p.sender().tell(new SimplePong(self));
            yield Become(m -> pongerBehavior(self, m, counter + 1));
        }
        case Ping p -> {
            out.println("ping! 💀");
            p.sender().tell(new DeadlyPong(self));
            yield Die();
        }
    };
}
```

because the signature is `Function<Ping, Effect<Ping>>` and we don't need a `default` clause,
both `case` clauses are matching against a `Ping`; thus, the entire switch
is effectively equivalent to a simple `if`/`else`! 

#### Closures vs Classes

Notice how the traditional way to increase a counter is to create a closure with the value:

```java
void static void main(String... args) {
    ...
    var ponger = actorSystem.actorOf(self -> msg -> pongerBehavior(self, msg, 0));
    ...
}
static Effect pongerBehavior(Address self, Ping msg, int counter) {
    return switch (msg) {
        case Ping p && counter < 10 -> {
            ...
            yield Become(m -> pongerBehavior(self, m, counter + 1));
        }
        ...
    }
}
```

However, a similar effect could be achieved with mutable state; this is perfectly acceptable, 
because the state of an actor is guaranteed to execute in a thread-safe environment. 
In this case we could have written:

```java
void run() {
    ...
    var ponger = actorSystem.actorOf(StatefulPonger::new);
    ...
}
class StatefulPonger implements Behavior<Ping> {
    Address<Ping> self; int counter = 0;
    StatefulPonger(Address<Ping> self) { this.self = self; }
    public Effect<Ping> apply(Ping msg) {
        if (counter < 10) {
            out.println("ping! 👉");
            msg.sender().tell(new SimplePong(self));
            this.counter++;
            return Stay();
        } else {
            out.println("ping! 💀");
            msg.sender().tell(new DeadlyPong(self));
            return Die();
        }
    }
}
```


### Example 3: A Vending Machine

> You can run the following example with:
> 
> ```sh
> j! https://github.com/evacchi/min-java-actors/blob/fix-typed/src/main/java/io/github/evacchi/typed/examples/VendingMachine.java
> ```

<div style="float:right">
<img src="/assets/actor-3/vending.jpg" alt="A drawing of a vending machine" />
</div>

In the previous example, we saw how we can use actors to maintain mutable state. 

But we also saw that when an actor receives a message it returns an `Effect`; that is, a *transition*
to a different behavior. So far we only only used two built-in effects: `Stay` and `Die`. 
But we can use the `Become` mechanism to change the behavior of our actors upon receiving a message. In other words, actors can be used to implement *state machines*.  

A classic example of a state machine is the *vending machine*: a vending machine awaits for a specific amount of coins before it allows to pick a choice. Let us implement a simple vending machine using our actor runtime.

Suppose that you are developing a vending machine that waits for you 
to insert an amount of 100 before you can pick a choice. Then you can pick your choice and your item will be retrieved. For simplicity, we assume that each coin has a value between 1 and 100.


<div>
<img src="/assets/actor-3/diag-vend.png" alt="Diagram of the state machine" />
</div>


The messages will be:

```java
sealed interface Vend {}
static record Coin(int amount) implements Vend {
    public Coin {
        if (amount < 1 && amount > 100)
            throw new AssertionError("1 <= amount < 100");
    }
}
static record Choice(String product) implements Vend {}
```

And the behavior will be:

```java
static Effect<Vend> initial(Vend message) {
    return switch(message) {
        case Coin c -> {
            out.println("Received first coin: " + c.amount);
            yield Become(m -> waitCoin(m, c.amount()));
        }
        default -> Stay(); // ignore message, stay in this state
    };
}
static Effect<Vend> waitCoin(Vend message, int counter) {
    return switch(message) {
        case Coin c && counter + c.amount() < 100 -> {
            var count = counter + c.amount();
            out.println("Received coin: " + count + " of 100");
            yield Become(m -> waitCoin(m, count));
        }
        case Coin c -> {
            var count = counter + c.amount();
            out.println("Received last coin: " + count + " of 100");
            var change = counter + c.amount() - 100;
            yield Become(m -> vend(m, change));
        }
        default -> Stay(); // ignore message, stay in this state
    };
}
static  Effect<Vend> vend(Vend message, int change) {
    return switch(message) {
        case Choice c -> {
            vendProduct(c.product());
            releaseChange(change);
            yield Become(m -> initial(m));
        }
        default -> Stay(); // ignore message, stay in this state
    };
}

static void vendProduct(String product) {
    out.println("VENDING: " + product);
}

static  void releaseChange(int change) {
    out.println("CHANGE: " + change);
}
```

This actor may be created by an actor system by passing the initial state:

```java
var actorSystem = new Actor.System(Executors.newCachedThreadPool());
var vendingMachine = actorSystem.actorOf((Address<Vend> self) -> (Vend msg) -> initial(msg));
```

and then started with:

```java
vendingMachine.tell(new Coin(50))
              .tell(new Coin(40))
              .tell(new Coin(30))
              .tell(new Choice("Chocolate"));
```

It will print the following:

```
Received first coin: 50
Received coin: 90 of 100
Received last coin: 120 of 100
VENDING: Chocolate
CHANGE: 20
```


If an action is costly (e.g. blocking), an actor may decide to spawn another actor
to serve that action. For instance:

```java
record VendItem(String product) implements Vend {}
record ReleaseChange(int amount) implements Vend {}

static Effect<Vend> vend(Vend message, int change) {
    return switch(message) {
        case Choice c -> {
            var vendActor = system.actorOf(self -> m -> vendProduct(m));
            vendActor.tell(new VendItem(c.product()))
            var releaseActor = system.actorOf(self -> m -> release(m));
            releaseActor.tell(new ReleaseChange(change))
            yield Become(m -> initial(m));
        }
        default -> Stay(); // ignore message, stay in this state
        ...
    }
}
Effect<Vend> vendItem(Vend message) {
    return switch (message) {
        case VendItem item -> {
            // ... vend the item ...
            yield Die() // we no longer need to stay alive.
        }
    }
}
Effect<Vend> release(Vend message, int change) {
    return switch (message) {
        case ReleaseChange c -> {
            // ... release the amount ...
            yield Die() // we no longer need to stay alive.
        }
    }
}
```
## Implementing The Actor System

We are now ready to implement the actor system and execution environment.
[The original `Scala` snippet][minscalaactors] does away with an `ActorSystem` object, because it leverages
the `implicit` feature to inject an `Executor` automatically:

```scala
implicit val e: java.util.concurrent.Executor = java.util.concurrent.Executors.newCachedThreadPool
// implicitly takes e as an Executor
val actor = Actor( self => msg => { println("self: " + self + " got msg " + msg); Die } ) 
```


Although that is convenient we can achieve just the same result by creating a factory object
(`TypedActor.System`) carrying the `ExecutorService` for us. The method `TypedActor#apply` is defined in Scala as follows:

```scala
def apply(initial: Address => Behavior)(implicit e: Executor): Address
```

that allows to create an actor with `TypedActor( self => msg => ... )`; can be substituted by a method
`actorOf` of the `TypedActor.System` factory. 

```java
public interface TypedActor {
    class System {
        Executor executor;
        public System(Executor executor) { this.executor = executor; }
        public Address actorOf(Function<Address, Behavior> initial) {
           // ... references the executor ...
        }
    }
} 
```

However, in order to keep the number of lines down, we can abuse the `record` construct so that we 
don't have to write an explicit constructor:

```java
record System(Executor executor) {
    public <T> Address<T> actorOf(Function<Address<T>, Behavior<T>> initial) {
        // ... references the executor ...
    }
}
```


The original Scala code defines a private abstract class `AtomicRunnableAddress` .
We picked `interface Actor` as the outer container. However `interface`s (currently) do not allow
private class members. But the purpose of `AtomicRunnableAddress` is really to create an 
anonymous class that implements two interfaces in the body of the `apply()` method:

```scala
private abstract class AtomicRunnableAddress extends Address with Runnable { val on = new AtomicInteger(0) }
def apply(initial: Address => Behavior)(implicit e: Executor): Address =              
  new AtomicRunnableAddress { // Memory visibility of "behavior" is guarded by "on" using volatile piggybacking
```

We can use another under-used feature of Java: local classes; i.e. a class that is local to the *body* of a method.
So instead of:

```java
abstract class AtomicRunnableAddress<T> implements Address<T>, Runnable
    { AtomicInteger on = new AtomicInteger(0); }
record System(ExecutorService executorService) {
    public <T> Address<T> actorOf(Function<Address<T>, Behavior<T>> initial) {
```

we write:

```java
record System(Executor executor) {
    public <T> Address<T> actorOf(Function<Address<T>, Behavior<T>> initial) {
        abstract class AtomicRunnableAddress<T> implements Address<T>, Runnable
            { AtomicInteger on = new AtomicInteger(0); }
```

which makes `AtomicRunnableAddress` private to that method (which is all we need).

We will use the `AtomicInteger` to turn on and off the actor (note: we could probably use an `AtomicBoolean`, but 
we are just following the original code).

we now create our object:

```java
var addr = new AtomicRunnableAddress() {
    // the mailbox is just a concurrent queue
    final ConcurrentLinkedQueue<T> mbox = new ConcurrentLinkedQueue<>();
    // current behavior is a mutable field.  
    // the initial behavior applies the `initial` function to `this`, seeding `self` reference to the initial behavior
    Behavior behavior = initial.apply(this);
  ...
};
addr.tell(addr); 
```

Here is the reason why our actors are created with this strange curried function:

```java
var actor = system.actorOf(self -> msg -> ...);
```

the signature for the initial behavior is really: `Function<Address<T>, Behavior<T>>` which "expands" to 

```java
Function<Address<T>, Function<T, Effect<T>>>
```

or, to write it in a possibly more readable format:

```
Address<T> -> T -> Effect<T>
// self -> msg -> ...
```

The reason why we write it this way is so that the `Function<T, Effect>` (i.e. the `Behavior`) 
can reference `self`. As we saw in [Example 3: A Vending Machine](#example-3-a-vending-machine) this is often equivalent to writing a class
that takes an `Address<T>` in its constructor. And that is because ["a closure is a poor man’s object; an object is a poor man’s closure”][closures-objects].

When the actor starts we initialize the `Behavior<T>` to a reference to `this`

```java
Behavior behavior = initial.apply(this);
```

Let us now take a look at the `tell()` method; at its core we may write it as:

```java
public Address<T> tell(Object msg) {
    // put message in the mailbox
    mb.offer(msg); 
    async(); 
    return this;
}
```

The async method verifies that the mbox contains an element and schedules the actor 
for execution on the `Executor`.


```java
void async() {
    // if the mbox is non-empty and the actor is active
    if (!mb.isEmpty() && on.getAndSet(1) == 0)) {
        // schedule to run on the Executor 
        try { executor.execute(this); }
        // in case of error deactivate the actor and rethrow the exception
        catch (Throwable t) { on.set(0); throw t; }
    }
}
```

In order to be schedulable, the actor must be a `Runnable`, so here is the `run()` method:

```java
    public void run() {
        try {
          // if it is active 
          if (on.get() == 1) 
            behavior = 
              behavior.apply(mbox.poll()) // apply the behavior to the top of the mailbox
                .apply(behavior); // as a result an Effect is returned: 
                                  // apply it to the current behavior
                                  // it returns the next behavior (which overwrites the old in the assignment) 
        } finally { on.set(0); async(); } // deactivate and resume if necessary
    }
```


## Wrapping Up

!!!! TODO: add notes on initialization!!





## Acknowledgements

- [Viktor Klang][klang] published the original snippet and reviewed a draft of this post
- [Max Andersen](https://xam.dk) is the author of [JBang][jbang] and reviewed a draft of this post
- I thook some inspiration from [Bob Nystrom's Crafting Interpreters](https://craftinginterpreters.com/) for the writing style of this tutorial
- [Miran Lipovaca for the title of his Haskell book](http://learnyouahaskell.com/)

[panettone]: https://en.wikipedia.org/wiki/Panettone
[klang]: https://viktorklang.com/
[actorjava]: https://gist.github.com/viktorklang/2557678
[minscalaactors]: https://gist.github.com/viktorklang/2362563
[akka-classic]: https://doc.akka.io/docs/akka/current/actors.html
[typed-akka]: https://doc.akka.io/docs/akka/current/typed/actors.html#first-example
[first-part]: https://evacchi.github.io/java/records/jbang/2021/10/12/learn-you-an-actor-system-java-17-switch-expressions.html
[minjavaactors]: https://github.com/evacchi/min-java-actors/blob/main/src/main/java/io/github/evacchi/Actor.java
[minjavaactors-typed]: https://gist.github.com/evacchi/b601498c3d8d483e3350f00b4c8aabf5
[jdk17-released]: https://inside.java/2021/09/14/the-arrival-of-java17/
[jbang]: https://www.jbang.dev
[jep305]: https://openjdk.java.net/jeps/305
[jep330]: https://openjdk.java.net/jeps/330
[jep361]: https://openjdk.java.net/jeps/361
[jep395]: https://openjdk.java.net/jeps/395
[jep409]: https://openjdk.java.net/jeps/409
[wiki-actor]: https://en.wikipedia.org/wiki/Actor_model#Fundamental_concepts
[closures-objects]: http://people.csail.mit.edu/gregs/ll1-discuss-archive-html/msg03277.html
