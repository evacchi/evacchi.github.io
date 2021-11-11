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

For this chat application ([after my friend Andrea][andrea] bugged me to no end), I have decided to use [the JDK's asynchronous Socket API][socket]. The `AsynchronousServerSocketChannel` API provides all you need to `accept()` incoming connections from clients. For instance: 

```java
var socketChannel = AsynchronousServerSocketChannel.open();
socketChannel.setOption(StandardSocketOptions.SO_REUSEADDR, true);
socketChannel.bind(new InetSocketAddress(HOST, PORT_NUMBER));

socketChannel.accept(null, new CompletionHandler<>() {
    public void completed(AsynchronousSocketChannel result, Void attachment) { /* handle result/attachment */ }
    public void failed(Throwable exc, Void attachment) { /* handle exception/attachment */ }
});
```

The API being asynchronous, it works through *callbacks*. Additionally, being the API from Java 1.7, it uses *callbacks* or the blocking `java.util.concurrent.Future` instead of lambdas or `j.u.c.CompletableFuture`s.

Each call to `accept()` on the `AsynchronousServerSocketChannel` results (if successful) in returning an `AsynchronousSocketChannel`, which represents a client connection. 

So, roughly, this how a chat server and client should work:

The `AsynchronousServerSocketChannel` waits for connections by `accept()`ing them:
- every time the callback is invoked a new incoming connection (an `AsynchronousSocketChannel`) is returned.
- every `AsynchronousSocketChannel` represents a client connection

For each `AsynchronousSocketChannel` has been established, the `Server`:

- reads input from each client connection with `AsynchronousSocketChannel#read()`
- it broadcasts every input to the output of all client connections `AsynchronousSocketChannel#write()`

Both `read()` and `write()` sport the same callback-based interface:

```java
var buf = ByteBuffer.wrap(msgBytes);
channel.write(buf, null, new CompletionHandler<>() {
    public void completed(Integer bytesWritten, Void ignored) { /* handle successful case */ }
    public void failed(Throwable exc, Void ignored) { /* handle exception */ }
});

var buf = ByteBuffer.allocate(BUFFER_SIZE);
channel.read(buf, buf, new CompletionHandler<>() {
    public void completed(Integer bytesWritten, ByteBuffer bb) { /* handle bytesWritten/buffer */ }
    public void failed(Throwable exc, ByteBuffer bb) { /* handle exception/buffer */ }
});
```

Both `read()` and `write()` use a `ByteBuffer` to represent an array of bytes. 
In particular, the `read()` method reads *at most* `BUFFER_SIZE` bytes, but it may return earlier.


The `Client` connects to the server by *creating* an `AsynchronousSocketChannel`:


```java
channel.connect(new InetSocketAddress(HOST, PORT_NUMBER), attachment, new CompletionHandler<>() {
    public void completed(Void ignored, Object attachment) { /* handle attachment */ }
    public void failed(Throwable exc, Object attachment) { /* handle exception/attachment */ }
})
```

- it `read()`s from the connection all the incoming messages
- it `write()`s each incoming message to the standard output (so the user can see it)
- it reads user messages from the standard input

### Protocol
We will use `Jackson` for serialization each message into JSON. Because JSON does not allow unescaped newlines,
we will use  `'\n'` as a message separator in the input/output stream of the socket.

That means that in order to *send* a message, a *client* must serialize a message to JSON,
concatenating a newline character at the end of the payload.
In order to *receive* a *client* must buffer the bytes in the input stream, until a newline is encountered; 
when a newline is found, the byte buffer is the serialized message, and it can be deserialized and displayed.

A server, is comparatively simple: for each client, it will tokenize its input stream at newline, 
and re-broadcast the token to all connected clients; in a real-world implementation the input should be at least validated,
but we will skip it here for simplicity.

## Managing Connections

Let's get our hands dirty.

First of all, let's create the "envelope" for our actors. In the [first post][minjavactors]: I chose to use an interface
as a code container. The reason is that most members will be `public` `static` by default. 

```java
interface Channels {

}
```

You can place here these constants, they will be useful in a bit:


```java
interface Channels {
    String HOST = "localhost";
    int PORT_NUMBER = 4444;
}
```


Now, order to keep our actors tidy, we can define a couple of handy private utility methods 
to convert `CompletionHandler`s into `CompletableFuture`s:

```java
interface Channels {
    private static <A, B> CompletionHandler<A, B> handleAttachment(CompletableFuture<B> f) {
        return new CompletionHandler<>() {
            public void completed(A result, B attachment) { f.complete(attachment); }
            public void failed(Throwable exc, B attachment) { f.completeExceptionally(exc); }
        };
    }
    private static <A, B> CompletionHandler<A, B> handleResult(CompletableFuture<A> f) {
        return new CompletionHandler<>() {
            public void completed(A result, B attachment) { f.complete(result); }
            public void failed(Throwable exc, B attachment) { f.completeExceptionally(exc); }
        };
    }
}
```

You may use these methods directly, but instead, I suggest we create two handy wrappers for `AsynchronousServerSocketChannel` and 
`AsynchronousSocketChannel`. Nest under `Channels` this `ServerSocket` wrapper for `AsynchronousServerSocketChannel`:

```java
class ServerSocket {
    AsynchronousServerSocketChannel socketChannel;
    private ServerSocket(AsynchronousServerSocketChannel socketChannel) { this.socketChannel = socketChannel; }

    static ServerSocket open() throws IOException {
        var socketChannel = AsynchronousServerSocketChannel.open();
        socketChannel.setOption(StandardSocketOptions.SO_REUSEADDR, true);
        socketChannel.bind(new InetSocketAddress(HOST, PORT_NUMBER));
        out.printf("Server started at %s.\n", socketChannel.getLocalAddress());
        return new ServerSocket(socketChannel);
    }

    CompletableFuture<Socket> accept() {
        var f = new CompletableFuture<AsynchronousSocketChannel>();
        socketChannel.accept(null, handleResult(f));
        return f.thenApply(Socket::new);
    }
}
```

and `Socket`, wrapping `AsynchronousSocketChannel`:

```java
class Socket {
    AsynchronousSocketChannel channel;
    Socket(AsynchronousSocketChannel channel) { this.channel = channel; }

    CompletableFuture<Socket> connect() {
        var f = new CompletableFuture<Socket>();
        channel.connect(new InetSocketAddress(HOST, PORT_NUMBER), this, handleAttachment(f));
        return f;
    }

    /**
     *  writes a string and concatenates a new line
     */
    CompletableFuture<Void> write(String line) {
        var f = new CompletableFuture<Void>();
        var buf = ByteBuffer.wrap((line + END_LINE).getBytes(StandardCharsets.UTF_8));
        channel.write(buf, null, handleAttachment(f));
        return f;
    }

    CompletableFuture<String> read() {
        var f = new CompletableFuture<ByteBuffer>();
        var buf = ByteBuffer.allocate(2048);
        channel.read(buf, buf, handleAttachment(f));
        return f.thenApply(bb -> new String(bb.array()));
    }
}
```

We are now ready to start.

## Chat Server

Let's create another "envelope" for our actors. 

```java
public interface ChatServer {

}
```

This interface will contain our `main` method and all of the `Behavior` methods. 
We will be also use it to define the messages, and a few utilities.

```java
public interface ChatServer {
    Actor.System system = new Actor.System(Executors.newCachedThreadPool());
}
```

In the actor-based implementation of the Server, I have chosen to break down the server-related behavior into two "root" actors. 
One deals directly with I/O (`serverSocketHandler`) and the other (`clientManager`) orchestrates the client connections: 

1. `serverSocketHandler` waits for incoming connection on the `ServerSocket`. When there is a new incoming connection, 
it sends a `CreateClient` message to the `clientManager`.

2. `clientManager` creates and keeps track of all the active clients. When a client sends a message, it forwards the message to all the clients.

```java
static void main(String... args) throws IOException, InterruptedException {
    var serverSocket = Channels.ServerSocket.open();

    var clientManager = 
            system.actorOf(self -> clientManager(self));
    var serverSocketHandler = 
            system.actorOf(self -> serverSocketHandler(self, clientManager, serverSocket));

    Thread.currentThread().join(); // ensure the main thread does not quit
}
```




### `serverSocketHandler`

This actor will `accept()` incoming connection; when such a connection is received, it will spawn a new `client` actor
and send it to the `clientManager`.

```java

record ClientConnection(Channels.Socket socket) { }
record ClientConnected(Address addr) { }

static Behavior serverSocketHandler(Address self, Address childrenManager, Channels.ServerSocket serverSocket) {
    out.println("Server in open socket!");
    serverSocket.accept()
            .thenAccept(skt -> self.tell(new ClientConnection(skt)))
            .exceptionally(exc -> { exc.printStackTrace(); return null; });

    return msg -> switch (msg) {
        case ClientConnection conn -> {
            out.println("Child connected!");
            var clientSocketHandler =
                    system.actorOf(ca -> clientSocketHandler(ca, childrenManager, conn.socket()));
            childrenManager.tell(new ClientConnected(client));

            yield Become(serverSocketHandler(self, childrenManager, serverSocket));
        }
        default -> throw new RuntimeException("Unhandled message " + msg);
    };
}

```


### `clientSocketHandler`

The `clientSocketHandler`:
- reads buffers from a `Channels.Socket` and tokenizes messages at each new line. 
- writes messages to a  `Channels.Socket`

Let us start with reads; define the following messages:

```java
record ReadBuffer(String content) {}
record LineRead(String payload) {}
```

We need to subscribe the channel, then notify the actor that a buffer is ready;
the `Behavior` will have more or less the following structure:


```java
static Behavior clientSocketHandler(Address self, Address clientManager, Channels.Socket channel) {
    // subscribe the channel
    channel.read()
            .thenAccept(s -> self.tell(new ReadBuffer(s)))
            .exceptionally(err -> { err.printStackTrace(); return null; });

    return msg -> switch (msg) {
        case ReadBuffer buffer -> {
            // tokenize the string
        }
        // other cases...
    };
}
```

Now, when a `ReadBuffer` is received, we need to look for newlines, and then keep a buffer of the message
that we have read so far:

```java
String buff; // previously read buffer 
case ReadBuffer incoming -> {
    // look for a new line
    int eol = incoming.content().indexOf(END_LINE);
    if (eol >= 0) {
        // if there is a new line, the message is the buffer + incoming.content() split at newline
        var line = buff + incoming.content().substring(0, eol);
        incoming.tell(new LineRead(line));
        // overwrite `buff` with the rest of the buffer (after the newline)
        buff = incoming.content().substring(eol + 2);
        // invoke again channel.read() and wait for more buffered input
    } else {
        // otherwise, concatenate the entire incoming buffer
        buff += incoming.content();
        // invoke again channel.read() and wait for more buffered input
    }
```

We can create a "recursive" behavior that accumulates into `buff` the message that has been read so-far.
We make `buff` an argument of the method:

```java
static Behavior clientSocketHandler(Address self, Address clientManager, Channels.Socket channel, String buff) {
    // subscribe the channel
    channel.read()
            .thenAccept(s -> self.tell(new ReadBuffer(s)))
            .exceptionally(err -> { err.printStackTrace(); return null; });

    return switch (msg) {
        case ReadBuffer incoming -> {
            // look for a new line
            int eol = incoming.content().indexOf(END_LINE);
            if (eol >= 0) {
                // if there is a new line, the message is the buffer + incoming.content() split at newline
                var line = buff + incoming.content().substring(0, eol);
                incoming.tell(new LineRead(line));
                // overwrite `buff` with the rest of the buffer (after the newline)
                var newBuff = incoming.content().substring(eol + 2);
                yield Become(clientSocketHandler(self, clientManager, channel, newBuff));
            } else {
                // otherwise, concatenate the entire incoming buffer
                var newBuff = buff + incoming.content();
                yield Become(clientSocketHandler(self, clientManager, channel, newBuff));
            }
            // other cases...
    };
}
```

When the recursive call is invoked, then the channel is subscribed again for reading. We can further simplify the `case` as such:


```java
case ReadBuffer incoming -> {
    // look for a new line
    int eol = incoming.content().indexOf(END_LINE);
    String newBuff; 
    if (eol >= 0) {
        // if there is a new line, the message is the buffer + incoming.content() split at newline
        var line = buff + incoming.content().substring(0, eol);
        incoming.tell(new LineRead(line));
        // overwrite `buff` with the rest of the buffer (after the newline)
        newBuff = incoming.content().substring(eol + 2);
    } else {
        // otherwise, concatenate the entire incoming buffer
        newBuff = buff + incoming.content();
    }
    yield Become(clientSocketHandler(self, clientManager, channel, newBuff));
}
```

This actor will also handle *writing* to the channel:

```java
record WriteLine(String payload) {}

static Behavior clientSocketHandler(Address self, Address parent, Channels.Socket channel, String buff) {
    ...
    msg -> switch (msg) {
        ...
        case WriteLine line -> {
            channel.write(line.payload());
            return Stay;
        }
        default -> throw new RuntimeException("Unhandled message " + msg);
    };
}
```

For convenience, you may want to add the overload without the `buff` argument:

```java
static Behavior clientSocketHandler(Address self, Address parent, Channels.Socket channel, String buff) {
    return clientSocketHandler(self, parent, channel, "");
}
```

and this is the one that gets invoked upon creation of the actor in `serverSocketHandler`. 
You can also make the other overload `private`.


### `clientManager`

A `clientManager` is notified when a new client is connected, and it receives lines that are read from the input stream.
Because the Client Manager keeps track of all Client Output actors, it is able to forward the `LineRead` to all of their outputs, i.e. their respective `clientSocketHandler` actor; this way all connected clients will receive a copy of the message.

![clientManager forwards ServerMessage to all the connected clients](/assets/actor-2/message.gif)


```java
    static Behavior clientManager(Address self) {
        var clients = new ArrayList<Address>();
        return msg -> {
            switch (msg) {
                case ClientConnected cc -> clients.add(cc.addr());
                case LineRead lr ->
                        clients.forEach(client -> client.tell(new WriteLine(lr.payload())));
                default -> throw new RuntimeException("Unhandled message " + msg);
            }
            return Stay;
        };
    }
```



You can now start your server with JBang! Add the following lines at the top of your file:

```java
//JAVA 17
//JAVAC_OPTIONS --enable-preview --release 17
//JAVA_OPTIONS  --enable-preview
//REPOS jitpack=https://jitpack.io/
//SOURCES Channels.java
//DEPS com.github.evacchi:min-java-actors:main-SNAPSHOT
```

If everything is right, then you can type:

```sh
j! ChatServer.java
```

If you are lazy, you can run it from this URL directly: 

```
j! ....
```

The program will start waiting for incoming connections, printing:

```sh
Server started at 0.0.0.0/0.0.0.0:4444.
```

Neat, huh?


## Chat Client

Of course, we are not done yet. We still need to write the client app.
Luckily that is incredibly short: we only need 2 actors. To be fair, you may just dedicate one thread to each and call it a day, but for completeness, let's use the actor system: the result will be quite compact. 

We will define:

- an actor that *reads* from and *writes* to the server socket
- an actor that handles the messages from user input, and display incoming messages to screen

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

Let us also define a functional interface for I/O; because serialization/deserialization methods throw `IOException`s we define an `IOBehavior` that throws: 

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
var myActor = sys.actorOf(self -> IO(msg -> /* IO-throwing body */));
```

There is only two types of messages:

```java
record ClientConnection(Channels.Socket socket) { }
record Message(String user, String text) {}
```

In this case, the `Message` contains the nickname of the user who wrote the message, and the actual text of the message.
Let us now write the `main` routine with the initialization logic and the main user input loop.

```java
Actor.System system = new Actor.System(Executors.newCachedThreadPool());

static void main(String[] args) throws IOException {
    // take the user name from the CLI args
    var userName = args[0];

    var channel = new Channels.Socket(AsynchronousSocketChannel.open());
    // start the `client` actor in `connecting` state.
    var client = system.actorOf(self -> clientConnecting(self, channel));

    out.printf("User '%s' connecting...", userName);

    // read user input line-by-line
    var scann = new Scanner(in);
    while (true) {
        var line = scann.nextLine();
        if (line != null && !line.isBlank()) {
            // for each non-empty line, send a message 
            // with the userName and the body
            client.tell(new Message(userName, line));
        }
    }
}
```

The only thing missing are now the 2 actors!

### Client

The client actor has two states: 

- connecting
- ready

We will model these using two behaviors. The first behavior `clientConnecting` connects to the socket
and when the connection is established it `Become`s `ready`.

If any `Message` is received while connecting, it is rejected with an error message.

```java
static Actor.Behavior clientConnecting(Address self, Channels.Socket channel) {
    channel.connect()
            .thenAccept(skt -> self.tell(new ClientConnection(skt)))
            .exceptionally(err -> { err.printStackTrace(); return null; });
    return msg -> switch (msg) {
        case ClientConnection conn -> {
            var socket = /// clientSocketHandler ?
            yield Become(clientReady(self, socket));
        }
        case Message m -> {
            err.println("Socket not connected");
            yield Stay;
        }
        default -> throw new RuntimeException("Unhandled message " + msg);
    };
}
```

you will notice that we need an actor to handle the socket. This is 100% identical to the one we wrote for the server. We can actually move that code to the `Channels` shared library and share it across the two implementations!

```java
interface Channels {
    ...
    interface Actor {
        record LineRead(String payload) {}
        record WriteLine(String payload) {}
        record ReadBuffer(String content) {}

        static Behavior socketHandler(Address self, Address parent, Channels.Socket channel) {
            return socketHandler(self, parent, channel, "");
        }
        private static Behavior socketHandler(Address self, Address parent, Channels.Socket channel, String acc) {
            channel.read()
                    .thenAccept(s -> self.tell(new ReadBuffer(s)))
                    .exceptionally(err -> { err.printStackTrace(); return null; });

            return msg -> switch (msg) {
                case ReadBuffer buffer -> {
                    var line = acc + buffer.content();
                    int eol = line.indexOf(END_LINE);
                    if (eol >= 0) {
                        parent.tell(new Channels.Actor.LineRead(line.substring(0, eol)));
                        yield Become(socketHandler(self, parent, channel, line.substring(eol + 2).trim()));
                    } else yield Become(socketHandler(self, parent, channel));
                }
                case WriteLine line -> {
                    channel.write(line.payload());
                    yield Stay;
                }
                default -> throw new RuntimeException("Unhandled message " + msg);
            };
        }
    }
}
```

then update:

```java
case ClientConnection conn -> {
    var socket = 
            system.actorOf(ca -> Channels.Actor.socketHandler(ca, self, conn.socket()));
    yield Become(clientReady(self, socket));
```

you can now do the same for the `serverSocketHandler` and get rid of a bunch of duplicate code:

```java
case ClientConnection conn -> {
    out.println("Child connected!");
    var client =
            system.actorOf(ca -> Channels.Actor.socketHandler(ca, childrenManager, conn.socket()));
    ...
}
```

Finally, here is the code for `ready`:

```java
static Actor.Behavior clientReady(Address self, Address socket) {
    out.println("Connected.");
    var mapper = new ObjectMapper();

    return IO(msg -> switch (msg) {
        case Message m -> {
            var jsonMsg = mapper.writeValueAsString(m);
            socket.tell(new Channels.Actor.WriteLine(jsonMsg));
            yield Stay;
        }
        case Channels.Actor.LineRead lr -> {
            var message = mapper.readValue(lr.payload(), Message.class);
            out.printf("%s > %s\n", message.user(), message.text());
            yield Stay;
        }
        default -> throw new RuntimeException("Unhandled message " + msg);
    });
}
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
//SOURCES Channels.java
```

If everything is right, then you can type:

```sh
j! ChatClient.java your-nickname
```

If you are lazy, you can run it from this Gist directly (make sure the server is running!): 

```
j! ..... your-nickname
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

In this post we have learned how to write a simple chat app. 

For simplicity, we used the [blocking Socket API][socket]. As an exercise, you can try to develop your own version using [Java NIO asynchronous APIs][asyncsocket].

As promised in the [previous post][minjavactors], in the final part of this series we will revisit the actor runtime and define a **fully-typed** version, which will benefit from exhaustiveness checks!

I am also happy to announce that [I have been selected](https://twitter.com/JavaAdvent/status/1457409222048636940) for the [Java Advent Calendar 2021][javaadvent], so the last part of this series will be first published on the [Java Advent Calendar][javaadvent] website! [Follow them on Twitter](https://twitter.com/JavaAdvent) for updates!

See you there!

[andrea]: https://twitter.com/and_prf
[minjavactors]: https://evacchi.github.io/java/records/jbang/2021/10/12/learn-you-an-actor-system-java-17-switch-expressions.html
[jbang]: https://jbang.dev
[socket]: https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/net/ServerSocket.html
[asyncsocket]: https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/nio/channels/AsynchronousServerSocketChannel.html
[loom]: https://inside.java/tag/loom
[javaadvent]: https://www.javaadvent.com/