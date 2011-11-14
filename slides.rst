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
systems. Mostly JSON over HTTP at the
network boundary, nothing too clever!

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
* Schema evolution is hard
* Sharding not always possible, and rarely fun

Case Study: Friendwad
=====================

* A social graph aggregator
* MochiGames, Facebook, Twitter, Myspace
* Original implementation built on Mnesia
* Mnesia causes us pain

Friendwad Data Model
====================

* Twitter-like social digraph
* Each user has a unique id
* following: user ids that this user follows
* followers: user ids that follow this user

Adding a friend [1]
===================

TODO - Diagram

::

    id        | alice   | bob     |
    followers | []      | []      |
    following | []      | []      |

Adding a friend [2]
===================

TODO - Diagram

::

    id        | alice   | bob     |
    followers | []      | []      |
    following | []      | [alice] |

Adding a friend [3]
===================

TODO - Diagram

::

    id        | alice   | bob     |
    followers | [bob]   | []      |
    following | []      | [alice] |

Mnesia Implementation
=====================

* Table in mnesia for user records (id, following, followers)
* Multi-row transaction for each graph change
* At least two rows in each transaction, possibly more (third-party
  import)

Why not Mnesia?
===============

* Mnesia issues beyond the scope of this talk :)
* Anyway, we decided to migrate to Riak

Riak
====

* Great solution for many of our data problems (thanks Basho!)
* Distributed eventually consistent key-value store
* But not a complete solution

Riak Migration
==============

* The simplest thing that could possibly work (incorrectly)
* … appears correct with serialized glasses
* Riak not transactional even for changes to a single row

Riak Migration Continued
========================

TODO - Facepalm

Eventual Inconsistency
======================

* Popular user claimed they were missing entries in "followers"
* Verified that they were missing by looking at our analytics built from
  our transaction logs
* Especially non-transactional with ``allow_mult=false``!
* My face probably still has a palm-shaped dent

Fix all of the things
=====================

* Turn on ``allow_mult=true``
* Easy fix if friends could only be added (set union)
* But you can remove friends :(
* Implemented statebox in anger to solve this problem

Statebox Design Philosophy
==========================

* Adding code to Riak should be avoided (maintenance)
* The only option is to resolve conflicts on read
* Growth should be bounded and configurable
* Doesn't need to be language agnostic
* Minimize magic

What's Statebox?
================

* Opaque container
* Serializes current state
* With recent operations
* Provides merge operation
* Monad-like (not important)

Statebox Terminology
====================

* ``op()`` :: N-ary function reference plus N-1 arguments
* ``event()`` :: ``{timestamp(), op()}``

.. class:: erlang

::

    {fun ordsets:add_element/2, [kitten]}

Statebox Internals
==================

* Designed to be used with Erlang's external term format (``term_to_binary``)
* Serializes function references, so is bound to exported code
* Prototyped in friendwad, but immediately extracted it because we had
  other projects that would need it
* Open sourced because I couldn't find anything else like it

Statebox Theory
===============

* Statebox algorithm can be used as-is with any eventually consistent
  KV store
* Similar to paper on `CRDT`_ (Convergent / Commutative Replicated
  Data Types)
* Stores current value plus a (configurably) bounded event queue
* Event queue is bound by length and can expire events by age

.. _`CRDT`: http://hal.archives-ouvertes.fr/inria-00555588/

Statebox Example [1]
====================

TODO - Diagram

::

   A     :: [kitten]
   [{1, Union([kitten])}]

Statebox Example [2]
====================

TODO - Diagram

::

   A     :: [kitten]
   [{1, Union([kitten])}]

   B     :: [puppy]
   [{2, Union([puppy])}]

Statebox Example [3]
====================

TODO - Diagram

::

   A     :: [kitten]
   [{1, Union([kitten])}]

   B     :: [puppy]
   [{2, Union([puppy])}]

   [A,B] :: [kitten, puppy]
   [{1, Union([kitten])},
    {2, Union([puppy])}]

Statebox Merge
==============

* B is newer, so use its value as the basis
* Merge sort event queues
* Apply ops in order from the beginning

Statebox Merge [1]
==================

TODO - Diagram

::

  Use B's value (arbitrarily newest)

              [puppy]

  Value = [puppy]

Statebox Merge [2]
==================

TODO - Diagram

::

  Apply ops oldest to newest (T=1)

        union([puppy], [kitten])

  Value = [kitten, puppy]

Statebox Merge [3]
==================

TODO - Diagram

::

  Apply ops oldest to newest (T=2)

  union(union([puppy], [kitten]), [puppy])

  Value = [kitten, puppy]


Interleaving for Fail
=====================

* To simulate failure we need multiple concurrent operations
* ``alice → bob`` and ``bob → carol`` will be interleaved

``alice → bob`` dissected
=========================

* read ``alice``
* write modified ``alice`` (following union [bob])
* read ``bob``
* write modified ``bob`` (followers union [alice])

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
