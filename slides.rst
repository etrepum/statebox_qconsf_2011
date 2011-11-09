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
systems, so I'm going to gloss over it.

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

* read 

Questions?
==========

* Twitter: `@etrepum`_
* Mochi Media: `www.mochimedia.com`_
* Slides: `etrepum.github.com/statebox_qconsf_2011`_

.. _`www.mochimedia.com`: http://www.mochimedia.com/
.. _`etrepum.github.com/statebox_qconsf_2011`: http://etrepum.github.com/statebox_qconsf_2011

.. _`@etrepum`: http://twitter.com/etrepum
