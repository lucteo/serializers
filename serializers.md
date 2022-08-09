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

TODO

Synopsis
========

Usage examples
==============

Design considerations
=====================

Specification
=============

---
