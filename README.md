# ago - [![Build Status](https://travis-ci.org/steveyen/ago.png?branch=master)](https://travis-ci.org/steveyen/ago)

A time travel library for clojurescript core.async

The "ago" library provides a limited form of time travel (snapshots
and restores) on top of clojurescript core.async.

## Rationale

The ago library came about because I was using core.async to handle
concurrent tasks.  In particular, I was trying to simulate some
concurrent clients, servers and network protocols of a "fake"
distributed system, and core.async was a good building block to model
all the concurrent activity.  (I used go routines for the client &
server "processes" and used channels to hook them up together.)

By using clojurescript core.async, too, I could run my simulated
"distrbuted system" as a 100% client-side, single page application and
get a quick web browser GUI and visualization of everything happening
in the simulation.

But, it wasn't clear how to "rewind the world" back to a previous
simulation state so that I could play "what-if" games with the
simulated world.

That is, I wanted to...
* snapshot all the inflight go routines, channels, messages,
  timeouts and all their inherent state;
* take multiple snapshots over time;
* and finally, later restore the world to some previous snapshot
  and restart the world from exactly that point in time.

The ago library is meant to provide that snapshot and restore ability,
so that one can have "TiVo for clojurescript core.async".

Or, if you like synonyms...

    ago lets you go backwards in your go routines to sometime ago

## How To Use ago

### require

First, you'll need to require the relevant clojurescript macros and
functions, like...

    (ns my-application
      (:require-macros [ago.macros :refer [ago]])
      (:require [cljs.core.async :refer [close! <! >! alts! put! take!]]
                [ago.core :refer [make-ago-world ago-chan ago-timeout
                                  ago-snapshot ago-restore]]))

### API: make-ago-world

The make-ago-world API function creates a world-handle, which is an
atom where the ago library tracks everything about a
snapshot'able world.

    (make-ago-world app-data) ; returns a world-handle.

You must also supply some opaque app-data (use that app-data
for whatever you want) that will be associated with the
world-handle. This can also be nil...

    (make-ago-world nil)

### API: ago

The ago library provides an alternative or twin set of API's which
wrap around the main creation API's of core.async.  These mirrored
API's usually have an additional first parameter of a world-handle.

For example, instead of...

    (go ...)

...use...

    (ago world-handle ...) ; returns a chan

### API: ago-chan

Instead of (chan), use...

    (ago-chan world-handle) ; returns a chan

### Example

So, you would create "ago channels" and "ago routines", passing in a
world-handle.  For example, instead of writing...

    (let [ch1 (chan)]
      (go (>! ch1 "hello world"))
      (go ["I received" (<! ch1)]))

...you would instead use the ago equivalent API, like...

    (let [ch1 (ago-chan world-handle)]
      (ago world-handle (>! ch1 "hello world"))
      (ago world-handle ["I received" (<! ch1)]))

### API: ago-snapshot

To snapshot a world, use...

    (ago-snapshot world-handle) ; returns a snapshot-handle

For example...

    (let [snapshot-handle (ago-snapshot world-handle)]
      ...
      )

### API: ago-restore

To restore a world-handle to a previous snapshot, use...

    (ago-restore world-handle snapshot-handle)

### API: ago-timeout

Instead of (timeout delay), use...

    (ago-timeout world-handle delay) ; returns a chan

If you use the ago-timeout feature, you may want to slow down
or speed up simulation time (or "logical time").

That is, there's a distinction between logical time and physical clock
time, where logical time can proceed at a different pace than physical
clock time.  Just set the :logical-speed value in a world-handle to
not 1.0.  That is, you could replay your movie in super slow-mo
so that you can actually watch normally fast timeouts happening.

Of note, logical time can rollback to lower values when you restore a
previous snapshot, which can have interesting, unintended rendering
effects if you're just doing simple delta "functional reactive" style
visualizations.

Logical time starts at 0 when you invoke (make-ago-world ...).

### Limitations

The snapshotting and rewinding in ago works only if you use
immutable/persistent data structures throughout your ago routines.

Because the ago routines and ago channels have additional overhead,
you should use regular clojurescript core.async API functions (go,
chan, timeout) for areas that you don't want to snapshot (i.e., not
part of your simulation/model), such as GUI-related go routines that
are handling button clicks or rendering output.

## LICENSE

Eclipse Public License

## Building

    lein cljsbuild once

This was inobvious to me until Aaron Miller pointed it out to me,
to help during development...

    lein cljsbuild auto

### Tests

    lein cljsbuild test

## Underneath The Hood

This section might be interesting only to those folks who get into how
core.async works or who want to understand more of ago's limitations.

ClojureScript provides hooks in its core.async macros which transform
go blocks to SSA form, and the ago library utilizes those hooks to
interpose its own take/put/alts callbacks so that ago has access to
the state machine arrays of each "go routine".  With those hooks the
ago library can then register those state machine arrays into its
world-handle.

A world-handle is just an atom to an immutable/persistent
associative hash-map that holds onto those state machine arrays
and other stuff.

Also, instead of using clojurescript core.async's default buffer
implementation (a mutable RingBuffer), the ago library instead
requires that you use its immutable/persistent buffer implementation
(fifo-queue).  These buffers are also all registered into the
world-handle.

Because the world-handle holds onto all relevant core.async data, a
snapshot is then implemented by copying a world-handle and also
cloning any contents of registered state-machine-arrays.  The buffer
snapshotting comes "for free" due to ago's use of immutable/persistent
buffer implemenations.

A restore is then copying a previous world-handle back into place,
and copying back any previous contents of the state-machine-arrays.

In short, this snapshotting and rewinding all works only if you use
immutable/persistent data structures throughout your model.

Each world-handle also tracks a branching version vector, so that any
inflight go routines can detect that "hey, I'm actually a go routine
apparently spawned on a branch that's no longer the main timeline, so
I should exit (and get GC'ed)".

The ago library also has its own persistent/immutable
re-implementation of the timeout queue (instead of core.async's
mutable skip-list implementation), again for easy snapshot'ability.

Although ago was written to not have any changes to core.async, one
issue with ago's current approach is that it may be brittle, where
changes to clojurescript core.async's SSA implementation or
Channel/Buffer/Handler protocols can easily break ago.

## TODO

* Need to learn how to publish libraries in clojure (clojars.org?).
* Need auto-generated docs.
* Need examples.
* Need to learn cljx.
* Run this in nodejs/v8 on the server-side?
* Figure out how to use this in clojure/JVM.
* Figure out how to serialize/deserialize a snapshot.
  (e.g, save a snapshot to a file.)
  This may be challenging if there's lots of state stuck in closures.
  Probably have to define some more app coding limitations to allow for
  serialization/deserialization.
