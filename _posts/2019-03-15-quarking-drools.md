---
title:  Quarking Drools
subtitle: How and why we turned our 13 years old Java project into a first-class serverless component
date:   2019-03-16 
---

This post has been originally published on [Red Hat Developers blog](https://developers.redhat.com/blog/2019/03/14/quarking-drools-how-we-turned-a-13-year-old-java-project-into-a-first-class-serverless-component/) and on the [Drools and jBPM blog](http://blog.athico.com/2019/03/quarking-drools-how-we-turned-13-year.html)

> The question of whether a computer can think is no more interesting
than the question of whether a submarine can swim. (*Edsger W.
Dijkstra*)

## Motivation

Rule-based artificial intelligence (AI) is often overlooked, possibly because people think it’s only useful in heavyweight enterprise software products. However, that’s not necessarily true. Simply put, a rule engine is just a piece of software that allows you to separate domain and business-specific constraint from the main application flow. We are part of the team developing and maintaining Drools—the world’s most popular open source rule engine and part of Red Hat—and, in this article, we will describe how we are changing Drools to make it part of the cloud and serverless revolution.

## Technical Overview

Our main goal was to make the core of the rule engine lighter, isolated,
easily portable across different platforms, and well-suited to run in a
container. The software development landscape has changed a lot in the
last 20 years. We are moving more and more towards a polyglot world, and
this is one of the reasons why we are working towards making our
technology work across a lot of different platforms. This is one of the
reasons why we also started looking into
[*GraalVM*](https://www.graalvm.org/), the new Oracle Labs polyglot
virtual machine ecosystem consisting of 

  - A polyglot VM runtime, alternative to the JVM with a just-in-time
    (JIT) compiler that improves efficiency and speed of applications
    over traditional HotSpot. This is also the "proper" GraalVM
  - A framework to write efficient dynamic programming languages such as
    JavaScript, Python, and R and mix and match them together (Truffle)
  - A tool to compile programs ahead-of-time (AOT) into a native
    executable

Meanwhile at Red Hat, another team was already experimenting with
GraalVM and native binary generation for application development. This
effort has been realized in a new project you may have already heard of
called [*Quarkus*](http://quarkus.io/). The Quarkus project is a
best-of-breed Java stack that works on good ol' JVM, but it is also
especially tailored for GraalVM, native binary compilation and
cloud-native application development. 

Now, GraalVM is an amazing tool but it also come with a number of
(understandable)
[*limitations*](https://github.com/oracle/graal/blob/master/substratevm/LIMITATIONS.md)[.
T](https://github.com/oracle/graal/blob/master/substratevm/LIMITATIONS.md)herefore,
not only is Quarkus designed to integrate seamlessly with GraalVM and
native image generation, but it also provides useful utilities to
overcome any related limitations. In particular, Drools used to make
extensive use of dynamic class generation, class-loading, and quite a
bit of reflection. In order to produce fast, efficient and small native
executable, Graal performs aggressive inlining and dead-code
elimination, and it operates under a **closed-world assumption**; that
is, the compiler removes any references to class and methods that cannot
be statically reachable in the code. In other words, unrestricted
reflective calls and dynamic class loading are a no-go. Although this
may first sound like a showstopper, in the next 2 sections we will
document in detail how we modified the core of Drools to overcome such
limitations, and we will explain why such limitations are not evil and
actually liberating.

## The Executable Model

In a rule engine, **facts** are inserted into a **working memory.**
**Rules** describe **actions** to take when certain **constraints** over
the facts that are inserted into the working memory become **true**. For
instance, the sentence "**when** *the sun goes down ***: ***turn on the
lights*" expresses a rule over the sun. The *fact* is that the sun is
going down. The *action* is to turn on the lights. In a rule engine we
*insert *the "sun is going down" fact inside the working memory. When we
*fire* the rules the action of *turning on the lights* will execute.

A rule definition has the form

```
      <constraints> → <consequence>
```

the *constraints* part, also called the *left-hand side *of the rule,
describes the constraints that activate the rule and make it ready to
*fire*; the *consequence* part, also called the *right-hand side* of the
rule, contains the action that rule will take when the rule is fired. 

In Drools, a rule is written using the Drools Rule Language (in short,
DRL), and it has the form:

```java
rule R1 when
  $r : Result()                                // constraints
  $p : Person( age >= 18 )    
then
  $r.setValue( $p.getName() + " can drink");   // consequence
end
```

Constraints are written using a form of pattern-matching over the data
(Java objects) that is inserted into the working memory. Actions are
basically a block of Java code with a few Drools-specific extensions.

Historically, the DRL used to be a dynamic language that was interpreted
at runtime by the Drools engine. In particular, the pattern matching
syntax had a major drawback: it made extensive use of reflection unless
the engine detected a pattern was "hot" enough for further optimization;
that is, if it had evaluated a certain number of times; in that case the
engine would compile it into bytecode on-the-fly. 

About one year ago, for performance reasons, we decided to go away with
runtime reflection and dynamic code generation and completed the
implementation of what we called the [*Drools Executable
Model*](http://blog.athico.com/2018/02/the-drools-executable-model-is-alive.html),
providing a pure Java-based representation of a rule set, together with
a convenient Java DSL to programmatically define such model. 

To give an idea of how this Java API looks like let’s consider again the
simple Drools rule reported above. The rule will fire if the working
memory contains any Result instance, and instance of Person where the
age field is greater or equal to 18. The consequence is to set the value
of the Result object to a String saying that the person can drink. The
equivalent rule expressed with the executable model API looks like it
follows (pretty-printed for readability):

```java
var r = declarationOf(Result.class, "$r");
var p = declarationOf(Person.class, "$p");
var rule = 
  rule("com.example", "R1").build(
    pattern(r),
    pattern(p).expr("e", p -> p.getAge() >= 18),
    alphaIndexedBy(int.class, GREATER\_OR\_EQUAL, 1, this::getAge, 18),
    reactOn("age")),
    on(p, r).execute(($p, $r) -> 
      $r.setValue($p.getName() + " can drink")));
```

As you can see this representation is more verbose and harder to
understand, partially for the Java syntax, but mostly because it
explicitly contains lots of details, like the specification of how
Drools should internally index a given constraint, that were implicit in
the corresponding DRL. We did this on purpose because we wanted a
totally explicit rule representation that did not require any convoluted
inference or reflection sorcery. However we knew that it would have been
crazy to ask users to be aware of all such intricate details and that’s
why we wrote a compiler to translate DRL into the equivalent Java code.
We achieved this using [*JavaParser*](http://javaparser.org/), a really
nice open-source library that allows to parse, modify and generate any
Java source code through a convenient API.

In all honesty when we designed and implemented the executable model we
didn’t have strictly GraalVM in mind. We simply wanted an intermediate
and pure Java representation of the rule that could be efficiently
interpreted and executed by the engine. Yet, by completely avoiding
reflection and dynamic code generation, the executable model was key to
allow us to support native binary generation with Graal. For instance,
because the new model expresses all constraints as lambda predicates, we
don’t need to optimize the constraints evaluators through bytecode
generation and dynamic classloading which are totally forbidden in
native image generation.

The design and implementation of executable model taught us an important
lesson in the process of making Drools compatible with native binary
generation: any limitation can be overcome with a sufficient amount of
code generation. We will further discuss this in the next section.

## Overcoming other Graal limitations

Having a plain Java model of a Drools rule base was a very good starting
point, but there was still some work to be done to make our project
compatible with native binary generation.

The executable model makes reflection largely unnecessary; however, our
engine still needs reflection for one last feature called[
](http://docs.jboss.org/drools/release/7.17.0.Final/drools-docs/html_single/#_fine_grained_property_change_listeners)[*property
reactivity*](http://docs.jboss.org/drools/release/7.17.0.Final/drools-docs/html_single/#_fine_grained_property_change_listeners).
Our plans are to get rid of reflection altogether, but because the
change is nontrivial, for this time we resorted to a handy feature of
the binary image compiler, which* does* support a form of reflection,
provided that we can declare upfront the classes we will need to reflect
upon at runtime. This can be supplied by providing a[
](https://github.com/oracle/graal/blob/master/substratevm/REFLECTION.md#manual-configuration)[*JSON
descriptor*](https://github.com/oracle/graal/blob/master/substratevm/REFLECTION.md#manual-configuration)
file to the compiler, or, if you are using Quarkus, you can just[
](https://quarkus.io/guides/rest-json-guide)[*annotate the domain
classes*](https://quarkus.io/guides/rest-json-guide). For instance, in
the rule that we have shown above, our domain classes would be Result
and Person. Then we may write:

```json
[
  {
  "name" : "org.drools.simple.project.Person",
  "allPublicMethods" : true
  },
  {
  "name" : "org.drools.simple.project.Result",
  "allPublicMethods" : true
  }
]
```

and then instruct the native binary compiler with the flag

```
    -H:ReflectionConfigurationFiles=reflection.json
```

We segregated other redundant reflection trickery to a *dynamic* module,
and implemented an alternative *static* version of the same components
that users can choose to import in their project. This is especially
useful for binary image generation, but it has benefits for regular use
cases as well, in particular, avoiding reflection and dynamic loading
can result in faster startup time and improved run-time.

At startup time, Drools projects read an XML descriptor called the
*kmodule*, where the user declaratively defines the configuration of the
project. Usually we parse this XML file and load it into memory, but our
current XStream-based parser uses a lot of reflection; first of all we
can load the XML with an alternative strategy that avoids reflection;
but we can go further: if we can guarantee that the in-memory
representation of the XML will never change across runs, and we can
afford to run a quick code-generation phase before repackaging a project
for deployment, we can avoid loading the XML at each boot-up altogether.
In fact, we are now able to translate the XML file into a class file
that will be loaded at startup time, like any other hand-coded class.
Here's a comparison of the XML with a snippet of the generated code
(again, pretty-printed for readability). The generated code is more
verbose because it makes explicit all the configuration defaults.

```xml
<kbase name="simpleKB" packages="org.drools.simple.project">
<ksession name="simpleKS" default="true"/>
</kbase>
```

vs.

```java
var m = KieServices.get().newKieModuleModel();
var kb = m.newKieBaseModel("simpleKB");
kb.setEventProcessingMode(CLOUD);
kb.addPackage("org.drools.simple.project");
var ks = kb.newKieSessionModel("simpleKS");
ks.setDefault(true);
ks.setType(STATEFUL);
ks.setClockType(ClockTypeOption.get("realtime"));
```

Another issue with startup time, is dynamic classpath scanning. Drools
supports alternate ways to take *decisions* other than DRL-based rules,
such as *decision-tables, *the *Decision Model and Notation (DMN) *or
*predictive models *using the *Predictive Model Markup Language
*(*PMML*). Such extensions are implemented as dynamically loadable
modules, that are hooked into the core engine by scanning the classpath
at boot-time. Although this is extremely flexible, it is not essential:
even in this case, we can avoid runtime classpath scanning and provide
*static *wiring of the required components either by generating code at
build-time, or by providing an explicit API to end users to hook
components manually. We resorted to provide a pre-built static module
with a minimal core.

```java
private Map<Class<?>, Object> serviceMap = new HashMap<>();
private void wireServices() {
  serviceMap.put(ServiceInterface.class,
  Class.forName("org.drools.ServiceImpl").newInstance());
  // … more services here
}
```

Please notice that, although here we are using Class.forName(), the
compiler is smart enough to recognize the constant, and substitute it
with an actual constructor. Of course it is possible to simplify this
further by generating a chain of **if** statements. 

Finally, we tied everything together by getting rid of the last few
pre-executable model leftovers: the legacy Drools class-loader. This was
the culprit behind to this apparently cryptic error message:

```

Error: unsupported features in 2 methods
Detailed message:
Error: com.oracle.graal.pointsto.constraints.UnsupportedFeatureException: Unsupported method java.lang.ClassLoader.defineClass(String, byte[], int, int, ProtectionDomain) is reachable: The declaring class of this element has been substituted, but this element is not present in the substitution class
To diagnose the issue, you can add the option --report-unsupported-elements-at-runtime. The unsupported element is then reported at run time when it is accessed the first time.
Trace:
        at parsing org.drools.dynamic.common.DynamicComponentsSupplier$DefaultByteArrayClassLoader.defineClass(DynamicComponentsSupplier.java:49)
Call path from entry point to org.drools.dynamic.common.DynamicComponentsSupplier$DefaultByteArrayClassLoader.defineClass(String, byte[], ProtectionDomain):
```

But really, the message is pretty clear: our custom class-loader is able
to dynamically *define* a class; this is useful when you generate
bytecode at *run-time.* But if the codebase relies completely on the
executable model then we can avoid this altogether, so we isolated the
legacy class-loader into the *dynamic *module.

This is the last thing that was necessary to successfully generate a
native image of our simple test project and the results exceeded our
expectations confirming that the time and efforts we spent in this
experiment were well invested. Indeed, executing the main class of our
test case with a normal JVM takes 43 milliseconds with a occupation of
73M of memory. The corresponding native image generated by Graal is
timed at less than 1 millisecond and uses only 21M of memory.

## Integrating with Quarkus

Once we had a first version of Drools compatible with Graal native
binary generation, the next natural step was to start leveraging the
features provided by Quarkus and try to create a simple web service with
it. The first thing that we noticed is that Quarkus offers a different
and simpler mechanism to let the compiler know that we need reflection
on a specific class. In fact, instead of having to declare this in a
JSON file like we did before, it is enough to annotate the class of your
domain model as follows: 

```java
@RegisterForReflection
public class Person { … }
```

We also decided to go one small step forward with our code generation
machinery. In particular we added one small interface to Drools code

```java
public interface KieRuntimeBuilder {
  KieSession newKieSession();
  KieSession newKieSession(String sessionName);
}
```

so that when the Drools compiler creates the executable model from the
DRL files it also generates an implementation of this class. This
implementation has the purpose of supplying a Drools session
automatically configured with the rules and the parameters defined by
the user.

After that we were ready to put both dependency injection and REST
support provided by Quarkus at work and developed a simple web service
exercising the Drools runtime.

```java
@Path("/candrink/{name}/{age}")
public class CanDrinkResource {
  @Inject
  KieRuntimeBuilder runtimeBuilder;

  @GET
  @Produces(MediaType.TEXT_PLAIN)
  public String canDrink( 
    @PathParam("name") String name, 
    @PathParam("age") int age ) {
    KieSession ksession = runtimeBuilder.newKieSession();
    Result result = new Result();
    ksession.insert(result);
    ksession.insert(new Person( name, age ));
    ksession.fireAllRules();
    return result.toString();
  }
}
```

The example is straightforward enough to not require any further
explanation and is fully deployable as a microservice in an OpenShift
cluster. Thanks to the extremely low startup time, due to the work we
did on Drools and Quarkus' low overhead, this microservice is fast
enough to be deployable in a
[*KNative*](https://cloud.google.com/knative/) cloud. You can find the
full source code on
[*GitHub*](https://github.com/kiegroup/submarine-examples). 

## Introducing Submarine

These days, rule engines are seldom a matter of discussion. This is
because they *just work*. A rule engine is not necessarily antithetical
to a cloud environment, but work might be needed to fit the new
paradigm. This was the story of our journey. We started with courage and
curiosity. In the next few months we will push this forward to become
more than a simple prototype, to realize a complete suite of business
automation tools, ready for the cloud. The name of the initiative is
**Submarine, **from the famous Dijkstra quote. So, sit tight, and get
ready to dive in.

