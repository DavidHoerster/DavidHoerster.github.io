+++
title = 'The Actor Model in a C# World'
date = 2023-12-07T17:46:43-05:00
draft = false
+++

*This post is the first in an occasional series on building a distributed system using Akka.NET*

The Actor Model is one of those approaches to system design that, at first, struck me as overly complex, but as I learned more about it, I discovered its simplicity and elegance, and its power. The [Actor Model](https://en.wikipedia.org/wiki/History_of_the_Actor_model) goes back to the 1970s with [Carl Hewitt](https://en.wikipedia.org/wiki/Carl_Hewitt) and has gained popularity over time for its ability to provide a simple approach to building highly scalable and concurrent systems.

(If you want an enjoyable video showing some greats having a discussion about the Actor Model, check out this [informal whiteboard chat](https://youtu.be/1zVdhDx7Tbs?si=bK2tx_0XIKC-10ak) with Erik Meijer, Carl Hewitt, and Clemens Szyperski.)

## What is the Actor Model
So briefly, what is the Actor Model and why should I use it? If you've had to design a system where you need to create a number of threads to handle some type of concurrent processing and ensure that you don't paint yourself into a deadlock corner, the Actor Model could be a tool to help solve that problem.

Actor Model systems are characterized by communication via immutable messages between objects called "actors". An actor is generally a small, lightweight object that should be specialized in its scope and can have its own state. An actor can communicate with other actors via messages, which are immutable POCOs. Actors are lightweight, cheap to spin up, and should be easy to tear down. Actor systems can see hundreds, thousands or more actors running at any given time.

Actors, in some actor model frameworks, can create actors of their own ("children") and can supervise those child actors. These supervisor actors can determine to tear down, restart, or ignore child actors that potentially get into a bad state.

## Hello Actor!

So let's see the codez, right!! Let's use Akka.NET 1.5 to create a simple Actor System:

``` csharp
var system = ActorSystem.Create("helloAkka");
```



## Taking Care of Your Children
