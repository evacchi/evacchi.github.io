---
title: 'Extending Kafka the Hard Way (Part 2)'
author: 'Edoardo Vacchi'
date: 2025-09-03
tags: [Kafka, Chicory, WebAssembly]
cover: /assets/kafka/kafka-wasm-2.jpg
---

<img src="/assets/kafka/kafka-wasm-2.jpg" alt="Picture of Franz Kafka looking surprised at the Wasm Logo" width="100%">

In [our old post about Kafka](https://www.getxtp.com/blog/pluggable-stream-processing-with-xtp-and-kafka) we described how to write a self-contained application to apply arbitrary transforms to an incoming stream of Kafka records and publish the result on a new topic.

In the [**previous post**](/posts/2025/08/25/extending-kafka-the-hard-way-part-1/) of this new series, we decided to embark on a new journey, towards embedding data transforms within **the broker**. In this case, instead of subscribing to a stream of records and applying transforms on the client-side, transforms would be applied **directly on the broker.**

Unfortunately, as we have seen earlier, at this time, there is no way to plug into the broker some custom behavior, unless you want to roll up your sleeves and patch it.

But this did not discourage us: in fact, the Kafka broker supports hooking custom code; it just does not support hooking *that specific code path.* So, **in the previous post, we described how to implement a Wasm policy for topic creation**, but we promised we would revisit this matter and build on top of that experience.

Well, that time has come.

## Topics and Partitions

If you are familiar with Kafka, you may already know that you write records to **named topics,** and each topic can be further broken down into **partitions**. 

A partition is identified by an integer **index**. Records are ultimately written to a specific pair `(topic-name, partition-index)`. High availability and resiliency are achieved by **replicating** such partitions. Possibly counterintuitively, ***clients*** **determine the partition index**: even if you, as a user, never made use of this feature explicitly, the index of the partition is **always computed on the client-side.** 

In fact, it is even possible to [customize the partitioning strategy](https://kafka.apache.org/documentation/#consumerconfigs_partition.assignment.strategy); if you don't do it explicitly, normally a default strategy is applied anyway. For instance, in the presence of a key, the index is determined by hashing the key.

Even though the interface of a `KafkaProducer` might give you the impression that records are processed one at a time, the primary unit of transfer from a client to a broker is the **ProduceRequest**. A produce request contains many **batches of records,** collected by `(topic-name, partition-index)`. A **ProduceRequest** may contain **any number of batches**. 

This already poses an interesting challenge in terms of API design. What does it mean for the broker to apply a data transform on a record?

## Intercepting records

The community has shown great interest in adding support to in-broker computations over records; in fact, in the past, multiple proposals were submitted for evaluation:

* [**KIP-686: API to ensure Records policy on the broker**](https://cwiki.apache.org/confluence/display/KAFKA/KIP-686%3A+API+to+ensure+Records+policy+on+the+broker)
* [**KIP-729: Custom validation of records on the broker prior to log append**](https://cwiki.apache.org/confluence/display/KAFKA/KIP-729%3A+Custom+validation+of+records+on+the+broker+prior+to+log+append)
* [**KIP-905: Broker interceptors**](https://cwiki.apache.org/confluence/display/KAFKA/KIP-905%3A+Broker+interceptors)
* [**KIP-940: Broker extension point for validating record contents at produce time**](https://cwiki.apache.org/confluence/display/KAFKA/KIP-940%3A+Broker+extension+point+for+validating+record+contents+at+produce+time)

None of them ended up completed, but in our opinion, the most promising attempts were [**KIP-905**](https://cwiki.apache.org/confluence/display/KAFKA/KIP-905%3A+Broker+interceptors) and [**KIP-940**](https://cwiki.apache.org/confluence/display/KAFKA/KIP-940%3A+Broker+extension+point+for+validating+record+contents+at+produce+time).

[**KIP-905**](https://cwiki.apache.org/confluence/display/KAFKA/KIP-905%3A+Broker+interceptors) only deals with **record validation**: in other words, similarly to our previous post, it returns an error in case a record is not valid, causing the error to be propagated to the original producer on the client-side. 

[KIP-940](https://cwiki.apache.org/confluence/display/KAFKA/KIP-940%3A+Broker+extension+point+for+validating+record+contents+at+produce+time) is more powerful but also more impactful: it allows **transforming**, **rejecting** (for validation), and **filtering out** records. Moreover, it allows to **register** **multiple interceptors:** each interceptor **declares a record topic pattern** (using a regular expression). 

**Both proposals apply to single records**, instead of working on the batch level.

In this blog post, **we implement a variant of** [**KIP-940**](https://cwiki.apache.org/confluence/display/KAFKA/KIP-940%3A+Broker+extension+point+for+validating+record+contents+at+produce+time), but we take a different approach in some key aspects:

* The interceptor is not applied to single a **record,** but to **batches of records for a given pair** `(topic-name, partition-index)`.
* Only **one interceptor** can be hooked per-broker: it will intercept **all ProduceRequests**. Internally, it might ignore that request and return the batch of records unchanged.
* The interceptor is allowed to **reject** **an entire batch of records** for validation reasons; this was also the behavior in KIP-940, even though the API gave the impression of per-record processing
* The interceptor is allowed to **apply a transform to a batch of records,** resulting in the **creation of a new batch of records**: the **number of records** in the resulting batch **MUST** be the **same number of records in the input** batch. 

It follows that in our approach

1. **it is** ***not*** **possible to** ***filter out*** **records**
2. **it is** ***not*** **possible to produce** ***more records than those that were submitted***. 

This is because the clients assume that producing a batch of N records, results in **acknowledging exactly N** records being stored. In other words, if a client produces **10 records** in a batch, then the broker **cannot acknowledge** **only 9** or, worse, **11.**

In fact, KIP-940 required a change in the client code to account for filtered-out records.

## Writing to a new topic

It follows that this approach only partially overlaps with [our first post about Kafka](https://www.getxtp.com/blog/pluggable-stream-processing-with-xtp-and-kafka), where

1. we allowed processing one record and **returning 0..n records** as a result
2. we received one record from one topic and **published the result to another topic**
3. however, because we subscribed to an **existing topic**, we **did not support** **rejecting** a record; we could only ***discard it*** and produce 0 records as a result.

Moreover, because the Kafka protocol deals with pairs of `(topic-name, partition-index)` it would be also **unclear how to receive data from one topic and write the result on a** ***new*** **topic**, because the problem is ill-defined: the broker ***receives*** ***batches*** **of records** for each `(topic-name, partition-index)` pair.

It follows that our interceptor should ***produce*** ***batches*** **of records** for a new `(topic-name, partition-index)`; but, because the **partition-index** is computed using a client-defined strategy, our transforms should be also configured somehow to compute that index too.

Moreover, this poses another interesting challenge: if we intercept a **ProduceRequest** on a given broker, we can verify that the pair  `(topic-name, partition-index)`  is valid for the current broker; i.e., that that partition exists on that broker. However, in order to write to a new pair  `(topic-name', partition-index')` , we need to know where that pair is: that pair **might exist** on the same broker, or it might exist only on a **different broker**: in that case it would mean we would have to propagate the resulting batch through the network. 

For instance, suppose that a request lands on broker **b1** with a batch for topic **t1**, partition **p1**. Imagine, that an interceptor generates a new batch for topic **t2**, partition **p2**, with **t1≠t2, p1≠p2**. If **t2,p2** is not stored on **b1**, we would have to look for a broker **b2** that is leader for **t2,p2!**

This is in fact doable, but it poses further challenges in terms of acknowledging the request: should we wait for an acknowledgment from **b2?** What kinds of constraints should we pose on the **batches of records?** Should the size still match? etc.

All of this is doable, but it would go well beyond the scope of this blog post, so we decided to keep it simple.

## Alright, enough with the chit-chat, can we do some coding now?

As opposed to the previous post, where we had an interface already available for implementation, in this post we will have to do all the heavy lifting ourselves. 

So let's start with defining an interface for the interceptor. This interface will be implemented in the `kafka-clients` module, as it is the only Kafka core module published to Maven Central.

```java
package org.apache.kafka.server.intercept;

// imports...

public interface ProduceRequestInterceptor extends Configurable, AutoCloseable {
    RecordBatch intercept(RecordBatch batch) 
      throws ProduceRequestInterceptorException;
}
```

Unfortunately, in the broker, all the classes dealing with sets of records are either non-public or they are marked as subject to change. So, if we want to do things right, we should provide our own `RecordBatch`. We also define a low-level `Record` interface for the same reason.

`TopicPartition` and `Header` are classes you should be already familiar with, if you ever wrote some Kafka-related code.

```java
    interface RecordBatch {
        TopicPartition topicPartition();
        Iterable<? extends Record> records();
    }

    interface Record {
        long timestamp();
        ByteBuffer key();
        ByteBuffer value();
        Header[] headers();
    }
```

We should now implement a manager for such `ProduceRequestInterceptor`.

First of all, our `ProduceRequestInterceptorManager` should load the configured `ProduceRequestInterceptor`. We might handle this directly in its constructor. We can use the `KafkaConfig` utility to automatically instantiate a class from its name: 

```java
public ProduceRequestInterceptorManager(KafkaConfig config) {
    this.interceptor = config.getConfiguredInstance(
            "produce.request.interceptor.class.name", 
            ProduceRequestInterceptor.class);
}
```

Then, for each topic, partition pair in a **ProduceRequest** we want to process the records with our interceptor:

```java
public Map<TopicPartition, PartitionResponse> intercept(ProduceRequest request) {
    var errors = new HashMap<TopicPartition, PartitionResponse>();
    ProduceRequestData requestData = request.data();
    var topicProduceData = requestData.topicData();
    for (var tpd : topicProduceData) {
        for (var ppd : tpd.partitionData()) {
            var topicPartition =
                    new TopicPartition(tpd.name(), ppd.index());
            try {
                // This cast to an internal class is safe to do in the broker
                // even though it's an implementation detail.
                // This is a collection of batches of records
                // This class won't be available for clients, because
                // it is not exported.
                var inputRecords = (MemoryRecords) ppd.records();

								// Apply the interceptor to the records. 
                MemoryRecords memoryRecords = 
                  interceptBatch(topicPartition, inputRecords);

                ppd.setRecords(memoryRecords);
            } catch (ProduceRequestInterceptorException ex) {
                errors.put(topicPartition,
                        new PartitionResponse(Errors.forException(ex)));
            }
        }
    }
    return errors;
}    
```

And the `interceptBatch()` method:

```java
  private MemoryRecords interceptBatch(
		  TopicPartition topicPartition, MemoryRecords inputRecords) 
          throws ProduceRequestInterceptorException {
      // Wrap the internal class into a our own wrapper exposing MemoryRecords 
      // using our new public API
      var inputBatch = new MemoryRecordBatch(topicPartition, inputRecords);

      // Apply the interceptor
      RecordBatch outputBatch = interceptor.intercept(inputBatch);

      // Convert back to Memory records
      // Using an utility class of our own 
      // (omitted here for brevity)
      return Conversions.asMemoryRecords(outputBatch);
  }
```

> 💡 Technically the `MemoryRecords` class wraps a **collection of batches**. For simplicity, we will treat this as a **single batch of records** (the union of all the records in all the batches).
>
> Also notice that batches may be optionally compressed. Compression is applied to individual batches, so each batch may be individually compressed, but the compression type is specified at the `MemoryRecords` level. 
>
> In order to apply the transform to a compressed batch, we would need to decompress it first. For simplicity, we assume that record batches are not compressed, and we will return an uncompressed result.

… and that's pretty much it. Now we only need to wire-in our brand-new **ProduceRequestInterceptorManager.** This is the part where we actually patch the Kafka broker.

Open [scala/kafka/server/KafkaApis.scala](https://github.com/apache/kafka/blob/b213c64f97fa6b4ab3b79b828e37095603841d6f/core/src/main/scala/kafka/server/KafkaApis.scala#L136), and in the class body, you can add a new field:

```scala
val produceRequestInterceptorManager = 
   new ProduceRequestInterceptorManager(config)
```

Congratulations! You just patched the Kafka broker! Aren't you proud?

> 💡 The code you will find online is slightly more complicated because it properly documents the new config keys, and it wires the `ProduceRequestInterceptorManager` in other places (mostly object builders and test cases) as needed. 

## Dude, Where's My Wasm?

**— This blog post is a travesty!** I was promised some Wasm, some Chicory, and a dash of XTP, and all I'm getting is a lousy Kafka patch.

We hear ya, so here's your cup of Chicory ☕️ 

If you have read the previous blog post, the new  `TransformManager` will almost feel underwhelming. Let's say each transform is registered to handle a topic, we can define a map from `String` (the topic name) to our `Transform`:

```java
public class TransformManager implements ProduceRequestInterceptor {
    private final ConcurrentHashMap<String, Transform> transforms = 
		    new ConcurrentHashMap<>();
    ...
}
```

Then, since our `ProduceRequestInterceptor` implements `Configurable` (like a `CreateTopicPolicy`) we can pull the configured Wasm interceptor from config keys. For instance, we could read from a property a comma-separated list of topic names:

```
produce.request.interceptor.wasm.topics=topic1,topic2,...
```

and then expect a config key `produce.request.interceptor.wasm.$topic.path` to be defined for each topic in the list. Again, for simplicity, we will assume this is a path on disk, but nothing prevents you from implementing fancier strategies.

```java
public void configure(Map<String, ?> configs) {
    List<String> topics = 
        parseList(configs.get("produce.request.interceptor.wasm.topics"));
    for (var topic : topics) {
        var path = configs.get(
            "produce.request.interceptor.wasm."+topic+".path");
        var inputStream = new FileInputStream(path);
        var manifest = new TransformManifest(inputStream, pluginName, topic);
        var transform = Transform.fromManifest(manifest);
        transforms.put(topic, transform);    
    }
}
```

Notice this is pretty much the same `Transform` that we saw [in our earlier article](https://www.getxtp.com/blog/pluggable-stream-processing-with-xtp-and-kafka), even though the Wasm interface will be slightly different. In fact, even though the **interceptor** works at the **RecordBatch** level, nothing prevents us from exposing a **per-record interface to our Wasm transforms!**

The transform, as usual, is pretty much a wrapper for an Extism Chicory SDK `Plugin`:

```java
public class Transform {
    ...
	  public Record transform(Record record) {
		    // InternalMapper wraps a Jackson mapper instance.
		    byte[] in = InternalMapper.asBytes(record);
		    byte[] out = plugin.call("transform", recordBytes);
		    return InternalMapper.fromBytes(manifest.outputTopic(), out);
	  }
}
```

Now it's the moment of truth: let's implement `ProduceRequestInterceptor#intercept()`

```java
public RecordBatch intercept(RecordBatch batch) {
    Transform t = ktransform.get(batch.topicPartition().topic());
    // No transform has been registered for this topic.
    if (t == null) {
        return batch; // NO WASM FOR YOU!
    }
    // This utility class implements the `RecordBatch` interface
    // and it's implemented in the client library.
    var result = new SimpleRecordBatch.Builder(batch.topicPartition());
    for (var record : batch.records()) {
        result.append(t.transform(record));
    }
    return result.build();
}
```

### My XTPrecious

Let's now create our plug-in interface. In this example we will be exposing a simpler interface, where both the key and the value of a record are strings:

```
version: v1-draft
components:
  schemas:
    Header:
      properties:
        key:
          type: string
        value:
          type: string
      description: A key/value header pair.
    Record:
      properties:
        key:
          type: string
        topic:
          type: string
        value:
          type: string
        headers:
          type: array
          items:
            $ref: "#/components/schemas/Header"
      description: A plain key/value record.
exports:
  transform:
    input:
      $ref: "#/components/schemas/Record"
      contentType: application/json
    output:
      $ref: "#/components/schemas/Record"
      contentType: application/json
    description: |
      This function takes one Record and returns a single Record.
```

Notice that you are free to pick the schema you decide, because you are still in full control of the `TransformManager`. Even if an interface like `ProduceRequestInterceptor` were to be upstreamed to Kafka, you will always be able to define a domain-specific API for your Wasm plug-ins!

Now let's scaffold our transform. Say, let's convert to **uppercase** the value of a record and write it back. Fire up your `xtp plugin init` and create a plugin called `upper`, and, in your `main.go`:

```go
package main

import "strings"

func Transform(input Record) (Record, error) {
	input.Value = strings.ToUpper(value)
	return input, nil
}
```

Now launch `xtp plugin build` and you are ready to configure your broker:

```
produce.request.interceptor.class.name=com.dylibso.examples.kafka.transforms.TransformManager
produce.request.interceptor.wasm.topics=test-topic
produce.request.interceptor.wasm.test-topic.path=/path/to/upper/dist/plugin.wasm
```

Start up your broker and write anything to `test-topic`. Your transform will eat it up and SHOUT IT OUT LOUD! We can also write a validation routine. For instance, you could reject all records where the value is upper case:

```go
package main

import "strings"

func Transform(input Record) (Record, error) {
	original := input.Value
	upper := strings.ToUpper(original)
	if original == upper {
		return Record{}, errors.New("stop shouting, you are hurting my ears")
	} else {
		return input, nil
	}
}
```

Because the transform returns an error, it will automatically reject the entire batch. Reminds you something? That's right, it works the same as our earlier CreateTopicPolicy!

## Killing me softly (again)

If there is a critical place where we don't want throughput to suffer is the request handler. So let's make sure that interceptors can be killed. Essentially, we want to give a single, predictable timeout **per-request**. One way to do that is to instantiate an ExecutorService at the beginning of our `intercept(ProduceRequest request)` method, and then forcibly terminate execution after a timeout:

```java
public Map<TopicPartition, PartitionResponse> intercept(ProduceRequest request) {
    ProduceRequestData requestData = request.data();
    var futures = new ConcurrentHashMap<TopicPartition, Future<?>>();
    
    try (var svc = Executors.newSingleThreadExecutor()) {
        var topicProduceData = requestData.topicData();
        for (var tpd : topicProduceData) {
            for (var ppd : tpd.partitionData()) {
                var topicPartition =
                        new TopicPartition(tpd.name(), ppd.index());
                var inputRecords = (MemoryRecords) ppd.records();

								Future<?> f = svc.submit(() -> {
								    MemoryRecords memoryRecords = 
								       intercept(topicPartition, inputRecords);
								    // This is safe because the ExecutorService is single-thread
								    // and because each ppd operates on a separate 
								    // <topic, partition> pair.
                    ppd.setRecords(memoryRecords);
								});
                futures.put(topicPartition, f);
            }
        }
        svc.shutdown();
        if (!svc.awaitTermination(maxMs, TimeUnit.MILLISECONDS)) {
            svc.shutdownNow();
        }
    } catch (Throwable e) {
        LOGGER.warn("Execution timed out", e);
    }

		// Collect errors from all the `Future<?>`s.
    return collectErrors(futures);
}
```

Now try to write a malicious plugin:

```go
package main

func Transform(input Record) (Record, error) {
  for {
      // infinite loop!
  }
	return input, nil
}
```

In this case, the transform will time out, it will produce a timeout exception in the `ProduceRequestInterceptor` and the record will be rejected.

## Conclusions

We are at the end of our journey through the internals of Kafka's broker. We have successfully implemented a WebAssembly-powered record interceptor that runs directly within the broker, navigating Kafka's complex internals while maintaining compatibility with existing clients.

I hope you enjoyed the exploration of Kafka's internals as much as I did. I believe this feature has a lot of potential: you can define simple data transformations but also complex validation rules, always running in the safe, interruptible Chicory sandbox. 

Who knows? Maybe your use case will be the one that finally gets broker-side transforms into mainstream Kafka!
