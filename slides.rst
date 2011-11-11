.. include:: <s5defs.txt>
.. include:: ui/beamerdefs.rst

.. raw:: html
    :file: includes/logo.html

=================================================
Eventually Consistent HTTP with Statebox and Riak
=================================================

:Author:
    Bob Ippolito (`@etrepum`_)
:Date:
    November 2011
:Venue:
    QCon San Francisco 2011

Introduction
============

This talk isn't really about web. It's about how we
model data for the web.

HTTP itself is not the interesting part of our
systems.

Mochi's Business
================

* We provide platforms for Flash game developers
* Ads, analytics, virtual currency, social, scores, etc.
* Terabytes of data to report on

Just a few years ago...
=======================

* Millions of tuples was big
* Scale up vertically
* Single master SQL databases

Why was this easy?
==================

* ACID is cheap on a single node
* Efficient to establish a total ordering for events
* Single node systems do not have network partitions!
* Most businesses can probably still get away with this

When does this break down?
==========================

* Availability is important (we're global!)
* Too expensive to scale vertically
* If you think that sharding sucks

Riak
====

* Great solution for many of our data problems (thanks Basho!)
* But not a complete solution

Example: Denormalized Social Graph
==================================

* Keys are user ids
* Initial values::

    [{following, []},
     {followers, []}]

Following another user (idealized)
==================================

``alice → bob`` (just the desired result)

``alice``::

  [{following, [bob]},
   {followers, []}]

``bob``::

  [{following, []},
   {followers, [alice]}]

Following another user (naive)
==============================

* read ``alice``
* write modified ``alice``
* read ``bob``
* write modified ``bob``

Concurrency ruins everything
============================

``alice → bob → carol``

* (``a→b``) read ``bob``
* (``b→c``) read ``bob``
* (``b→c``) write modified ``bob``
* (``a→b``) write modified ``bob``
* Last write wins, ``bob`` is not friends with ``carol``

Sibling Rivalry
===============

* If ``allow_mult`` is on, the next read of ``bob`` will
  have two siblings (``[a→b, b→c]``) because they descend
  from the same vector clock.
* Default strategy is "last write wins". This strategy often
  leads to pain.

How to fix?!
============

* Riak (or equivalent) + Statebox (or equivalent)!

What's Statebox?
================

* Opaque container
* Serializes current state
* With recent operations
* Provides merge operation
* Monad-like (but that's not important)

Statebox Theory
===============

* Statebox algorithm can be used as-is with any eventually consistent
  KV store
* Similar to paper on `CRDT`_ (Convergent / Commutative Replicated
  Data Types)

.. _`CRDT`: http://hal.archives-ouvertes.fr/inria-00555588/

Declarative (ordsets)
=====================

.. class:: erlang

::

    Add = fun ordsets:add_element/2,
    Empty = statebox:new(fun () -> [] end),
    A = statebox:modify({Add, [a]}, Empty),
    B = statebox:modify({Add, [b]}, Empty),
    Resolved = statebox:merge([A, B]),
    statebox:value(Resolved) =:= [a, b].

Composable (orddict of ordsets)
===============================

.. class:: erlang

::

    Empty = statebox_orddict:from_values([]),
    Union = fun statebox_orddict:f_union/2,
    A = statebox:modify([Union(following, [b]),
                         Union(followers, [c])],
                        Empty),
    B = statebox:modify([Union(following, [b]),
                         Union(followers, [d])],
                        Empty),
    Resolved = statebox:merge([A, B]),
    statebox:value(Resolved) =:= [{followers, [c, d]},
                                  {following, [b]}].

statebox_riak wrapper
=====================

.. class:: erlang

::

    %% bob → alice, bob → carol
    S = statebox_riak:new([{riakc_pb_socket, Pid},
                           {expire_ms, 5000},
                           {max_queue, 50}]),
    Union = fun statebox_orddict:f_union/2,
    statebox_riak:apply_bucket_ops(
        <<"users">>,
        [{[<<"alice">>, <<"carol">>],
          Union(followers, [bob])},
         {[<<"bob">>],
          Union(following, [alice, carol])}],
        S).

Restrictions
============

* Operations must be repeatable (idempotent unary operation)
* Repeatable if and only if ``F(V) = F(F(V))``
* Old operations in the queue are replayed in-order on merge,
  but seeded with newer data

Repeatable Operations
=====================

* Most set operations
* Most dictionary operations
* NOT most list operations (ordered lists may be ok!)
* NOT most integer operations

What about non-repeatable operations?
=====================================

* Many can be built on top of repeatable operations
* ``statebox_counter`` is one example

statebox_counter
================

* Represent a counter as an ordered list of events
* ``[{{Timestamp, Ident}, Delta}]``
* ``Ident`` is just a unique-ish identifier (node counter, random
  number, etc.)

statebox_counter optimization
=============================

* Prevent unbounded growth by coalescing old events into a single
  event with a fixed ``Ident``
* Events older than this are ignored

Better than Statebox
====================

* We'd all be better off if this kind of data structure was built-in
  to the database
* Redis-like features, but concurrent and multi-node
* Higher level APIs

Why could Riak do it better
===========================

* The database could serialize access to a given key per node
* The database could automatically prune/expire because it knows
  the state of the cluster
* Can leverage information already present in vector clocks

Questions?
==========

* Twitter: `@etrepum`_
* Mochi Media: `www.mochimedia.com`_
* Slides: `etrepum.github.com/statebox_qconsf_2011`_
* http://git.io/statebox
* http://git.io/statebox_riak

.. _`www.mochimedia.com`: http://www.mochimedia.com/
.. _`etrepum.github.com/statebox_qconsf_2011`: http://etrepum.github.com/statebox_qconsf_2011
.. _`@etrepum`: http://twitter.com/etrepum
