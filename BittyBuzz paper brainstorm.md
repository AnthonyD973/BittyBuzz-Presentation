# BittyBuzz Paper Brainstorm

This describes in a partly unorderly way things that could potentially have an
interest to journals. Most things would be just 1 or 2 sentences, or be
completely removed, depending on which journal we send to. This partly about
implementation specifics, and partly about swarm robotics / differences
between Buzz and BittyBuzz.

----------------------------------

- Cheap robot = resource constraints. Kilobots: 8MHz clock, 8-bit AVR
processor, 32 KB flash, 2 KB RAM, low bandwidth we measured around 400 B/s,
very high packet drop rates (in the order of 50%) that increases with infrared
communication saturation.
- BittyBuzz: implementation of Buzz for microcontrollers. Uses VM. Must be as
much like Buzz as possible while respecting the above constraints. Should be
easily portable to new robots. We have a platform-independent layer
containing the BittyBuzz core features, and a thin layer for
platform-dependant operations.

VM design
---------

- Same bytecode as Buzz, but we use a program to turn 32-bit arguments into
16-bit arguments to save space --warning about out-of-range values-- and
remove strings that compiled Buzz objects (`.bo`) contain. Fetching
bytecode is platform-specific.
- Stack-based VM. BittyBuzz's stack inspired from IA-32 (uses stackptr,
blockptr, we push old local symbols).
- On VM errors, the VM calls a platform-specific function that a user has the
choice to define.
- Closures like Buzz, but self table is pushed as implicit argument rather than
staying in activation record => closures that are inside a table but not
created in any context do not require an activation record, which accounts
for most of them, such as the closures of `swarm`, `neighbor` and
`stigmergy`.
- Unlike Buzz, strings (var names, string literals) are never actually used, but
string IDs are generated and used instead. A metaprogram takes the compiled Buzz
bytecode object (`.bo`), reads the strings used from the Buzz side and
generates a header containing string IDs. The header can then be used from the
user's C program (e.g., to get the string ID of `my_var`:
`BBZSTRING_ID(my_var)`).

Heap usage
----------

- Microcontroller => no OS, no `malloc`. Wrote our own heap to
allocate/deallocate symbols.
- Allocations always fixed-size. Two types of allocations: objects and
table segments. [Possibly explain implementation of both, as well as the
symbols' metadata]
- Garbage-collector for minimal heap usage. Problems when allocating objects
in interrupt routine, but we believe it would be possible to change object
allocation depending on whether or not we are in the garbage-collector.

Kilobot specifics
-----------------

- Use modified code from the kilolib, a wrapper library for kilobots. Allows
us to program large numbers of kilobots with the Overhead Controller from
KTeam, and use kilolib-like syntax for the C side of the user program.
- Buzz programs stored inside the Flash to have space for them.
- VM Error function stops motors and makes LED flash.
- Random transmission period to offsync transmissions and reduce
simultaneous communications.

Implementation of Buzz features
-------------------------------

- User has the possibility of disabling any of these features to reduce
RAM/Flash overhead.
- Virtual stigmergy:
  - Brief description.
  - Implementation identical to Buzz, but only one stigmergy permitted.
- Neighbors:
  - Brief description.
  - Two implementations: a flash-savvy, dynamic implementation that relies
  on heap tables, and memory-savvy implementation that uses a ringbuffer.
  - The goal for which the `neighbors` structure was created requires that
  the data about known neighbors is up to date as much as possible => 
  must somehow forget neighbors we do not receive messages from while
  using small RAM and performance overhead, and expecting high packet
  drops rates. Tradeoff between accuracy and overhead was implemented.
  Periodically, we enter a 'neighbor garbage-collection' phase where we
  we mark neighbors we received messages from. After a few timesteps, we
  remove all unmarked neighbors. Under the memory-savvy implementation
  which uses a fixed-length C array, we also bubble up neighbors we
  receive  messages from to make sure their data is not overridden.
  - Didn't implement the `kin` and `nonkin` closures, but will be
  implemented in the future (disabled for kilobots?).
- Swarm:
  - Brief description.
  - A robot's swarm membership (its "swarmlist") is a 1B value => up to 8
  swarms numbered from 0 to 7.
  - Buzz uses swarmlist transmission (`SWARM_JOIN`, `SWARM_LEAVE`,
  `SWARM_LIST` messages), however BittyBuzz does not require them because
  the `neighbors.kin` and `neighbors.nonkin` closures, which are the only
  ones that would require them, are not implemented.
- Messages:
  - Just as Buzz, the BittyBuzz VM handles platform-independent messages
  through an outgoing and an incoming buffer. The kilolib implements
  packets with 9B of payload, thus BittyBuzz messages were tailored to fit
  this data size. The buffers themselves are implemented as ring buffers
  so that data overriding is possible and affects the oldest pending
  messages only.

Behavioral tests
----------------

- Testing on around 40 laboratory kilobots.
- Showoff some Buzz scripts, and put in a cool picture of kilobots
executing some behaviors, especially the more complex ones (e.g. distance
gradient).

Misc
----

- Give an idea of how much Flash/RAM a user has left when all features
are enabled.

Future work
-----------

- More robot types (lab is curreently working on an implementation of zooids
with stm32f05 controller that has 64KB Flash and 8KB RAM).
