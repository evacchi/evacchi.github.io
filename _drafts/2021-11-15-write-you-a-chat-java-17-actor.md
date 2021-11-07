---
title:  'Write You A Chat For Great Good! (with Java 17, actors, and JBang!)'
subtitle: ''
categories: [Java, Records, JBang]
date:   2021-11-03
---

Hello! 

Welcome back to ["Learn You An Actor (System) For Great Good!"][minjavactors]. If you haven't read [the first part][minjavactors], jump there to learn how to write a [minimalistic actor runtime using Java 17][minjavactors].

As promised, in this second part I am showing how to write a tiny chat client/server [using the runtime we wrote last time][minjavactors], that then we will run using [JBang!][jbang] Next time, we will learn how to create a **typed** version of the same actor runtime and revisit the examples!

For this chat application I have decided to use [the JDK's blocking Socket API][socket]. Now, you should always avoid calling a blocking API from the body of an actor, because you will block the entire underlying thread. However, we will do some damage-control by creating a special actor system for I/O, on its own thread pool. 

On a related note, on a [Loom-enabled JDK][loom], where threads are virtual, light-weight but still pre-emptible, even blocking I/O operations won't result in blocking a "real" OS thread. But until that lands in a stable JDK release, we will have to make do.

Because this post is quite long, here is a table of contents.

- [Overview](#overview)
  - [A Naive Java Implementation](#a-naive-java-implementation)
- [Boilerplate](#boilerplate)
- [Chat Server](#chat-server)
  - [Boilerplate](#boilerplate) ??
  - [`clientManager`](#clientmanager)
  - [`clientOutput`](#clientoutput)
  - [Polling](#polling)
  - [`clientInput`](clientinput) 
  - [`serverSocketHandler`](#serversockethandler)
- [Chat Client](#chat-client)
  - [Message Writer](#message-writer)

## Overview

Our chat applications will be extremely simple.

![Chat application where Duffman says "Are you ready?" to Carl, Lenny and Barney.](/assets/actor-2/chat.gif)

Each user picks a nickname, they connect to a server through their client and then they write their messages. The client waits for user input and displays the messages that it receives from the server to the screen.

The chat server accepts incoming connections, receives messages and it propagates them to all the clients that are connected at that time.

For instance, in this picture `Duffman` is sending the message `"Are you ready?"` to the server, and all of `Carl`, `Lenny` and `Barney` receive it ([to great sadness for Barney, who's the designated driver](https://www.youtube.com/watch?v=eEt-n0ZsYx8)).

### A Naive Java Implementation

In a Java program, the `ServerSocket` API provides all you need to `accept()` incoming connections from clients. 

Here is a naive snippet that is omitting all the details of spawning, handling threads and dealing with concurrency issues:

```java
class Server {
    ServerSocket serverSocket;
    List<ClientHandler> clientHandlers;
    Server(int port) throws IOException {
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

- The `Server` waits for connections and then it spin loops (`while (true)`); 
- `accept()` is a blocking API; every time it completes, it will return a new incoming connection (a `Socket`).
- every incoming connection represents a client connection (`ClientHandler`)
- we keep each client connection in a `List`


The `ClientHandler` class will read from the socket's `inputStream` and provide the `write()` method, writing to the socket's `outputStream`:

```java
class ClientHandler {
    Server parent; BufferedReader in; PrintWriter out;

    ClientHandler(Socket clientSocket, Server server) throws IOException {
        // get the in/out streams from the socket
        in = new BufferedReader(new InputStreamReader(clientSocket.getInputStream()));
        out = new PrintWriter(clientSocket.getOutputStream(), true);
        // keep a handle to the server to propagate messages back
        parent = server;
    }
    // on its own thread
    void read() throws IOException {
        while (in.ready()) {
            var line = validate(in.readLine());
            // for each line, broadcast to all other connected clients
            parent.broadcast(line);
        }
    }

    String validate(String readLine) { /* validate or throw */ }

    // called by server.broadcast()
    void write(String msg) {
        out.println(msg);
    }
}
```

- we read input from each client connection
- we broadcast every input to the output of all client connections 

As for the client, it is *slightly* less complicated:

- it connects to the server
- it reads from the connection all the incoming messages
- it writes each incoming message to the standard output (so the user can see it)
- it reads user messages from the standard input

```java

class Client {
    Socket socket; BufferedReader userInput, socketInput; PrintWriter socketOutput;
    Client(String host, int port) throws IOException {
        this.socket = new Socket(host, port);
        this.userInput = new BufferedReader(new InputStreamReader(System.in));
        this.socketInput = new BufferedReader(new InputStreamReader(socket.getInputStream()));
        this.socketOutput = new PrintWriter(socket.getOutputStream(), true);
    }
    // on its own thread
    void readUserInput() throws IOException {
        while (userInput.ready()) {
            var line = parse(userInput.readLine());
            write(line);
        }
    }
    // on its own thread
    void readServerInput() throws IOException {
        while (userInput.ready()) {
            var line = serialize(userInput.readLine());
            System.out.println(line); // echo to the user
            write(line);
        }
    }
    void write(String line) { socketOutput.println(line); }
    String serialize(String line) { /* serialize */ }
    String parse(String line) { /* parse or throw */ }
}
```

Now you will have noticed that there are quite a few blocking routines:

- `Server#start()`  
- `ClientHandler#read()`
- `Client#readUserInput()`
- `Client#readServerInput()`

Each of these routines should run on its own thread. I am sure you may be able to spot more issues; for instance, it is probably safer to guard the `Server#write()` and the `ClientHandler#broadcast()` from concurrent accesses with `synchronized`.

If we use actors instead, we will be guaranteed that the actor body is completely single-threaded and safe. Moreover, we will be forced to think about the concerns of our system, which may result in encapsulating each concern in its own actor.

## Boilerplate

Some methods in the `Socket` API will throw `IOException`s, which is a checked exception. Checked exception do not play nicely with lambdas, unless you define a method signature that accepts them. So let us define a couple of utilities in a shared library.

I am defining a `ChatBehaviors` interface where I will nest the few utilites
that will be used across the client and the server. Remember from the [first post][minjavactors]: the reason I am using interfaces is that most members will be `public` `static` by default. 


```java
public interface ChatBehavior {

}
```

Let's nest a `IOBehavior` functional interface:

```java
public interface ChatBehavior {
    interface IOBehavior {
        Actor.Effect apply(Object msg) throws IOException;
    }
}
```

It is basically *identical* to `Function<Object, Effect>` except it declares a `throws IOException` clause. We can now create a companion static method to transform an `IOBehavior` to a `Behavior` that throws an `UncheckedIOException`: we will use this in our actors.

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


## Chat Server

The simple architecture of our chat server is to have two root actors in our system


1. The `serverSocketHandler` waits for incoming connection on a Java `ServerSocket`. When there is a new incoming connection, it sends a `CreateClient` message to the `clientManager`.

2. `clientManager` creates and keeps track of all the active clients. When a client sends a message, it forwards the message to all the clients.


`clientManager` accepts two types of messages:
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

### Boilerplate

Let's create the envelope for our actors. Remember from the [first post][minjavactors]: the reason I am using interfaces is that most members will be `public` `static` by default. 

```java
public interface ChatServer {
    Actor.System sys = new Actor.System(Executors.newCachedThreadPool());
    Actor.System io = new Actor.System(Executors.newCachedThreadPool());

}
```

This interface will contain our `main` method and all of the `Behavior` methods. We will be also use it to define the messages, and a few utilities.

Some methods in the `Socket` API will throw `IOException`s, which is a checked exception. Checked exception do not play nicely with lambdas, unless you define a method signature that accepts them. So let us define a couple of utilities in a shared library.

```java
interface IOBehavior { Actor.Effect apply(Object msg) throws IOException; }
```

It is basically *identical* to `Function<Object, Effect>` except it declares a `throws IOException` clause. We can now create a companion static method to transform an `IOBehavior` to a `Behavior` that throws an `UncheckedIOException`: we will use this in our actors.

```java
interface IOBehavior { Actor.Effect apply(Object msg) throws IOException; }
static Actor.Behavior IO(IOBehavior behavior) {
    return msg -> {
        try { return behavior.apply(msg); } 
        catch (IOException e) { throw new UncheckedIOException(e); }};
}

```

This way we can create an actor that catches `IOException` and rethrows `UncheckedIOException`s as such

```java
var myActor = sys.actorOf(self -> IO(msg -> /* IO-throwing body */))
```


### `clientManager`

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
    return IO(msg -> {
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

### `clientOutput`

The `clientOutput` actor is extremely simple: it will just receive a message, and write it to the given `PrintWriter` instance.

Because we are serializing to JSON, we can assume newlines are not part of the serialized message, so they are safe both to write and to read.

```java
static Behavior clientOutput(PrintWriter writer) {
    return IO(msg -> {
        if (msg instanceof Message m) writer.println(m.payload());
        return Stay;
    });
}
```


### Polling

The blocking API will require polling. In the Java code we saw at the beginning we  implemented `while` loops of the form:

```java
while (true) {
    // blocking code here
}
```

![img](/assets/actor-2/server-poll.gif)


We will simulate this by creating a `Poll` message,
that the actor will send to itself. The type of the message
is not important; in fact we may match against the message directly:

```java
Behavior polling = msg -> {
    if (msg == Poll) {
        // blocking code here
    }
};
```

Now a word of warning should be added; you should usually avoid spinning
through a `while (true)` as you will keep the CPU busy 100% of the time.

It is usually better to add a tiny `sleep`, to release the CPU.

```java
while (true) {
    // blocking code here
    Thread.sleep(someSmallAmount);
}
```


In order to simulate the `sleep`, we may schedule a `Poll` message through a Java `ScheduledExecutor`.

```java
scheduler.schedule(() -> self.tell(Poll), 1, SECONDS);
```

Add the `Poll` message and the scheduler at the top of the `ChatServer` interface:

```java
public interface ChatServer {
    Actor.System sys = new Actor.System(Executors.newCachedThreadPool());
    Actor.System io = new Actor.System(Executors.newCachedThreadPool());
    ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(4);

    Object Poll = new Object();

```

### `clientInput` 

The input handler `clientInput` receives `Poll` messages at a given delay,
and reschedules the same message after processing input. It will discard any message that is not a `Poll`.

When it receives a `Poll` it checks the given `BufferedReader` if there is input,  and then read a line.

Each line contains a serialized payload: we don't need to inspect the contents, so  we just wrap it inside a `ServerMessage` instance, and then broadcast it to the `clientManager`.

Of course, in a proper server you should at least validate the payload so that it does not contain invalid or malicious values, but for simplicity we will not do it here.



```java
static Behavior clientInput(Address self, Address clientManager, BufferedReader in) {
    // schedule a message to self
    scheduler.schedule(() -> self.tell(Poll), 100, MILLISECONDS);

    return IOBehavior.of(msg -> {
        // ignore non-Poll messages
        if (msg != Poll) return Stay;
            if (in.ready()) {
                var m = in.readLine();
                // log message to stdout
                out.println(m);
                // broadcast to all other clients
                clientManager.tell(new ServerMessage(m));
            }


        // "stay" in the same state, ensuring that the initializer is re-evaluated
        return Become(clientInput(self, clientManager, in));
    });
}
```

The input handler `clientInput` gets initialized with the `clientManager` address, 
and a `BufferedReader` that wraps the socket's input stream. This is just a convenient way to read line-wise.

The first line of the `clientInput` method schedules to send `Poll` to `self`. 
This is to simulate an infinite loop; however, by scheduling a message instead
of actually writing a loop, we release each time the underlying thread for re-scheduling.

The last line is a `Become` to the same behavior. Because the behavior initially schedules the `Poll`, this means that every time the actor body gets executed, it will re-schedule the `Poll`, effectively making it loop indefinitely.

You may have noticed that we do not use `scheduleAtFixedRate()` or `scheduleWithFixedDelay()`; instead, we execute the body *and then* reschedule the message. The reason is that an actor may receive other messages in the meantime, and we don't want to clog its mailbox with `Poll` messages while the body is still executing.

### `serverSocketHandler`

The last actor that we need will `Poll` the server socket for incoming connection, and
`tell` the client manager to spawn a new pair of `client` actors (our friends `clientInput` and `clientOutput`).

```java
static Behavior serverSocketHandler(Address self, Address clientManager, ServerSocket serverSocket) {
    scheduler.schedule(() -> self.tell(Poll), 1000, MILLISECONDS);
    return IO(msg -> {
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

Of course, we are not done yet. We still need to write the client app.
Luckily that is incredibly short.

We only need 3 actors here. In fact, there is even little reason to use an actor system at all: to be fair,
the client could be almost single-threaded.

Yet, the actor-based API is so neat, that, you'd wonder why you should not. 

Let us create the envelope with a few utilities: 

```java
public interface Client {
    interface IOConsumer<T> { void accept(T t) throws IOException; }
    interface IOBehavior { Actor.Effect apply(Object msg) throws IOException; }
    static Actor.Behavior IO(IOBehavior behavior) {
        return msg -> {
            try { return behavior.apply(msg); }
            catch (IOException e) { throw new UncheckedIOException(e); }
        };
    }
}
```

For simplicity I am duplicating `IOBehavior` and `IO` here. You may also write your own shared class,
but this will keep the script stand-alone for running through JBang later!

### Message Writer

We need an actor to receive messages and write them to the server socket.

```java
var serverOut = sys.actorOf(self -> IO(msg -> {
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

    return IO(msg -> {
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
interface IOConsumer<T> { void accept(T t) throws IOException; }
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
[jbang]: https://jbang.dev
[socket]: socket
[loom]: loom