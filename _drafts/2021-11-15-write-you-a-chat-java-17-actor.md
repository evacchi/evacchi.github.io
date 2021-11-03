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


The `clientOutput` actor is extremely simple: it will just receive a message, and serialize
it to the given `PrintWriter` instance.

Because we are serializing to JSON, we can assume newlines are not part of the serialized message,
so they are safe both to write and to read.

```java
static Behavior clientOutput(PrintWriter writer) {
    return IOBehavior.of(msg -> {
        if (msg instanceof Message m) writer.println(Mapper.writeValueAsString(m));
        return Stay;
    });
}
```

The input handler `clientInput` receives `Poll` messages at a given delay,
and reschedules the same message after processing input. It will discard any message that is not a `Poll`.

When it receives a `Poll` it checks the given `BufferedReader` if there is input,  and then read a line.

Each line contains a serialized message: thus, we deserialize that message into a `Message` instance,
and then broadcast it to the `clientManager`.

```java
static Behavior clientInput(Address self, Address clientManager, BufferedReader in) {
    // schedule a message to self
    scheduler.schedule(() -> self.tell(Poll), 100, MILLISECONDS);

    return IOBehavior.of(msg -> {
        // ignore non-Poll messages
        if (msg != Poll) return Stay;
        if (in.ready()) {
            var input = in.readLine();
            // log message to stdout
            out.println(input);
            // deserialize from JSON
            Message m = Mapper.readValue(input, Message.class);
            // broadcast to all other clients
            clientManager.tell(m);
        }

        // "stay" in the same state, ensuring that the initializer is re-evaluated
        return Become(clientInput(self, clientManager, in));
    });
}
```

The input handler `clientInput` gets initialized with the `clientManager` address, 
and a `BufferedReader` that wraps the socket's input stream. This is just a convenient
way to read line-wise.

The first line of the `clientInput` method schedules to send `Poll` to `self`. 
This is to simulate an infinite loop; however, by scheduling a message instead
of actually writing a loop, we release each time the underlying thread for re-scheduling.

Please notice that we `schedule` the message instead of just sending with `self.tell(Poll)`,
because otherwise we would easily spike the CPU.

The last actor that we need will `Poll` the server socket for incoming connection, and
`tell` the client manager to spawn a new pair of `client` actors (our friends `clientInput` and `clientOutput`).

```java
static Behavior serverSocketHandler(Address self, Address clientManager, ServerSocket serverSocket) {
    scheduler.schedule(() -> self.tell(Poll), 1000, MILLISECONDS);
    return IOBehavior.of(msg -> {
        if (msg != Poll) return Stay;
        var socket = serverSocket.accept();
        clientManager.tell(new CreateClient(socket));
        return Become(serverSocketHandler(self, clientManager, serverSocket));
    });
}
```

That's it, you are done! You only need to get the ball rolling in your `main` method:

```java
static void main(String... args) throws IOException {
    var serverSocket = new ServerSocket(portNumber);
    out.printf("Server started at %s.\n", serverSocket.getLocalSocketAddress());

    var clientManager =
            sys.actorOf(self -> clientManager(self));
    var serverSocketHandler =
            io.actorOf(self -> serverSocketHandler(self, clientManager, serverSocket));
}
```

You can now start your server!

```sh
Server started at 0.0.0.0/0.0.0.0:4444.
```

Neat, huh?

## Chat Client

Of course, I lied. We are not done yet. We still need to write the client app.
Luckily that is incredibly short.

We only need 3 actors here. In fact, there is even little reason to use an actor system at all: to be fair,
the client could be almost single-threaded.

Yet, the API is so neat, that, you'd wonder why you should not. 

### Message Writer

We need an actor to receive messages and write them to the server socket.

```java
var serverOut = sys.actorOf(self -> ChatBehavior.IOBehavior.of(msg -> {
    if (msg instanceof Message m)
        socketOutput.println(ChatBehavior.Mapper.writeValueAsString(m));
    return Stay;
}));
```

Then we need two actors; both of them will `Poll` for input, and read line-by-line;
- in one case, we will read the input from the server socket and print it to screen
- in the other, we will read the input from the user and publish it to the server socket

In both cases the main meat is this method:

```java
static Actor.Behavior lineReader(Actor.Address self, BufferedReader in) {
    // schedule a message to self
    scheduler.schedule(() -> self.tell(Poll), 100, MILLISECONDS);

    return ChatBehavior.IOBehavior.of(msg -> {
        // ignore non-Poll messages
        if (msg != Poll) return Stay;
        if (in.ready()) {
            var input = in.readLine();
            /** handle input here **/
        }

        // "stay" in the same state, ensuring that the initializer is re-evaluated
        return Become(lineReader(self, in, lineConsumer));
    });
}
```


We can rewrite the following line as such:

```java
        if (in.ready()) {
            var input = in.readLine();
            lineConsumer.accept(input);
        }
```

and then declare it in the method signature as such:

```java
static Actor.Behavior lineReader(Actor.Address self, BufferedReader in, IOConsumer<String> lineConsumer) {
```

where the complete signature of `IOConsumer` is just:

```java
interface IOConsumer<T> {
    void accept(T t) throws IOException;
    static <T> Consumer<T> of(IOConsumer<T> f) {
        return msg -> {
            try { f.accept(msg); }
            catch (IOException e) { throw new UncheckedIOException(e); }};
    }
}
```

Thereby allowing to define `userIn` and `serverSocketReader` as such:

```java
var userIn = sys.actorOf(self -> lineReader(self,
        userInput,
        line -> serverOut.tell(new Message(userName, line))));
```

```java
var serverSocketReader = sys.actorOf(self -> lineReader(self,
        socketInput,
        line -> {
            Message message = ChatBehavior.Mapper.readValue(line, Message.class);
            out.printf("%s > %s\n", message.user(), message.text());
        }));
```


[minjavactors]: blah
[loom]: loom