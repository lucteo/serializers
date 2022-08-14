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
void trigger_save(game_state state) {
  auto do_save = [state] {
    // this is going to be serialized; at most one thread in this scope, at a given time
    game_save_process saver = ...;
    saver.save(state);
  };
  scheduler auto sch = ctx.scheduler();
  save_ser.spawn(on(sch, just() | then(do_save)));
}
```

## Limiting the maximum number of parallel accesses

Let's assume that we have an application that tries to download multiple files on an embedded device with limited bandwidth.
Although we can download multiple files in parallel, but because of the limited bandwidth, it may not be worth to download too many files at the same time.
In addition to that, let's assume that creating a connection is a complex process, and we want to limit the number of connections we created.

Thus, we have a problem in which we have a limited number of connections we can use.
In the classic concurrent model, one can use a counting semaphore to limit the number of connections that can be used at a given time.
With this paper, one can use an `n_serializer`:

```c++
using namespace std::execution;

constexpr int num_parallel_conns = 5;
vector<connection_t> connections = ...;           // assume we have 5 connections
using conn_handle_t = ...; // handle to a connection_t from the vector; dtor will mark the connection as free

n_serializer connections_ser{num_parallel_conns};
system_context ctx;

auto download(url_t url) {
  auto protected_work = [url] {
    // At most `num_parallel_conns` threads will enter this region

    // Get a free connection (guaranteed to have one)
    conn_handle_t conn = acquire_free_connection(connections);
    // Start downloading on that connection
    return just(conn)
         | let_value([url](conn_handle_t conn) {
            return conn.download(url);
         });
    // When the handle is destroyed, the connection will go back in the pool as a free connection
  };

  // Spawn the work and return a sender that is triggered when the download completes
  scheduler auto sch = ctx.scheduler();
  return connections_ser.spawn_future(on(sch, just() | then(protected_work)));
}

```

## Read/write access to a resource

Let's assume we have a game for which we load data dynamically as the player explore the game world.
Once the game data is loaded, it is immutable.
We might have multiple parts of the game reading the game data at the same time without any safety issue, but we cannot write the game data in parallel with any reads (or other writes).

In the classic concurrency model, this would be solved with a read-write mutex (shared mutex).
With this paper, we can use `rw_serializer` to apply the same idea to senders.

```c++
using namespace std::execution;

class game_data_t;
void load_data_for_area(area_id id, game_data_t& game_data);
void unprotected_render(const game_data_t& game_data);

game_data_t game_data;

rw_serializer game_data_ser;
system_context ctx;

void start_load_area(area_id id) {
  auto protected_work = [id] {
    // exclusive access to game_data while in this scope
    load_data_for_area(id, game_data);
  };

  scheduler auto sch = ctx.scheduler();
  game_data_ser.get_writer().spawn(on(sch, just() | then(protected_work)));
}

auto render() {
  auto protected_work = [] {
    // data doesn't change while rendering the scene
    unprotected_render(game_data);
  };
  scheduler auto sch = ctx.scheduler();
  return game_data_ser.get_reader().spawn_future(on(sch, just() | then(protected_work)));
}

auto spawn_enemies() {
  // similar to render()
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

## A new abstraction

*Note:* this section talks about the simple serializer, i.e., `serializer` class, but the argument extends to other serializer abstractions defined here.

After agreeing that serializers are useful, one question that arise is whether we need a separate abstraction (i.e., class) for a serializer, or we can just use existing facilities.

One idea was to use [@P2300R5] algorithms to simulate a serializer behaviour, maybe in conjunction with some helper class.
In particular, the idea of using the `on()` algorithm with a custom scheduler was considered.
However, there are two problems with this approach:

1. we can control only the starting point of a computation, and not necessarily the exit point
2. conceptually, schedulers work if we want the work to be executed within a given context; we might want to use with serializers computations that involve using multiple execution contexts

Let's assume that we want to pass through a (simple) serializer a complex computation that does work across multiple execution contexts.
The idea of the serializer is to ensure that no other computation is started in the serializer before the first computation is completed.
And, as this computation involves going through multiple execution contexts, we cannot simply detect the termination point.
Thus, point 1 above prevents use to use `on()` and schedulers to implement serializers.

Furthermore, looking from the perspective of the second point, if the computation is defined in terms of multiple schedulers, it is conceptually wrong to model the serializer as a scheduler.
One cannot have a scheduler to execute work that needs to be executed on a series of other schedulers.

Another idea that was considered is to use [@libunifex]'s `async_mutex` abstraction.
This can be used to model a simple serializer.
The problem with this approach is that it requires the user to manually *lock* and *unlock* the mutex.
This is considered unstructured and error-prone to be generally used.
However, it is worth noting that a simple serializer can be implemented in terms of `async_mutex`.

## Interface similar to `async_scope`

*Note:* this section talks about the simple serializer, i.e., `serializer` class, but the argument extends to other serializer abstractions defined here.

Designing the interface of a simple serializer can take several directions.
We chose to follow the interface of `async_scope` ([@D2519R0]) for two reasons:

* it provides the functionality that we need
* consistency

In terms of needed functionality, we need the following:

* a method of adding work to the serializer
	* this can be obtained with `spawn()`, `spawn_future()` and `nest()`
* a method of querying the serializer whether it has work: `on_empty()`
* allow cancellation

Once we have `spawn()` as a way to add work to the serializer, one might wonder why do we need `spawn_future()` and `nest()`.
We follow the same rationale as [@D2519R0] because the needs of enqueuing work are similar.

Similar to `async_scope` we need to be able to detect whenever the serializer doesn't have work enqueued anymore.
One reason for this is that we cannot destruct the serializer object while there is still work within it.

And, the same way that `asynsc_scope` requires cancellation, we require cancellation support for serializers, for the same reasons.

Please note that, one important point of considering `async_scope` as the starting point for the serializers work is the fact that they provide a structured way to work with dynamic computations.
See more discussion of about this aspect in [@D2519R0].

See also [Comparison with `async_scope`]


## Including `n_serializer` and `rw_serializer`

One might want to separate into separate papers the 3 abstractions proposed here: `serializer`, `n_serializer` and `rw_serializer`.
While there are reasons to do that (incremental progress works well with C\+\+ standard committee), there are also reasons to consider all 3 abstractions into the same proposal.

First, these abstractions correspond to the main ways of synchronization in the classical threading world; treating them together seems to make sense.
They provide tools to solve one type of problem.

While there are differences between them, the commonalities seem more important.

If we look at `serializer` and `n_serializer` we realise that they are extremely close.
Actually, a `serializer` object behaves the same as an `n_serializer` object constructed with a count of `1`.
Thus, `serializer` is a special case of `n_serializer`.

Furthermore, an `rw_serializer` can be thought of as a generalisation of a simple `serializer` (all computations given to it are *write* computations).
But, in this case, the interfaces of the two classes need to be different.
This is because `rw_serializer` can accept two types of computations.

Keeping the two abstractions in the same paper allows us to consider the enqueueing of dynamic computations with constraints in a more general form.

## Number of resources protected by an `n_serializer` is given to the constructor

## `rw_serializer` with two sub-objects

## Concepts vs. concrete classes

## `nest` as a basis for `spawn` and `spawn_future`

## Use of CPOs


Comparison with other models
============================

## Comparison with mutexes

## Comparison with P2300 model

## Comparison with `async_scope`

## Comparison with libunifex's `async_mutex`

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
  - id: libunifex
    citation-label: libunifex
    title: "libunifex: Unified Executors"
    author:
      - family: Facebook
    url: https://github.com/facebookexperimental/libunifex
---