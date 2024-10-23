---
layout: article
description:  "Concurrency is the bread and butter of most modern applications, and it is important to consider the different models which can model and solve the problems that concurrency brings, apart from threads and locks."
title: "Actor Concurrency Model, Message Passing and its Guarantees in Erlang/Elixir"
date: 2022-07-16
categories:
 - programming
tags: 
    - elixir
    - erlang
    - actor-concurrency
image: images/Erlang_logo.jpeg
---

## The Concept

The actor concurrency model is a conceptual model where an actor represents the primitive unit of computation.
An Actor has 3 responsibilities: 

 - Communication
 - Storage
 - Processing

### Communication
A single actor on its own makes no sense. Actors come in systems and communicate with each other using mailboxes. Each actor is associated with an address and everything in the system would be modeled using actors.

### Storage

An actor can have a private internal state. Each actor is completely isolated from each other since no actor can access the state of another. They can only communicate using messages.

### Processing

An actor is allowed to do 3 things upon recieving a message

- Create more actors
- Send messages to other actors
- Setting the state for the next message

An example of an actor designating the state for the next message could be incrementing a counter on each message. Each message will be processed synchronously, and in the conceptual model, there is no guarantee on the ordering of messages.

The actor models falls nicely into a distributed system, as they are completely isolated and the messages can be passed across machines, via adresses. 

Every message will be delivered utmost only once. The delivery of the message itself can be considered as "best efforts". Everything else including how the the processes will communicate etc, is left out to the implementors. Let's take a look at Erlang, which is modeled on the concept.

## Erlang/Elixir

`
"Message passing starts with a Process Identifier. If it exists, 
the message is inserted into its signal queue. 
The messages are always copied."
`
[🔗](https://www.erlang.org/blog/message-passing/)

Erlang implements the actor concurrency model, and it's worth looking into the message passing guarantees that it gives.
 - Signals between two processes are guaranteed to arrive in the order they were sent.

This process is not the same as an operating system process. In Erlang, processes are lightweight and can be considered as actors
It is important to note that if more than one process sends signals to a common process they can arrive in any order.

Consider the following scenario: 

- Process A sends `$[1,2,3]$` to Common Process C.
- Process B sends `$[4,5,6]$` to Common Process C.

The messages may be recieved by C as `$[1,2,4,5,6,3]$`. The messages sent from A to C will arrive in order they were sent.

`
if an entity sends multiple signals to the same destination entity, the order is preserved; that is, if A sends a signal S1 to B, and later sends signal S2 to B, S1 is guaranteed not to arrive after S2. Note that S1 may, or may not have been lost.
` [🔗](https://www.erlang.org/doc/reference_manual/processes.html#signal-delivery)

The actual delivery of a message is not guaranteed, [only the order is](https://www.erlang.org/faq/academic.html#idm1231). 

## Implications

### Fault Tolerance

Processes are completely isolated. One process going down does not affect any other part of the system since there is no shared state amongst any of them. 
Instead of trying to program defensively and trying to handle every single fault that could happen, we let the process crash, and let a supervising process know what happened.

The supervisor is responsible for knowing what to do when the process crashes, and can restart the process with known state, or handle the failure with grace. 
A supervisor itself may be supervised and this can go up all the way.

### Distributed Systems

Since processes are completely isolated, it does not matter where the processes live, and could be distributed across networks. A process only needs to know the address it needs to send to, and everything else works the same.

### Deadlocks
The actor model can still cause deadlocks. Process A and B can end up both waiting for messages from each other, creating a deadlock. Even though this is rare, one should keep this in mind when designing systems.

- https://www.youtube.com/watch?v=7erJ1DV_Tlo 
- https://www.brianstorti.com/the-actor-model/ 
- Interesting Read: https://keunwoo.com/notes/rebooting/ 
