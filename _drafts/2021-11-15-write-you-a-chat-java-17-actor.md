---
title:  'Write You A Chat For Great Good! (with Java 17, actors, and JBang!)'
subtitle: ''
categories: [Java, Records, JBang]
date:   2021-11-03
---

Hello! 

Welcome back to this blog series! If you haven't read [the previous post][minjavactors] jump there to learn how to write a [minimalistic actor runtime][minjavactors] using Java 17.

As promised, with this blog post, I am showing how to write a tiny chat client/server.

For simplicity, I have decided to use [Java's blocking Socket API][socket]. Now, you should always avoid calling a blocking API from the body of an actor, because you will block the entire underlying thread.
However, we will do damage-control by creating a special actor system for I/O, on its own thread pool. 

On a related note, on a [Loom-enabled JDK][loom], where threads are virtual, light-weight but still pre-emptible, even blocking I/O operations won't result in blocking a "real" OS thread. But until that lands in a stable JDK release, we will have to make do.

## Overview

Everyone knows how a chat client works. 

Each user picks a nickname, connects to a server through their client and then they write their messages. The client waits for user input and displays the messages that it receives from the server to the screen.

The role of the chat server is to accept inconming connections from the clients, receive messages from each client and propagate them to all the clients that are connected at that time.

In a Java program, the `ServerSocket` API provides all you need to `accept()` incoming connections from clients. Here is a naive snippet
that is omitting all the details of spawning and handling threads, and all of the concurrency issues    :

```java
class Server {
    ServerSocket serverSocket;
    List<ClientHandler> clientHandlers;
    Server() {
        serverSocket = new ServerSocket(port);
        clientHandlers = new ArrayList<>();
    }
    void start() throws IOException {
        while (true) {
            // blocks until a new connection is established
            var clientSocket = serverSocket.accept();
            // then for each clientSocket...
            var handler = new ClientHandler(clientSocket, this);
            clientHandlers.add(handler);
            /* start handler.read() in a separate thread */
        }
    }
    // called by clientHandlers that want to propagate
    // messages to all the connected clients
    void broadcast(String msg) {
        for (var handler: clientHandlers) {
            handler.write(msg);
        }
    }
}
```

The `ClientHandler` class will read from the socket's `inputStream` and provide the `write()` method, writing to the socket's `outputStream`.

```java
class ClientHandler {
    Server parent; BufferedReader in; PrintWriter out;
    
    ClientHandler(Socket clientSocket, Server server) {
        // get the in/out streams from the socket
        in = new BufferedReader(new InputStreamReader(clientSocket.getInputStream()));
        out = new PrintWriter(clientSocket.getOutputStream(), true);
        // keep a handle to the server to propagate messages back
        parent = server;
    }
    void read() throws IOException {
        while (in.ready()) {
            var line = parse(in.nextLine());
            // for each line, broadcast to all other connected clients
            parent.broadcast(line);
        }
    }
    // called by server.broadcast()
    void write(String msg) {
        out.println(msg);
    }    
}
```

Now you will have noticed at at least two issues:

1. `Server#start` blocks the main thread 
2. `ClientHandler#read()` needs necessarily to run on the same threa

and I am sure you may be able to spot more; for instance, it is probably safer to guard the `Server#write()` and the `ClientHandler#broadcast()` from concurrent accesses with `synchronized`

Now, by writing this using actors, we will be guaranteed that the actor body is completely single-threaded and safe. Moreover, we will be forced to think about the concerns of our system, which may result in encapsulate each in their own actor.

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
    Actor.Effect apply(Object msg) throws IOException;
    static Actor.Behavior of(IOBehavior behavior) {
        return msg -> {
            try { return behavior.apply(msg); } 
            catch (IOException e) { throw new UncheckedIOException(e); }};
    }
}
```

This way we can create an actor that catches and rethrows `IOException`s as such

```java
  var myActor = sys.actorOf(self -> IOBehavior.of(msg -> /* IO-throwing body */))
```

### Polling

![img](/assets/actor-2/server-poll.png)

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

The simple architecture of our chat server is to have two root actors in our system


1. The `serverSocketHandler` waits for incoming connection on a Java `ServerSocket`. When there is a new incoming connection, it sends a `CreateClient` message to the `clientManager`.

2. The Client Manager creates and keeps track of all the active clients. When a client sends a message, it forwards the message to all the clients.



accepts two types of messages:
- `CreateClient` 
- `Message` 

Each time a `CreateClient` message is received, it  spawns two actor:

- a `clientOutput` actor to forward messages
- a `clientInput` actor to receive input from each client


![img](/assets/actor-2/actor-create.gif)


The `clientInput` listens for input coming from the client socket. When input is available, it parses the input and forwards it as a `Message` object that is sent back to the `clientManager`.


Because the Client Manager keeps track of all Client Output actors, it is able
to forward that message to all of them

![img](/assets/actor-2/message.gif)

Our chat server will receive incoming connections

![img](/assets/actor-2/server-poll.gif)


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
[socket]: socket
[loom]: loom