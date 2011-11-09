In order to build low-latency and high availability services over HTTP,
Mochi Media has migrated to an eventually consistent data model. Riak provides
the foundation, but does not provide any strategy to resolve conflicts with
concurrent updates other than "last write wins" or "build it yourself". Rather
than waiting for someone else to do it, we built our own abstraction that
simplifies the process of building it yourself.

statebox is the open source library that we've built for automatic conflict
resolution of several eventually consistent data structures such as sets,
dictionaries, and counters. It also provides facilities to develop your own,
given that certain constraints are satisfied. In this session you'll learn
about the implementation of statebox and the applications we have built with
it.
