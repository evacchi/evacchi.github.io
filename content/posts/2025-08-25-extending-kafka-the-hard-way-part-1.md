---
title: 'Extending Kafka the Hard Way (Part 1)'
author: 'Edoardo Vacchi'
date: 2025-08-25
tags: [Kafka, Chicory, WebAssembly]
---

<img src="/assets/kafka/kafka-wasm.jpg" alt="Picture of Franz Kafka looking surprised at the Wasm Logo" width="100%">

> 💡 It is worth noticing [Redpanda pioneered in-broker Wasm data transform](https://docs.redpanda.com/current/develop/data-transforms/how-transforms-work/) and this blog series is largely inspired by that work! Check it out!

You might have read my article about [plugging Wasm into Kafka Connect](https://www.getxtp.com/blog/pluggable-stream-processing-with-xtp-and-kafka), but I wanted to revisit this from a different angle: what if instead of **consuming data** from a topic and **manipulating such data on the client-side,** we **intercepted** data as it lands **on the Kafka broker?**

But first of all, **is it possible to extend the Kafka broker at all**?

Well, **yes and no**. There are indeed some predefined hooks where we can plug custom Java code; in particular, there are places where you can plug your custom Java policy to validate pieces of data; but, **at this time**, **these extension mechanisms do *not* deal with records.** 

The Kafka broker has supported custom user-defined code [since at least version 0.10.0](https://cwiki.apache.org/confluence/display/KAFKA/KIP-108%3A+Create+Topic+Policy) with the interface [**CreateTopicPolicy**](https://kafka.apache.org/10/javadoc/org/apache/kafka/server/policy/CreateTopicPolicy.html)**, ** which essentially amounts to implementing **one method:**

```java
void validate(CreateTopicPolicy.RequestMetadata requestMetadata) 
   throws PolicyViolationException
```

This policy **intercepts all the topic creation requests performed by an admin**, injecting custom validation logic. In other words, the code you plug here will be executed every time someone invokes the standard utility `kafka-topic.sh` to create a new topic.

```shell
kafka-topics.sh --topic my-new-topic --create
```

Users define their own policy by implementing the `CreateTopicPolicy` interface, then they can package their implementation as a jar, and finally, provided that that jar will be available on the classpath at run-time, they can **specify the class name** using the broker config property `create.topic.policy.class.name`.

This is the perfect spot to plug the [Chicory Extism SDK](https://github.com/extism/chicory-sdk/): [Chicory](https://chicory.dev) is **just plain, simple Java, but with a twist** that proves essential when you are hosting user-defined code: **sandboxing and **_guaranteed_** interruptability.** This is what we will show in this blog post.

## That's all fine and dandy, but what about records?

In the past, there were a **number of attempts to support record validation or transformation in the broker**, but **none landed on the trunk**. For instance, in the order in which they were proposed:

* [**KIP-686: API to ensure Records policy on the broker**](https://cwiki.apache.org/confluence/display/KAFKA/KIP-686%3A+API+to+ensure+Records+policy+on+the+broker)
* [**KIP-729: Custom validation of records on the broker prior to log append**](https://cwiki.apache.org/confluence/display/KAFKA/KIP-729%3A+Custom+validation+of+records+on+the+broker+prior+to+log+append)
* [**KIP-905: Broker interceptors**](https://cwiki.apache.org/confluence/display/KAFKA/KIP-905%3A+Broker+interceptors)
* [**KIP-940: Broker extension point for validating record contents at produce time**](https://cwiki.apache.org/confluence/display/KAFKA/KIP-940%3A+Broker+extension+point+for+validating+record+contents+at+produce+time)

In many cases, **the proponents** eventually **did not follow through** with their idea. One key issue is that supporting this kind of extension means **running user-defined code** on **one of the hottest paths in the broker.**

Not only is this controversial, but it is also **highly risky** because user-defined Java code might fail, go haywire, and, depending on how the code was written, even be hard to interrupt abruptly.

But the same is not true for Chicory. A Chicory runtime can be **always** interrupted. 

So, **in the next post of this series,** we will walk you through a **PoC where we patch the Kafka broker** and expose a new interface in the style of `CreateTopicPolicy`. **We will then implement this interface using Chicory** to support **interruptible, sandboxed data transforms.** The new `ProduceRequestInterceptor` will, again, expose essentially one method:

```java
RecordBatch intercept(RecordBatch batch) 
   throws ProduceRequestInterceptorException;
```

In practice, the basic **design** **principles for our patch** will be the same as those for  `CreateTopicPolicy` (or any other Java extension point that is *already * present in Kafka). And, when we will plug **Wasm support,** we will follow the same pattern that we will detail in this blog post for `CreateTopicPolicy`.

Consider this as your way to get your feet wet.

## Nice interface, it would be a shame if someone plugged some Wasm in it

So, let's start simple. In this blog post we will implement the `CreateTopicPolicy` interface with Chicory, we will plug a Wasm module, and implement one such policy using Wasm and Go.

We will also show that even a malicious payload can be given a timeout and killed abruptly at any time.

`CreateTopicPolicy` implements the `Configurable`  interface, which means you can also implement a method to configure your policy. This method will be passed the config properties of the broker. It is safe to assume that this method will be invoked before any topic creation request is handled; so for all intents and purposes, this may act as our initializer.

For instance, we could look for the path of Wasm binary at `create.topic.policy.wasm.path` and instantiate it. If you have read [the previous blog post, you will notice a pattern.](https://www.getxtp.com/blog/pluggable-stream-processing-with-xtp-and-kafka)

```java
public class WasmCreateTopicPolicyHandler implements CreateTopicPolicy {

    private WasmCreateTopicPolicy policy;

    @Override
    public void configure(Map<String, ?> configs) {
        // Get the config policy
        String wasmPath = configs.get("create.topic.policy.wasm.path").toString();
        try {
            // Read our wasm file and instantiate a Manifest.
            Path path = Path.of(wasmPath);
            FileInputStream inputStream = new FileInputStream(wasmPath);
            var manifest = new WasmTopicPolicyManifest(
                    inputStream, path.getFileName().toString(), Map.of());
            // set the policy
            this.policy = WasmCreateTopicPolicy.fromManifest(manifest);
        } catch (IOException e) {
            throw new ConfigException("Invalid Wasm binary", e);
        }
    }

    @Override
    public void validate(RequestMetadata requestMetadata) throws PolicyViolationException {
        policy.validate(requestMetadata);
    }

}
 ```

`WasmCreateTopicPolicy` is, [like in the article](https://www.getxtp.com/blog/pluggable-stream-processing-with-xtp-and-kafka), just a wrapper around an Extism plugin, exposing the `validate` method. 

Alright, that's all we need to get started. Now let's define our **validation extension point.**

> 💡 For simplicity, in this example, instead of calling into the XTP service, we assume the plugin is available somewhere on disk, but the same loading strategy we demonstrated in the previous article could be also applied here, potentially allowing you to reload strategies from the network and on the fly. Be mindful, however, that you might not want to clog your broker by repeatedly polling an HTTP service.

## XTP? More like ex-tee-policy am I right

In this case, the XTP schema needs very little design. It's just a matter of porting over the Java interface, including the [`RequestMetadata`](https://kafka.apache.org/10/javadoc/org/apache/kafka/server/policy/CreateTopicPolicy.RequestMetadata.html) data structure:

```yaml
exports:
  validate:
    input:
      $ref: "#/components/schemas/RequestMetadata"
      contentType: application/json
version: v1-draft
components:
  schemas:
    RequestMetadata:
      properties:
        topic:
          type: string
          description: the name of the topic to create
        configs:
          type: object
          description: topic configs in the request, not including broker defaults
        numPartitions:
          type: integer
          description: the number of partitions to create or null if replicaAssignments is not null.
        replicationFactor:
          type: integer
          description: the number of replicas to create or null if replicaAssignments is not null
        replicasAssignments:
          type: object
      description: a map from partition id to replica (broker) ids or null if numPartitions and replicationFactor are set instead.
```

We can now write our policy plugin with `xtp plugin init`. I named it `create-topic-policy-example`. Here's my `main.go`:

```go
package main

import (
	"errors"
	"strings"
)

func Validate(input RequestMetadata) error {
	if strings.Contains(input.Topic, "__INVALID__") {
		return errors.New("contained the magic keyword `__INVALID__`")
	}
	return nil
}
```

This simple plug-in intercepts all topic creation requests and rejects all those that contain the text `__INVALID__`. Build your plugin with `xtp plugin build` and then configure the broker with the newly created property to point to your wasm binary:

```
create.topic.policy.wasm.path=/path/to/create-topic-policy-example/dist/plugin.wasm
```

Start your Kafka broker, and if everything goes as planned, you can now try to create a topic with the magic string:

```
❯ bin/kafka-topics.sh --bootstrap-server localhost:9092 \
  --topic my__INVALID__topic --create
Error while executing topic command : Wasm Policy rejected the topic 
'my__INVALID__topic' with error 'Error evaluating function validate: 
Topic rejected, contained the magic keyword '__INVALID__''
```

while all other topics will work as expected

```shell
❯ bin/kafka-topics.sh --bootstrap-server localhost:9092 \
	--topic my-topic --create
Created topic my-topic.
```

There's no trick here! The policy is evaluated **by the broker,** and the error is returned **to the client.**

## Killing me softly

Now what if our policy goes rogue? Let's modify our plug-in as follows:

```java
func Validate(input RequestMetadata) error {
	for {
		// infinite loop
	}
	return nil
}
```

The Chicory runtime handles thread interruption requests, so execution can be safely interrupted at any time. Let's modify our `validate()` routine with a timeout:

```java
public void validate(RequestMetadata requestMetadata) throws PolicyViolationException {
    try {
        CompletableFuture
                .runAsync(() -> this.policy.validate(requestMetadata))
                .get(500, TimeUnit.MILLISECONDS);
        } catch (InterruptedException | TimeoutException e) {
            throw new PolicyViolationException(
                    "Policy evaluation exceeded the given time limit. " +
                            "Default to reject.", e);
    } catch (ExecutionException e) {
       // handle the execution exception and turn it into a PolicyViolationException
       throw new PolicyViolationException(...);
    }
}
```

Now the policy will be terminated after a maximum of 500 milliseconds.

```
❯ bin/kafka-topics.sh --bootstrap-server localhost:9092 --topic my_topic --create
Error while executing topic command : Policy evaluation exceeded the given time 
limit. Default to reject.
```

Congrats! You just plugged Wasm in a Kafka broker!

> 💡 **Rough around the edges.** The example we have written is not thread-safe because it reuses a single Wasm instance against all requests. If multiple requests were to be served at the same time, we might corrupt the Wasm memory. You might want to create a **pool of Wasm plug-ins** and reuse them to serve distinct validation requests.

## Conclusions

I hope you enjoyed this introduction to extending the Kafka broker with Chicory and Extism. In the next post, we will detail the delicate path where the Kafka broker handles incoming records.

Brace for some open-heart surgery on the core of Kafka!