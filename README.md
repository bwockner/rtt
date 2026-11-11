
The Orocos Real-Time Toolkit (fork)
===================================

About this fork
---------------

This repository is a fork of [orocos-toolchain/rtt](https://github.com/orocos-toolchain/rtt).
It has two purposes:

1. **Keep RTT 2.x usable** for existing systems on Linux (standard and
   PREEMPT_RT kernels, x86-64 and ARM), with low-risk improvements to
   determinism and throughput that need no change to component source code.
2. **Develop `rtt3`**, a successor with a new C++ API, built alongside the
   existing code in this repository.

The CORBA transport and the Python and Lua bindings are not a focus of
this fork.

### Current state

`master` contains upstream `master` with the `toolchain-2.9` and
`toolchain-2.10` branches merged in. The code is still RTT 2.x and builds
and installs as described under "Quick Installation" below. No `rtt3` code
exists yet; the work so far is planning.

### Direction for `rtt3`

The decisions taken so far, in brief:

- GCC 16 with C++26 reflection is required; there is no older language mode.
- A new API in namespace `rtt3`, installable next to RTT 2.x. The classic
  `TaskContext` API is not carried forward.
- Components are plain structs. Ports, properties and operations are
  described by their declarations, read through reflection, with no
  registration code, name strings or macros.
- Deployments are described in C++ and checked and wired at compile time,
  with per-machine variation (sensor and actuator stacks, kinematic
  variants, simulated hardware) selected from a configuration file at
  start-up.
- Algorithms are plain functions over plain data types that can be moved
  between a simulator and the real-time system unchanged.
- Real-time paths make no allocations, take no locks and make no system
  calls beyond the periodic sleep; errors are reported with
  `std::expected`.
- Linux only, on x86-64 and ARM.

These are proposals under discussion, and each plan lists the questions
that are still open.

### Plans

| Document | Content |
|---|---|
| [doc/rtt3/README.md](doc/rtt3/README.md) | Index of the `rtt3` plan: design by topic, stages, their dependencies and what can run in parallel |
| [doc/rtt3/logging.md](doc/rtt3/logging.md) | Lock-free logger for `rtt3` |
| [doc/rtt3/numerics.md](doc/rtt3/numerics.md) | SIMD, floating-point settings and agreement between simulator and target |
| [doc/optimisation-plan.md](doc/optimisation-plan.md) | Low-risk optimisations of the existing RTT 2.x code, with no API change |
| [doc/open-questions.md](doc/open-questions.md) | Undecided points, the licence position, and unverified assumptions |
| [AGENTS.md](AGENTS.md) | Layout, build and test instructions, and working rules for contributors and automated agents |

### Relationship to upstream

Upstream history, tags and branches are kept. The existing code remains
under its original licence, GPL version 2 with a linking exception (see
[COPYING.md](COPYING.md)); the licence for new `rtt3` code is not yet
decided. Changes made here to the RTT 2.x code are intended to remain
compatible with existing components at source level.


RTT 2.x
=======

Quick Installation
------------------

The Orocos RTT requires the [cmake](http://www.cmake.org) program to be present, version
2.8.3 or later is required.

Download the latest version from https://github.com/orocos-toolchain/rtt/releases

```
tar -xvjf <version>.tar.gz
cd rtt-<version>
mkdir build
cd build
cmake ..
make
make check
make install
```

The installation manual can be found in the doc dir.  You can also
consult it online on the http://www.orocos.org website.


Configuring Orocos RTT
----------------------

Instead of `cmake ..`, issue `ccmake ..` from your build
directory.  The keys to use are 'arrows'/'enter' to modify a setting,
'c' to run a configuration check (may be required multiple times), 'g'
to generate the makefiles. If an additional configuration check is
required, the 'g' key can not be used and you must press 'c' and
examine the output.

CORBA
-----

CORBA is not enabled by default. If you have [ACE/TAO](https://objectcomputing.com/products/tao) or [omniORB](http://omniorb.sourceforge.net/) installed, use

```
cmake .. -DENABLE_CORBA=ON -DCORBA_IMPLEMENTATION=TAO
```

or

```
cmake .. -DENABLE_CORBA=ON -DCORBA_IMPLEMENTATION=OMNIORB
```
