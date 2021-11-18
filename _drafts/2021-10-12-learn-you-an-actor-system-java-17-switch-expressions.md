---
title:  'Write You A Typed Actor Runtime For Great Good! (with Java 17, records, switch expressions and JBang)'
subtitle: ''
categories: [Java, Records, JBang]
date:   2021-11-18
---

The festive season is that period of the year when they tempt you to indulge in those dear sweet, sugary treats. 

<div style="float:right">
<img src="/assets/actor-3/shakespeare.png" alt="Whimsical sketch of Shakespeare" />
</div>

Personally, as an Italian, I do love me some [panettone][panettone]. And as much as I enjoy the bitter taste of Java coffee, I have been enjoying the sugar that has been introduced in the most recent versions. Indeed, I believe that Java 17 really hits the sweet spot, when it comes to treats. So what better time of the year to indulge in Java's sweet, sweet sugar than this December?

In the last couple of months [I have published a blog series][first-part] with my take on [Viktor Klang][klang]'s original [tiny Java][actorjava] and [Scala][minscalaactors] actor system, updated for Java 17.

Untyped actors in the style of [Akka Classic][akka-classic] used to be clunky to write in Java, because Java used to lack some key goodies: 

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


1. [JEP-330 and JBang](#jep-330-and-jbang)
1. [A Bit of Boilerplate](#a-bit-of-boilerplate)
2. [The Actor Model](#the-actor-model)
  - [Behaviors and Effects](#behaviors-and-effects)
  - [Example 1: A Hello World](#example-1-a-hello-world)
  - [Example 2: Ping Pong](#example-2-ping-pong)
  - [Example 3: A Vending Machine](#example-3-a-vending-machine)
3. [Implementing The Actor System](#implementing-the-actor-system)
4. [Wrapping Up](#wrapping-up)

This is the full listing of our typed actor runtime. You can also find it [at this repository][minjavaactors-typed]:

```java
package io.github.evacchi;

import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.function.Function;
import static java.lang.System.out;

public interface Actor {
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
            var addr = new AtomicRunnableAddress<T>() {
                // Our awesome little mailbox, free of blocking and evil
                final ConcurrentLinkedQueue<T> mbox = new ConcurrentLinkedQueue<>();
                Behavior<T> behavior = // Rebindable top of the mailbox, bootstrapped to identity
                        m -> (m instanceof Address self) ? Become(initial.apply((Address<T>) self)) : Stay();
                public void tell(T msg) { mbox.offer(msg); async(); }  // Enqueue the message onto the mailbox and try to schedule for execution
                // Switch ourselves off, and then see if we should be rescheduled for execution
                public void run() {
                    try { if (on.get() == 1) behavior = behavior.apply((T) mbox.poll()).apply(behavior); } finally { on.set(0); async(); }
                }
                // If there's something to process, and we're not already scheduled
                void async() {
                    if (!mbox.isEmpty() && on.compareAndSet(0, 1)) {
                        // Schedule to run on the Executor and back out on failure
                        try { executor.execute(this); } catch (Throwable t) { on.set(0); throw t; }
                    }
                }
                void init() { executor.execute(() -> { behavior = initial.apply(this); async();}); }
            };
            addr.init(); // Make the actor self aware by seeding its address to the initial behavior
            return addr;
        }
    }
}
```
{: style="font-size: small"}

In the following, we will go through each line and learn what it is doing, and try the code along the way.

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

For instance, consider the case of an HTTP request to initiate
a long computation. You may have an actor that receives a message and acknowledges the sender; 
then it may create the actor to process the request.
The long computation will start and run asynchronously in the background, while
the response will float back to the sender.

<div>
<img src="/assets/actor-3/diag-spawn.png" alt="Diagram of an HTTP request actor" />
</div>


At its core, an actor is just a routine paired with a message queue. 
But instead of being invoked directly, when a message is sent, 
the system submits that message to the queue of the receiver.
Then, at some point, the system will "wake up" that actor, taking
one message from the queue, and applying the routine to that message. 
The routine returns a description of the *next state* of the actor; 
i.e. the behavior that should be called when a new message is evaluated. 

Such a routine is called a *behavior*, and in code, the `Behavior` can be defined 
as a function that takes a message of some type, and it returns a *transition*
between states that we call an `Effect`:
 

```java
Function<Object, Effect> // Behavior
```


An `Effect` would be a way to describe a *transition* between two 
states of the actor. It can be represented as a *function* that takes the current `Behavior` and 
returns the next `Behavior`:

```java
Function<Behavior, Behavior> {}
```


Now, we want to our actor to be *typed*. Thus, a `Function<Object, Effect>`
is not the best way to express it; in fact, that would mean that an actor
may receive *any* type of objects; instead what we really want is to *tie*
an actor to a *specific* type `T`.

Thus, we want something like `Function<T, Effect>`; more specifically, 
we may define it as:

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
    out.println("Got msg " + msg);
    return Actor.Die();
} 
```

or written differently:

```java
Behavior<String> receiveThenDie = msg -> {
    out.println("Got msg " + msg);
    return Actor.Die();
};
```


### Example 1: A Hello World

> You can run the following example with:
> 
> ```sh
> j! https://gist.github.com/evacchi/40b19b0e116f3ce8f635787e0be79c95
> ```

In this example we will create an actor system, 
then spawn an actor that will process one message and then `Die`.
You will recognize the behavior `receiveThenDie` that we defined above.

```java
// create an actor runtime (an actor "system")
Actor.System actorSystem = new Actor.System(Executors.newCachedThreadPool());
// create an actor
Address<String> actor = actorSystem.actorOf(self -> (String msg) -> {
    out.println("self: " + self + " got msg " + msg);
    return Actor.Die();
});
```

The `actorOf` method returns an `Address` which is defined as follows:

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
self: io.github.evacchi.Actor$System$1@198a98aa got msg foo
Dropping msg [bar] due to severe case of death.
```

because the `"bar"` message was sent to a dead actor.

If we change the lambda to return `stay` instead:

```java
var actor = actorSystem.actorOf(self -> (String msg) -> {
    out.println("self: " + self + " got msg " + msg);
    return Actor.Stay();
});
```

then the output would read:

```java
self: io.github.evacchi.Actor$System$1@146b69a3 got msg foo
self: io.github.evacchi.Actor$System$1@146b69a3 got msg bar
```

`Stay` can be defined as such:

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
why `Stay` and `Die` could not just be constant fields:

```java
static Effect<T> Stay =  return current -> current; 
```

If you are familiar with the [untyped version][first-part] we, you'll remember that's how
we did it at that time. However, the key here is that little `<T>` up there. We have to use
a method to let the compiler infer the `T`.

You may also be wondering about that little `self` up there.
`self` is a self-reference to the actor. It serves the same purpose as `this` in a class.
Because the behavior  is written as a function, we need to "seed" a reference to
`this` into the function. But there is no `this` until
the actor is actually created by the runtime, so
we provide it in the closure, so that it may be filled lazily.

If this is not too clear, don't worry for now; we'll get to that later.


### Example 2: Ping Pong

> You can run the following example with:
> 
> ```sh
> j! https://gist.github.com/evacchi/97909cd131b8c358341b51463a40da04
> ```

In the previous example, the actor was consuming messages of *any* type (`Object`). 
Let us now make it more interesting, and only accept *some* types. 
This is where pattern matching and records come in handy.

<div style="float:right">
<img src="/assets/actor-3/pingpong.jpg" alt="A drawing of two ping-pong rackets" />
</div>

An actor routine usually matches a pattern against the incoming message.
In Java versions before 17, `switch` expression and pattern matching
support was experimental; in general, they are very recent additions 
([JEP-305][jep305] and [JEP-361][jep361] both delivered in JDK 14).

In a classic example actor example, one actor sends a "ping" to another;
the second replies with a "pong", and they go on back and forth.

In order to make this more interesting (and also not to loop indefinitely):
- one of the actors (the `ponger`) will receive `Ping` and reply with `Pong`; 
- it will also count 10 `Ping`s, then `Die`; 
- upon reaching 10 and before it `Die`s, the `pinger` 
  will also send a message (`DeadlyPong`) to the `ponger`
- the `pinger` receives `Ping` and replies with `Pong`
- when it receives a `DeadlyPong` it `Die`s.

First, we define the `Ping`, `Pong` messages, with the `Address` of the sender.


```java
sealed interface TPong { Address<Ping> sender(); }
```

```java
static record Ping(Address<TPong> sender) {}
static record Pong(Address<Ping> sender) implements TPong {}
```

We also define the `DeadlyPong` variant for `Pong`. This will be sent to the `ponger` 
when the `pinger` is about to shut off.  

```java
static record DeadlyPong(Address<Ping> sender) implements TPong {}
```

```java
void run() {
    var actorSystem = new Actor.System(Executors.newCachedThreadPool());
    var ponger = actorSystem.actorOf(StatefulPonger::new);
    var pinger = actorSystem.actorOf((Address<TPong> self) -> (TPong msg) -> pingerBehavior(self, msg));
    ponger.tell(new Ping(pinger));
}
    Effect<Ping> pongerBehavior(Address<Ping> self, Ping msg, int counter) {
        return switch (msg) {
            case Ping p && counter < 10 -> {
            out.println("ping! 👉");
            p.sender().tell(new Pong(self));
            yield Become(m -> pongerBehavior(self, m, counter + 1));
        }
        case Ping p -> {
            out.println("ping! 💀");
            p.sender().tell(new DeadlyPong(self));
            yield Die();
        }
    };
}
Effect<TPong> pingerBehavior(Address<TPong> self, TPong msg) {
    return switch (msg) {
        case Pong p -> {
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
Dropping msg [Ping[sender=io.github.evacchi.Actor$System$1@3b9c1475]] due to severe case of death.
```

the line:

```java
        case Ping p && counter < 10 -> {
```

is a "guarded pattern", that is, a pattern with a guard (a boolean expression). The expression may be 
on any variable or field that is in scope including the matched value (`p` in this case). This is still
a preview feature, so it requires the `--enable-preview` flag.

#### Ignored Messages

You may have noticed that all our switches have a `default` case. This is because 
any type of message is allowed; in fact, recall the `Address` interface and the signature of `tell()`:

```java
interface Address { Address tell(Object msg); }
```

If we send an object of a type that is not handled, it will just be silently swallowed:

```java
    ponger.tell("Hey hello"); // prints nothing
```

In order to keep these examples simple we just ignored the message;
you may want to add some logging to the `default` case, instead:

```java
    return switch (msg) {
        case ...
        default -> {
            out.println("Ignoring unknown message: " + msg);
            yield Stay;
        }
    };
```

#### Closures vs Classes

Notice how the traditional way to increase a counter is to create a closure with the value:

```java
void run() {
    ...
    var ponger = actorSystem.actorOf(self -> msg -> pongerBehavior(self, msg, 0));
    ...
}
Effect pongerBehavior(Address self, Ping msg, int counter) {
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
        return switch (msg) {
            case Ping p && counter < 10 -> {
                out.println("ping! 👉");
                p.sender().tell(new Pong(self));
                this.counter++;
                yield Stay();
            }
            case Ping p -> {
                out.println("ping! 💀");
                p.sender().tell(new DeadlyPong(self));
                yield Die();
            }
        };
    }
}
```


### Example 3: A Vending Machine

> You can run the following example with:
> 
> ```sh
> j! https://gist.github.com/evacchi/a46827cdcbdfefd93cf003459bb6fee1
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
record Coin(int amount){
    public Coin {
        if (amount < 1 && amount > 100)
            throw new AssertionError("1 <= amount < 100"); } 
}
record Choice(String product){}
```

And the behavior will be:

```java
Effect<Vend> initial(Vend message) {
    return switch(message) {
        case Coin c -> {
            out.println("Received first coin: " + c.amount);
            yield Become(m -> waitCoin(m, c.amount()));
        }
        default -> Stay(); // ignore message, stay in this state
    };
}
Effect<Vend> waitCoin(Vend message, int counter) {
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
Effect<Vend> vend(Vend message, int change) {
    return switch(message) {
        case Choice c -> {
            vendProduct(c.product());
            releaseChange(change);
            yield Become(this::initial);
        }
        default -> Stay(); // ignore message, stay in this state
    };
}


void vendProduct(String product) {
    out.println("VENDING: " + product);
}

void releaseChange(int change) {
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
record VendItem(String product){}
record ReleaseChange(int amount){}

Effect vend(Object message, int change) {
    return switch(message) {
        case Choice c -> {
            var vendActor = system.actorOf(self -> m -> vendProduct(m));
            vendActor.tell(new VendItem(c.product()))
            var releaseActor = system.actorOf(self -> m -> release(m));
            releaseActor.tell(new ReleaseChange(change))
            yield Become(this::initial);
        }
        default -> Stay; // ignore message, stay in this state
        ...
    }
}
Effect vendItem(Object message) {
    return switch (message) {
        case VendItem item -> {
            // ... vend the item ...
            yield Die // we no longer need to stay alive.
        }
    }
}
Effect release(Object message, int change) {
    return switch (message) {
        case ReleaseChange c -> {
            // ... release the amount ...
            yield Die // we no longer need to stay alive.
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
(`Actor.System`) carrying the `ExecutorService` for us. The method `Actor#apply` is defined in Scala as follows:

```scala
def apply(initial: Address => Behavior)(implicit e: Executor): Address
```

that allows to create an actor with `Actor( self => msg => ... )`; can be substituted by a method
`actorOf` of the `Actor.System` factory. 

```java
public interface Actor {
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
    public Address actorOf(Function<Address, Behavior> initial) {
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
  abstract class AtomicRunnableAddress implements Address, Runnable
    { AtomicInteger on = new AtomicInteger(0); }
  record System(ExecutorService executorService) {
        public Address actorOf(Function<Address, Behavior> initial) {
```

we write:

```java
  record System(ExecutorService executorService) {
        // Seeded by the self-reference that yields the initial behavior
        public Address actorOf(Function<Address, Behavior> initial) {
            abstract class AtomicRunnableAddress implements Address, Runnable
                { AtomicInteger on = new AtomicInteger(0); }
```

which makes `AtomicRunnableAddress` private to that method (which is all we need).

We will use the `AtomicInteger` to turn on and off the actor (note: we could probably use an `AtomicBoolean`, but 
we are just following the original code).

we now create our object:

```java
var addr = new AtomicRunnableAddress() {
    // the mailbox is just concurrent queue
    final ConcurrentLinkedQueue<Object> mbox = new ConcurrentLinkedQueue<>();
    // current behavior is a mutable field.   
    Behavior behavior = 
            // the initial behavior is to receive an address and then apply the initial behavior
            m -> { if (m instanceof Address self) return new Become(initial.apply(self)); else return Stay; };
  ...
};
addr.tell(addr); 
```

Here is the reason why our actors are created with this strange curried function:

```java
var actor = system.actorOf(self -> msg -> ...);
```

the signature for the initial behavior is really: `Function<Address, Behavior>` which "expands" to 

```java
Function<Address, Function<Object, Effect>>
```

or, to write it in a possibly more readable format:

```
Address -> Object -> Effect
// self -> msg -> ...
```

The reason why we write it this way is so that the `Function<Object, Effect>` (i.e. the `Behavior`) 
can reference `self`. As we saw in [Example 3: A Vending Machine](#example-3-a-vending-machine) this is often equivalent to writing a class
that takes an `Address` in its constructor. And that is because ["a closure is a poor man’s object; an object is a poor man’s closure”][closures-objects].

When the actor starts we send the address to itself:


```java
addr.tell(addr); 
```

Let us now take a look at the `tell()` method; at its core we may write it as:

```java
public void tell(Object msg) {
    // put message in the mailbox
    mb.offer(msg); 
    async(); 
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

In this long blog post we introduced actors as a running example to showcase some of the best
features of Java 17. One major feature has been left out, though; **sealed type hierarchies**.

If you recall, in [Behaviors and Effects](behaviors-and-effects) we noticed that we could 
use a specific type of message instead of `Object`:

```java
interface Message {}
interface Behavior {
    Effect receive(Message message);
}
```

But that would make our entire actor runtime tied to a built-in type
of message. Not a huge deal, but still a limitation.

Later, in the [Ping Pong Example](#example-2-ping-pong) we mentioned that we had
to always handle the default case because the signature of `tell()` is:


```java
interface Address { Address tell(Object msg); }
```

That would not be much better if the signature were:

```java
interface Address { Address tell(Message msg); }
```

For now, this was the best we could do. However, even 
[Akka now has evolved and it implements *Typed* Actors][typed-akka].
of an actor is actually typed, allowing us to match against **sealed hierarchies** of types.
In Typed Actors, the *Behavior* and the *Address* of an actor are *typed*; i.e., 
they specify the type of objects that they accept.

This allows you to match against **sealed hierarchies** of types,
and even avoid pattern matching entirely in some cases. 

Next time, we will see how to write a tiny chat server with its corresponding client, and run it through JBang; but in the final post, I will publish a follow-up to this blog post with an **updated, typed version**  of this tiny actor runtime, where, instead of:

```java
interface Behavior extends Function<Object, Effect> {}
interface Effect extends Function<Behavior, Behavior> {}
interface Address { Address tell(Object msg); }
```

we will have:

```java
interface Behavior<T> extends Function<T, Effect<T>> {}
interface Effect<T> extends Function<Behavior<T>, Behavior<T>> {}
interface Address<T> { Address<T> tell(T msg); }
```

and we will learn what changes will be needed to adjust for that change.
In the meantime, you can try it for yourself!


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
[wiki-actor]: https://en.wikipedia.org/wiki/Actor_model#Fundamental_concepts
[closures-objects]: http://people.csail.mit.edu/gregs/ll1-discuss-archive-html/msg03277.html
