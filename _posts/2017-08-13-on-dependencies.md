---
title: "On dependencies"
date: 2017-08-13
tags: core domain
---

Say, we develop an application. There are several hundreds of classes, couple of thousands of tests, few modules, tons of documentation... There are numerous things to focus on, but certainly few are more important, than others. What is the most important one?

As developers we have to manage a lot of code these days. Even tiny application consists of few thousands lines of code, which all comes from reusing a lot of code developed by a lot of other people outside our group. Let's look at an average web-service written in, say, java ecosystem. Most probably we would use few more restful services, store something in a relational database and use spring (or even spring boot) to assembly our code into working application to serve client's needs. Most probably it would look similar to the following. 

![Wrong dependencies]({{ site.url }}/assets/2017-08-13/wrong-dependencies.png) 
 
`Core` here is a module (or a set of modules) that client needs, which is something specific, which really makes client's business running, which represents the business. 
There are only two cases related to changes:
 1. At any time, code reflects business and when business requirements are changed, the code is changed along. _Changed_ code in this case reflects _changed_ business. Thus with respect to business, code is _invariant_. 
 2. Business might not require any changes, thus code won't be changed, because of it. In this case code is literally an _invariant_ with respect to business. 

The second case implies there are other reasons, besides business, which require code to change - different kinds of refactoring. That might sound suspicious as since there are no changes required by the business, then we don't need to change anything in the code. Actually we might use open source libraries and tools instead of proprietary or less number of faster nodes for database to decrease overall cost, or we might simply refactor our code to reuse it in another project. 

So, we have an invariant piece and variable one. I think it'd be proper to say that invariant is a center and variables are on the surface. And inevitably somewhere in between there should be some kind of border or a space, which splits one from the other. That's center only because that's heart of the business that code represents. Without this piece, whole application simply wouldn't exist, it wouldn't make any sense to write it, to use all the libraries and tools, and other applications. Thus, I'm saying that center is a `core` and everything else is just some details. 

What would those pieces consist of? Let's get back to our average web-service and start with the surface: 
- I would say that logging library is definitely a detail. We could use java util logging, log4j or slf4j api and any kind of implementation including binding to multiple logging libraries.
- transport: be it http, custom anything on top of plain tcp or udp. Large Hadron Collider collect so much data, so moving hard drives loaded into trucks provides higher throughput. Literally transport, huh? ;-)
- protocols: plain bytes, headers, buffers, compressed or encoded bytes, whole serialized objects or libraries like protocol buffers. All of that can be changed without touching anything business related.
- application artifact assembly like spring's or guice's DI capabilities.
- persisting: there are numerous of libraries nowadays like hibernate, mybatis, spring jdbc, chose whatever you want, change it for your convenience, just for fun or when appropriate!
- database... would that be an (un)important detail? Say, we might use relational database like Oracle (very powerful one) or whole bunch of clustered MySQLs instances. Or we might take NoSQL. Or keep everything in memory with any kind of Memcached. Of course changing paradigms is not simple - there are very different sets of features in different databases, but it's possible in principle, which makes it a detail with respect to the core of our web-service, to the business.
- jdk... that's certainly not a detail, right? Well, to some extent, but for most of it, it's just a library with all the primitives we need. The trick here is to differentiate primitives like Collection framework, Date utilities, whole set of IO decorators - essentially everything that we can write by ourselves - from, say, java.nio primitives like IO channels or concurrency primitives like Thread, Locks and others. The thing here is that we could use any library we want or even write some by ourselves to support DateTime utilities, concurrent utilities like ConcurrentHashMap and others. The reason that it's already there must not stop us from doing something better, when appropriate. For long time Joda-Time was much better, than default date-time.

What would constitute `the core`, then? If we filter out all the details, what's left? That, I think, would be a model of our business. Any model, actually, considering infinite solution space. And no single bit of any of those models relates to http, spring, hibernate, databases... Just a model per se. 

Someone might argue there are few edge-cases. Hibernate is someone's business, after all. Well, hibernate is not in the vacuum and it uses logging libraries, it probably uses DSL libraries, it supports different caching libraries - all of those are details with respect to hibernate's business. Changing these details won't change core functionality of it, the reason why it's been developed in the first place. 

What is that `core`, then anyway? That would be something that we write using java language. If business domain is chemistry, then in our `core` library we better have types like `Atom`, `Molecule`, `Descriptor`. If those `Descriptors` are calculated by some external library, then we might want to have interface `DescriptorCalculator`, whose sole purpose is to actually calculate `Descriptors` for given `Molecules`. Interface here is not to provide different implementations, which interface does by itself, but to name things properly. Such a `core` would describe all the details of our business without falling back to even standard jdk library. Instead of using `List<Atom>` we might define our own `Atoms` type (plural). Depending on context not every list of atoms would constitute a `Molecule`, but that's also possible. Don't get me wrong, I'm not saying we need to avoid using Collection framework or anything as such, I'm just saying it's possible, and sometimes it might be good to have such an option.

But we need at least some libraries to be available in the `core`! Sure, that's true. Say, to have dependency to slf4j is fine, as well as google's guava or apache commons. As a general rule, I'd say that it's fine to use set of static functions without (external) side effects. Any IO is an external side effect: it changes something outside of the `core` by sending it out there, stores it there, updates, deletes, etc. Libraries like http, hibernate, sql and others like that do something external, thus they're not recommended as they are details, which might be changed without real business need. As usual infinite decision space and lack of a precise theory what to do leaves us with subjective reasoning in each particular case.   

What would happen with our dependencies if we follow that? Dependencies are going to be reverted, like this ![Right dependencies]({{ site.url }}/assets/2017-08-13/right-dependencies.png)
Which is very good sign, because 
- our `core` doesn't depend on details anymore! Which in turn means that it won't be changed, because of some weird dependency.
- our `core` can be tested in complete isolation! We could provide 100% test coverage with only unit tests. Everyone knows that apache's http client works, we probably don't need to test it thoroughly (besides testing actual integration between our components), but the `core` is very different.
- depending on your style, business analysts or even managers would be able to at least partially understand what's in the code. The following is going to be clear to anyone, really: 
 {% highlight java %}     List<Molecule> desalted = molecularService.desalt(molecules);{% endhighlight %}
Properly define your own type system, put that into a `core` module and make your clients happy!