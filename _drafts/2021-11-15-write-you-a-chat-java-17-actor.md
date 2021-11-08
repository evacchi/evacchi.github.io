---
title:  'Write You A Chat For Great Good! (with Java 17, actors, and JBang!)'
subtitle: ''
categories: [Java, Records, JBang]
date:   2021-11-03
---

Hello! 

Welcome back to ["Learn You An Actor (System) For Great Good!"][minjavactors]. If you haven't read [the first part][minjavactors], jump there to learn how to write a [minimalistic actor runtime using Java 17][minjavactors].

As promised, in this second part I am showing how to write a tiny chat client/server [using the runtime we wrote last time][minjavactors], that then we will run using [JBang!][jbang] Next time, we will learn how to create a **typed** version of the same actor runtime and revisit the examples!

Because this post is quite long, here is a table of contents.

- [Overview](#overview)
  - [A Naive Java Implementation](#a-naive-java-implementation)
- [Chat Server](#chat-server)
  - [Root Actors](#root-actors) 
  - [`clientManager`](#clientmanager)
  - [`clientOutput`](#clientoutput)
  - [Polling](#polling)
  - [`clientInput`](clientinput) 
  - [`serverSocketHandler`](#serversockethandler)
- [Chat Client](#chat-client)
  - [Client Actors](#client-actors)
- [Conclusions](#conclusions)

## Overview

Our chat applications will be extremely simple.

![Chat application where Duffman says "Are you ready?" to Carl, Lenny and Barney.](/assets/actor-2/chat.gif)

Each user picks a nickname, they connect to a server through their client and then they write their messages. The client waits for user input and displays the messages that it receives from the server to the screen.

The chat server accepts incoming connections, receives messages and it propagates them to all the clients that are connected at that time.

For instance, in this picture `Duffman` is sending the message `"Are you ready?"` to the server, and all of `Carl`, `Lenny` and `Barney` receive it ([to great sadness for Barney, who's the designated driver](https://www.youtube.com/watch?v=eEt-n0ZsYx8)).

### A Naive Java Implementation

For this chat application I have decided to use [the JDK's blocking Socket API][socket]. Now, you should always avoid calling a blocking API from the body of an actor, because you will block the entire underlying thread. However, we will do some damage-control by creating a special actor system for I/O, on its own thread pool. 

On a related note, on a [Loom-enabled JDK][loom], where threads are virtual, light-weight but still pre-emptible, even blocking I/O operations won't result in blocking a "real" OS thread. But until that lands in a stable JDK release, we will have to make do.

The `ServerSocket` API provides all you need to `accept()` incoming connections from clients. 

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

## Chat Server

Let's get our hands dirty.

First of all, let's create the "envelope" for our actors. In the [first post][minjavactors]: I chose to use an interface as a code container. The reason is that most members will be `public` `static` by default. 

```java
public interface ChatServer {

}
```

This interface will contain our `main` method and all of the `Behavior` methods. We will be also use it to define the messages, and a few utilities.

Some methods in the `ServerSocket` and `Socket` APIs are blocking; some of them throw `IOException`s, which is a checked exception. 

- Blocking APIs do not play nicely with an actor runtime because they block the underlying thread.
- Checked exception do not play nicely with lambdas, unless you define a method signature that accepts them. 

So, we will create **two** actor systems, and dedicate one to I/O and blocking calls:

```java
public interface ChatServer {
    Actor.System sys = new Actor.System(Executors.newCachedThreadPool());
    Actor.System io = new Actor.System(Executors.newFixedThreadPool(2));
}
```

Notice that for `io` we are using a `FixedThreadPool`. This prevents the executor from spawning more threads when some of them are blocked. 

Let us also define a functional interface for I/O; because many I/O methods throw `IOException`s we define an `IOBehavior` that throws: 

```java
public interface ChatServer {
    ...
    interface IOBehavior { Actor.Effect apply(Object msg) throws IOException; }
    static Actor.Behavior IO(IOBehavior behavior) {
        return msg -> {
            try { return behavior.apply(msg); } 
            catch (IOException e) { throw new UncheckedIOException(e); }};
}
```

`IOBehavior` is basically *identical* to `Function<Object, Effect>` except it declares it `throws IOException`. The `IO` method turns a `IOBehavior` that throws `IOException`s into a plain `Behavior` that catches an `IOException` turning it into an `UncheckedIOException`. This will let us write:

```java
var myActor = sys.actorOf(self -> IO(msg -> /* IO-throwing body */))
```

### Root Actors

In the actor-based implementation of the Server, I have chosen to break down the server-related behavior in two "root" actors. One deals directly with I/O (`serverSocketHandler`) and the other (`clientManager`) orchestrates the client connections: 

1. `serverSocketHandler` waits for incoming connection on the `ServerSocket`. When there is a new incoming connection, it sends a `CreateClient` message to the `clientManager`.

2. `clientManager` creates and keeps track of all the active clients. When a client sends a message, it forwards the message to all the clients.

```java
int portNumber = 4444;

static void main(String... args) throws IOException {
    var serverSocket = new ServerSocket(portNumber);
    out.printf("Server started at %s.\n", serverSocket.getLocalSocketAddress());

    var clientManager =
            sys.actorOf(self -> clientManager(self));
    var serverSocketHandler =
            io.actorOf(self -> serverSocketHandler(self, clientManager, serverSocket));
}
```

Notice how `serverSocketHandler` was created on the `io` actor system as it will invoke a blocking API.

### `clientManager`

The "core" of the server is really the `clientManager`. A `clientManager` accepts two types of messages:

```java
record CreateClient(Socket socket) {}
record ServerMessage(String payload) {}
```

Each time a `CreateClient` message is received, it  spawns two actors:

- a `clientOutput` actor to forward messages
- a `clientInput` actor to receive input from each client

![When the CreateClient message is received, the clientManager creates clientOutput and clientInput pairs](/assets/actor-2/actor-create.gif)

`clientInput` listens for input coming from the client socket. When input is available, it parses the input and forwards it as a `ServerMessage` object that is sent back to the `clientManager`.

Because the Client Manager keeps track of all Client Output actors, it is able to forward the `ServerMessage` to all of their outputs, i.e. their respective `clientOutput` actor; this way all connected clients will receive a copy of the message.

![clientManager forwards ServerMessage to all the connected clients](/assets/actor-2/message.gif)


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
                var clientInput =
                    io.actorOf(me -> clientInput(me, self, in));

                var writer = new PrintWriter(socket.getOutputStream(), true);
                var clientOutput =
                    sys.actorOf(client -> clientOutput(writer));

                clients.add(clientOutput);
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

Notice how `clientInput` was created on the `io` actor system, as it will invoke a blocking API.

This is more or less the most complicated actor you'll see.

### `clientOutput`

The `clientOutput` actor is extremely simple: it will just receive a message, and write it to the given `PrintWriter` instance.

Because we are serializing to JSON, we can assume newlines are not part of the serialized message, so they are safe to read and write line-wise

```java
static Behavior clientOutput(PrintWriter writer) {
    return IO(msg -> {
        if (msg instanceof Message m) writer.println(m.payload());
        return Stay;
    });
}
```

### Polling

I have delayed describing `clientInput` and `serverSocketHandler` because both must invoke blocking APIs. To this effect, they were created on the `io` actor system, with its own, dedicated thread pool.
 
As we saw at the beginning these blocking APIs are usually handled within `while` loops of the form:

```java
while (true) {
    // blocking code here
}
```

![Actors polling connections](/assets/actor-2/server-poll.gif)

Now, we can't write a `while` loop in the body of our actors as that would steal the thread forever. We can still simulate a similar behavior by creating a `Poll` message, that the actor will send to itself. The type of the message is not important; in fact we may match against the message directly; for instance:

```java
static Behavior polling(Address self){
    return msg -> {
        if (msg == Poll) {
            // blocking code here
            self.tell(Poll);
        }
        return Stay.
    };
}
```

Now, because the behavior actually returns, we are effectively releasing the thread each time. Because the `Poll` message will be re-enequeued in the actor mailbox, we give the chance to other actors to run, but we are effectively scheduling this actor back for execution as soon as possible. 

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

Add the `Poll` message and the scheduler at the top of the `ChatServer` interface:

```java
public interface ChatServer {
    Actor.System sys = new Actor.System(Executors.newCachedThreadPool());
    Actor.System io = new Actor.System(Executors.newCachedThreadPool());
    ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(4);

    Object Poll = new Object();
    ...
```

We can then rewrite the behavior as:

```java
static Behavior polling(Address self){
    return msg -> {
        if (msg == Poll) {
            // blocking code here
            scheduler.schedule(() -> self.tell(Poll), 1, SECOND);
        }
        return Stay.
    };
}
```

An alternative way to write may be also:

```java
static Behavior polling(Address self){
    scheduler.schedule(() -> self.tell(Poll), 1, SECOND);

    return msg -> {
        if (msg == Poll) {
            // blocking code here
        }
        return Become(polling(self)).
    };
}
```

In this case, the first time the behavior is evaluated we already schedule the first `Poll`; each time a `Poll` is received, we force re-evaluating the body of the method, causing it to re-schedule the `Poll`.

You may have noticed that we do not use `scheduleAtFixedRate()` or `scheduleWithFixedDelay()`; instead, we execute the body *and then* reschedule the message. The reason is that these actor will invoke a blocking API, thus they may block for an indefinite amount of time. scheduling at a fixed rate would end up enqueing messages to the mailbox, regardless if they are blocked or not, clogging their mailbox.

By re-scheduling, we ensure that the message is re-scheduled only when the actor has unblocked. 

This will be the blueprint for both `clientInput` and `serverSocketHandler`.

### `clientInput` 

The input handler `clientInput` receives `Poll` messages at a given delay, and reschedules the same message after processing input. It will discard any message that is not a `Poll`.

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

The input handler `clientInput` gets initialized with the `clientManager` address, and a `BufferedReader` that wraps the socket's input stream. This is just a convenient way to read line-wise.

### `serverSocketHandler`

The last actor that we need will `Poll` the server socket for incoming connection, and `tell` the client manager to spawn a new pair of `client` actors (our friends `clientInput` and `clientOutput`).

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

You can now start your server with JBang! Add the following lines at the top of your file:

```java
//JAVA 17
//JAVAC_OPTIONS --enable-preview --release 17
//JAVA_OPTIONS  --enable-preview
//REPOS jitpack=https://jitpack.io/
//DEPS com.github.evacchi:min-java-actors:main-SNAPSHOT
```

If everything is right, then you can type:

```sh
j! ChatServer.java
```

If you are lazy, you can run it from this Gist directly: 

```
j! https://gist.github.com/evacchi/b6dbd0b848b0940b862246750026460e
```

The program will start waiting for incoming connections, printing:

```sh
Server started at 0.0.0.0/0.0.0.0:4444.
```

Neat, huh?


## Chat Client

Of course, we are not done yet. We still need to write the client app.
Luckily that is incredibly short: we only need 3 actors. To be fair, you may just dedicate one thread per actor and call it a day, but for completeness, let's use the actor system: the result will be quite compact. 

We will define:

- `serverOut`: an actor that *writes* to the server socket
- `userIn`: an actor that *reads* the messages from standard input
- `serverSocketReader`: an actor that *reads* the messages that the server re-broadcasts from/to other clients


![A client connecting to socket, reading messages from standard input, writing to standard output](/assets/actor-2/actor-client.png)

Notice that, for simplicity, the messages that are written locally
are not echoed immediately to screen (as it would usually happen). Instead, we will always print to screen whatever comes back from the server. Because the server always re-broadcasts *everything* to *everyone*, we will *also* effectively echo whatever the user wrote.

Let's start with the "envelope" with a few utilities: 

```java
public interface ChatClient {
    interface IOBehavior { Actor.Effect apply(Object msg) throws IOException; }
    static Actor.Behavior IO(IOBehavior behavior) {
        return msg -> {
            try { return behavior.apply(msg); }
            catch (IOException e) { throw new UncheckedIOException(e); }
        };
    }
}
```

For simplicity I am duplicating `IOBehavior` and `IO` here. You may also write your own shared class, but this will keep the script stand-alone for running through JBang later!

There is only 3 actors in this actor system. So if our thread pool size is at least >=3 we don't really need to separate the I/O thread from the message thread.

```java
Actor.System sys = new Actor.System(Executors.newCachedThreadPool());
```
However, it is still wise to `Poll` using a scheduler:

```java
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(4);
```

There is only two types of messages, the `Poll` and the client `Message`

```java
static Object Poll = new Object();
record Message(String user, String text) {}
```

In this case, the `Message` contains the nickname of the user who wrote the message, and the actual text of the message.

Let us now write the `main` routine with the initialization logic.

```java
String host = "localhost";
int portNumber = 4444;

static void main(String[] args) throws IOException {
    // read the nickname from command-line arguments
    var userName = args[0];

    // we will use this to (de)serialize `Message` to JSON
    var mapper = new ObjectMapper();
    
    // initialize the socket, the readers and the writers
    var socket = new Socket(host, portNumber);
    var userInput = new BufferedReader(new InputStreamReader(in));
    var socketInput = new BufferedReader(new InputStreamReader(socket.getInputStream()));
    var socketOutput = new PrintWriter(socket.getOutputStream(), true);

    // print some info to screen
    out.printf("Login........%s\n", userName);
    out.printf("Local Port...%d\n", socket.getLocalPort());
    out.printf("Server.......%s\n", socket.getRemoteSocketAddress());

    // ... actors go here ...
}
```

The only thing missing are now the 3 actors!

### Client Actors

As we saw at the beginning, `serverOut` is the simplest actor of all, because it just receives messages and serializes them to the server socket.

```java
var serverOut = sys.actorOf(self -> IO(msg -> {
    if (msg instanceof Message m)
        socketOutput.println(mapper.writeValueAsString(m));
    return Stay;
}));
```

`userIn` and `serverSocketReader` are very similar: 

- both of them will `Poll` for input, and read line-by-line;
- `userIn` reads the input from the user and publish it to the server socket
- `serverSocketReader` reads the input from the server socket and print it to screen

We expect both actors to respect the following blueprint:

```java
static Behavior readLine(Address self, BufferedReader in) {
    // schedule a message to self
    scheduler.schedule(() -> self.tell(Poll), 100, MILLISECONDS);

    return IO(msg -> {
        // ignore non-Poll messages
        if (msg != Poll) return Stay;
        if (in.ready()) {
            var input = in.readLine();
            handleInput(input); // undefined
        }

        // "stay" in the same state, ensuring that the initializer is re-evaluated
        return Become(lineReader(self, in, lineConsumer));
    });
}
```

`handleInput(input)` is still undefined, because each actor should
customize it. We can keep the code tidy by defining a lambda.

Add the following definition of `IOLineReader` at the top:

```java
public interface Client {
    interface IOLineReader { void read(String line) throws IOException; }
    ...
```


and then declare it in the method signature as such:

```java
static Behavior readLine(Address self, BufferedReader in, IOLineReader lineReader) {
```

Then we can use it as:

```java
        if (in.ready()) {
            var input = in.readLine();
            lineReader.read(input); // undefined
        }
```

Thereby allowing to define `userIn` and `serverSocketReader` as such:

```java
var userIn = sys.actorOf(self -> readLine(self,
        userInput,
        line -> serverOut.tell(new Message(userName, line))));
```

`userIn` reads a line then sends it to `serverOut`

```java
var serverSocketReader = sys.actorOf(self -> readLine(self,
        socketInput,
        line -> {
            Message message = mapper.readValue(line, Message.class);
            out.printf("%s > %s\n", message.user(), message.text());
        }));
```

`serverSocketHandler` receives each line from the server, 
deserializes it, and prints it to standard output.

And now you are *really* done: you can now start a client with JBang! Add the following lines at the top of your file:

```java
//JAVA 17
//JAVAC_OPTIONS --enable-preview --release 17
//JAVA_OPTIONS  --enable-preview
//DEPS com.fasterxml.jackson.core:jackson-databind:2.13.0
//DEPS com.github.evacchi:min-java-actors:main-SNAPSHOT
```

If everything is right, then you can type:

```sh
j! ChatClient.java your-nickname
```

If you are lazy, you can run it from this Gist directly (make sure the server is running!): 

```
j! https://gist.github.com/evacchi/6bbe4bca3df51a29d17d5c10917c91ec your-nickname
```

The program will start waiting for incoming connections, printing something like:

```sh
Login........your-nickname
Local Port...57942
Server.......localhost/127.0.0.1:4444
```

Here is a full demo!

<div style="text-align: center">
<video controls autoplay width="402" height="600">
  <source src="/assets/actor-2/chat-demo.mp4" type="video/mp4">
  <a href="/assets/actor-2/chat-demo.mp4">Full demo</a>
</video>
</div>

## Conclusions

In this post we have learned how to write a simple chat app. As promised in the [previous post][minjavactors], In the final
part of this series we will revisit the actor runtime and define a **fully-typed** version, which will benefit from exhaustiveness checks!

I am also happy to announce that [I have been selected](https://twitter.com/JavaAdvent/status/1457409222048636940) for the [Java Advent Calendar 2021][javaadvent], so the last part of this series will be first published on the [Java Advent Calendar][javaadvent] website! [Follow them on Twitter](https://twitter.com/JavaAdvent) for updates!

See you there!


[minjavactors]: blah
[jbang]: https://jbang.dev
[socket]: socket
[loom]: loom
[javaadvent]: https://www.javaadvent.com/