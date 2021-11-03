---
title:  'Write You A Chat For Great Good! (with Java 17, actors, and JBang!)'
subtitle: ''
categories: [Java, Records, JBang]
date:   2021-10-12
---

Hello! Welcome back to this blog series! In this blog post we will use our [minimalistic actor runtime][minjavactors] to realize a tiny chat system.

For simplicity, I have decided to use Java's blocking API. Using a blocking API on an actor thread is generally a no-no. You should never block within the body of an actor. Unless you run on a [Loom-enabled JDK][loom], where threads are virtual, light-weight but still pre-emptible.

## Boilerplate

Some methods in the `Socket` API will throw `IOException`s, which is a checked exception.
Checked exception do not play nicely with lambdas, unless you define a method signature 
that accepts them.

### IOBehavior

Let's define a `IOBehavior` interface:

```java
interface IOBehavior {
    Actor.Effect apply(Object msg) throws IOException;
}
```

It is basically *identical* to `Function<Object, Effect>` except it declares a `throws IOException` clause.
We can now create plain `Behavior`s as our actor runtimes expect by defining the companion static method: 

```java
interface IOBehavior {
    ...
    static Actor.Behavior of(IOBehavior behavior) {
        return msg -> {
            try { return behavior.apply(msg); } 
            catch (IOException e) { throw new UncheckedIOException(e); }};
    }
}
```

This way we can create an actor that catches and rethrows `IOException`s as such

```java
  var myActor = sys.actorOf(self -> IOBehavior(msg -> /* IO-throwing body */))
```

### Polling

The blocking API will require polling. Normally we would implement this as a `while` loop 
of the form:

```java
while (true) {
    // do something here
}
```

But this is a rookie mistake. You should always put a tiny sleep in your loops
so that they won't spin and spike the CPU to 100%.

```java
while (true) {
    // do something here
    Thread.sleep(someSmallAmount);
}
```


We will simulate this by creating a `Poll` message,
that the actor will send to itself. The type of the message
is not important; in fact we may match against the message directly:

```java
Object Poll = new Object();
// then inside the actor body:
if (msg == Poll) {
    // read from the socket
}
```

In order to simulate the `sleep`, we will use a scheduled executor
and use that to schedule the `Poll` message to `self`, as such:

```java
scheduler.schedule(() -> self.tell(Poll), 1, SECONDS);
```


## Chat Server

Our chat server will receive incoming connections


The "core" of the server is really an actor I have called the `clientManager`.

The `clientManager` keeps track of all the connected clients.
- every time a new client connects:
   1. it creates an actor that handles the socket input `clientInput`
   2. it also creates a companion `clientOutput` which will be responsible for publishing the messages to the socket output 
- the `clientManager` also receives messages from all `clientInput` actors, so that every time that a user
  writes a message, they can be broadcasted to all other clients.

```java
static Behavior clientManager(Address self) {
    var clients = new ArrayList<Address>();
    return IOBehavior.of(msg -> {
        switch (msg) {
            // create a connection handler and a message handler for that client
            case CreateClient n -> {
                var socket = n.socket();
                out.println("accepts : " + socket.getRemoteSocketAddress());

                var in = new Scanner(socket.getInputStream());
                var clientSocketHandler =
                        io.actorOf(me -> clientInput(me, self, in));

                var writer = new PrintWriter(socket.getOutputStream(), true);
                var clientMessageHandler =
                        sys.actorOf(client -> clientOutput(writer));

                clients.add(clientMessageHandler);
                return Stay;
            }
            // broadcast the message to all connected clients
            case Message m -> {
                clients.forEach(c -> c.tell(m));
                return Stay;
            }
            // ignore all other messages
            default -> {
                return Stay;
            }
        }
    });
}
```

This is more or less the most complicated actor you'll see.

The input handler `clientInput` gets initialized with the `clientManager` address, 
and a `BufferedReader` that wraps the socket's input stream. This is just a convenient
way to read line-wise.

The first line of the `clientInput` method schedules to send `Poll` to `self`. 
This is to simulate an infinite loop; however, by scheduling a message instead
of actually writing a loop, we release each time the underlying thread for re-scheduling.

Please notice that we `schedule` the message instead of just sending with `self.tell(Poll)`,
because otherwise we would easily spike the CPU.

```java
    static Behavior clientInput(Address self, Address clientManager, BufferedReader in) {
        // schedule a message to self
        scheduler.schedule(() -> self.tell(Poll), 1, SECONDS);

        return IOBehavior.of(msg -> {
            // ignore non-Poll messages
            if (msg != Poll) return Stay;
            if (in.ready()) {
                var input = in.readLine();
                out.println(input);
                clientManager.tell(new Message(input));
            }

            // "stay" in the same state, ensuring that the initializer is re-evaluated
            return Become(clientInput(self, clientManager, in));
        });
    }
```

```
static Behavior clientOutput(PrintWriter writer) {
    return msg -> {
        if (msg instanceof Message m) writer.println(m.text());
        return Stay;
    };
}
```


[minjavactors]: blah
[loom]: loom