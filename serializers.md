---
title: "serializers -- Adding constraints between dynamic computations"
subtitle: "Draft Proposal"
document: D????R0
date: today
audience:
  - "SG1 Parallelism and Concurrency"
  - "LEWG Library Evolution"
author:
  - name: Lucian Radu Teodorescu
    email: <lucteo@lucteo.ro>
toc: true
---

Changes
=======

## R0

- first revision

Introduction
============

[@P2300R5] is a major step towards structured concurrency.
It defines building blocks that can be used to structure concurrent programs hierarchically, ensuring safety without any performance penalty. Senders are the right abstractions to deal with concurrency.

Most of the algorithms defined in [@P2300R5] are focused on building chains of work.
That is, to build *static* graphs of concurrent work; the actual work that needs to be performed needs to be known upfront.

In practice, though, there is also "dynamic" work.
This is work that we don't fully know upfront.
One example of "dynamic" work is the processing of *N* chunks of data, possible with different concurrency constraints, where *N* is a variable.

[@P2300R5] provides `start_detached` and `ensure_started` to handle with work that doesn't necessarily fall into "static" work category.
But these primitives are not following structured concurrency primitives, and can be unsafe.
[@D2519R0] fixes these problems with the introduction of `async_scope`.
With this, we can dynamically launch concurrent work safely.

However, these abstractions don't allow us to establish constraints between different work chunks.
For example, if multiple parts of an application (i.e., dynamic work) want to write data to a file, we can't simply add a constraint so that maximum one computation can write to the file at any given type.
As another example, we might have a limited number of resources, let's say *N*, so maximum *N* threads can actively use these resources; again, there is no easy way to express this constraint.

This paper aims at providing some abstractions that allow the users to start dynamic work with constraints associated to them.
It provides 3 abstractions: `serializer`, `n_serializer` and `rw_serializer`.
These abstractions provide similar constraints to those found in mutexes, semaphores and read-write mutexes, but with applicability to senders.


Motivation
==========

## Adding constraints to dynamic work.

Virtually all threading libraries and frameworks support adding constraints between (dynamic) work that can be executed on multiple threads.

Currently, both [@P2300R5] and [@D2519R0] lack abstractions that allow adding constraints to dynamic work.
One can use `start_detached()` (with [@P2300R5]) and `async_scope::spawn()` (with [@D2519R0]), to start dynamic work, but there is no way to add constraints between the dynamic computations that one wants to start.

This paper adds abstractions that allow dynamically starting computations with simple constraints attached to them.
Even though the constraints are simple, practice shows us that these are the most useful constraints for most of the programs.


## Moving away from the world of blocking primitives

With the new concurrency model introduced by [@P2300R5], people would want to move from their legacy code-bases (with threads and locks) to this new model.
To make this transition, among others, people would have to replace synchronization primitives with something equivalent.

This paper provides abstractions that integrate well with senders / receivers framework that are equivalent to the following classic synchronization primitives:

* mutexes
* semaphores
* read/write mutexes

Using the provided primitives, the transition from threads & synchronization primitives to senders is easier.


## Mutual exclusion is an important concept

It is said that concurrency in software has its origins in the 1965 Dijkstra's paper *Solution of a problem in concurrent programming control* [@Dijkstra65].
The paper provided the first solution to a concurrent problem, and the solution was what we now call a *mutex*.
A way to ensure that multiple processes (machines in Dijkstra's paper) have access to a resource (*critical section*, as Dijkstra names it) in such a way that the resource is accessed at most by one process.
Mutexes are the first solution to concurrency issues, appearing at the same time with concurrency.

But, despite being the oldest solution to concurrency problems, other solutions did not supersede it.
Other synchronization primitives exist, but one can safely say that mutex is still the most frequently used.
We can find mutexes in low-level threading frameworks, programming language constructs, threading libraries, etc. On the other hand, the use of semaphores (for example) is not as widespread.

Programs also tend to use mutexes a lot.
A search on [@codesearch] yields that almost one in 200 files contain `std::mutex`.
As a comparison, the ratio for `std::thread` usage is 1:593, the ratio for `std::atomic` is 1:445, while the ratio for `std:condition_variable` is 1:2700.
The occurrence frequency of `std::mutex` is even greater than the occurrence frequency of `std::unordered_map` (which is about 1:212).

Moreover, in our approaches of teaching concurrency, mutexes appear as one of the important synchronization primitives.
Typically, the teaching materials (lectures/books/articles) describe threads, and intermediately after the problems caused by the lack of synchronization, and then it lists mutexes as the most common solution to this problem.

For better or worse, the idea of a mutually exclusive critical section is deeply rooted in the minds of concurrent programmers.
To have the senders/receivers model be a success, we should to provide an abstraction that is familiar to the existing programmers.


Usage examples
==============

## Mutual exclusion while accessing a resource

Let's assume that we are building a game.
In this game, we need to save the state of the game at certain points, based on multiple triggers.
The triggers are independent of each-other, and we can imagine multiple triggers will activate at the same time.

The functionality that saves the state of the game doesn't make sense to be called in parallel by multiple threads.
Thus, after one trigger starts the saving of the state, other triggers cannot start another save process.
The process for saving the state corresponding to the second trigger must start only after the first saving of the state process completes.

In the classical concurrency model with threads and locks, we would add a mutex around the process of saving the game state.
This is the classic example of using a mutex.

With this paper, one can use senders and achieve mutual exclusion:

```c++
using namespace std::execution;

serializer save_ser;
system_context ctx;

// can be called in parallel from multiple threads
// exits immediately
void trigger_save(const game_state& state) {
  auto do_save = [] {
    // this is going to be serialized; at most one thread in this scope, at a given time
    game_save_process saver = ...;
    saver.save(state);
  };
  scheduler auto sch = ctx.scheduler();
  save_ser.spawn(on(sch, just() | then(do_save)));
}
```




Synopsis
========

Depending on the outcome of [@D2519R0], the 3 proposed abstractions might look like:

```c++
// Similar to async_scope, but with the following constraint:
//      maximum one given sender can be executed in parallel
// Same idea as with mutexes, but using senders.
struct serializer {
    serializer();
    ~serializer();
    serializer(const serializer&) = delete;
    serializer(serializer&&) = delete;
    serializer& operator=(const serializer&) = delete;
    serializer& operator=(serializer&&) = delete;

    template <sender S>
    void spawn(S&& snd);
    template <sender S>
    @_spawn-future-sender_@<S> spawn_future(S&& snd);

    template <sender S>
    @_nest-sender_@<S> nest(S&& snd);

    [[nodiscard]]
    @_on-empty-sender_@ on_empty() const noexcept;
    
    in_place_stop_source& get_stop_source() noexcept;
    in_place_stop_token get_stop_token() const noexcept;
    void request_stop() noexcept;
};

// Similar to async_scope/serializer, but with the following constraint:
//      maximum N senders can be executed in parallel, where N is given to ctor
// Same idea as with semaphores, but using senders.
struct n_serializer {
    explicit n_serializer(int max_parallel);
    ~n_serializer();
    n_serializer(const n_serializer&) = delete;
    n_serializer(n_serializer&&) = delete;
    n_serializer& operator=(const n_serializer&) = delete;
    n_serializer& operator=(n_serializer&&) = delete;

    template <sender S>
    void spawn(S&& snd);
    template <sender S>
    @_spawn-future-sender_@<S> spawn_future(S&& snd);

    template <sender S>
    @_nest-sender_@<S> nest(S&& snd);

    [[nodiscard]]
    @_on-empty-sender_@ on_empty() const noexcept;
    
    in_place_stop_source& get_stop_source() noexcept;
    in_place_stop_token get_stop_token() const noexcept;
    void request_stop() noexcept;
};

// Similar to async_scope/serializer, but with the following constraints:
//      - there are two types of senders: READ and WRITE
//      - a WRITE sender cannot be executed in parallel with other WRITE or READ senders
//      - multiple READ senders can be executed in parallel.
// Same idea as with read-write mutexes, but using senders.
struct rw_serializer {
    rw_serializer();
    ~rw_serializer();
    rw_serializer(const rw_serializer&) = delete;
    rw_serializer(rw_serializer&&) = delete;
    rw_serializer& operator=(const rw_serializer&) = delete;
    rw_serializer& operator=(rw_serializer&&) = delete;

    struct reader {
        ~reader();
        reader(const reader&) = delete;
        reader(reader&&) = delete;
        reader& operator=(const reader&) = delete;
        reader& operator=(reader&&) = delete;

        template <sender S>
        void spawn(S&& snd);
        template <sender S>
        @_spawn-future-sender_@<S> spawn_future(S&& snd);

        template <sender S>
        @_nest-sender_@<S> nest(S&& snd);

    private:
        reader(rw_serializer* parent);
    };

    struct writer {
        ~writer();
        writer(const writer&) = delete;
        writer(writer&&) = delete;
        writer& operator=(const writer&) = delete;
        writer& operator=(writer&&) = delete;

        template <sender S>
        void spawn(S&& snd);
        template <sender S>
        @_spawn-future-sender_@<S> spawn_future(S&& snd);

        template <sender S>
        @_nest-sender_@<S> nest(S&& snd);

    private:
        writer(rw_serializer* parent);
    };

    reader& get_reader();
    writer& get_writer();

    [[nodiscard]]
    @_on-empty-sender_@ on_empty() const noexcept;
    
    in_place_stop_source& get_stop_source() noexcept;
    in_place_stop_token get_stop_token() const noexcept;
    void request_stop() noexcept;
};

```


Design considerations
=====================

Specification
=============

---
references:
  - id: Dijkstra65
    citation-label: Dijkstra65
    type: book
    title: "Solution of a problem in concurrent programming control"
    author:
      - family: Dijkstra
        given: E. W.
    publisher: Communications of the ACM, September 1965
    issued:
      year: 1965
      month: September
  - id: codesearch
    citation-label: codesearch
    title: "Code search engine website"
    author:
      - family: Tomazos
        given: Andrew
    url: https://codesearch.isocpp.org
---