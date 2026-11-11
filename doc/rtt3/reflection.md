# rtt3: reflection on GCC 16

Part of the `rtt3` plan; see [README.md](README.md) for the index, the stages and how they depend on each other.

Stages: S1.

Reflection is the description mechanism of `rtt3`. There is no other.

**What GCC 16 provides.** Reflection is experimental and off by default; it
needs `-std=c++26 -freflection` and the `<meta>` header. Status below is
from the GCC 16 release notes and the cppreference support table; it has
not been compile-tested here, as this plan was written on a machine with
GCC 13 only. Confirming it on the real toolchain is the purpose of
stage S1, because the whole design now rests on it.

| Paper | Feature | Use here |
|---|---|---|
| P2996R13 | `^^` operator, splicers, `std::meta::info`, `<meta>` queries | Everything below |
| P3394R4 | Annotations, `[[=value]]` | Port, operation, property and activity metadata |
| P3096R12 | Function parameter reflection | Operation argument names and types |
| P3293R3 | Splicing a base class subobject | Descriptions of derived data types |
| P3491R3 | `define_static_string`, `define_static_object`, `define_static_array` | Placing generated tables in static storage |
| P3795R2 | Reflection clean-ups | - |
| P1306 | Expansion statements, `template for` (experimental) | Iterating members without recursive templates |

To confirm on a real GCC 16: P3560 (error handling in `std::meta`
functions) and P3068 (constexpr exceptions) were not confirmed by the
sources available.

**How it is used.**

- *Port discovery*: walk `nonstatic_data_members_of(^^T, ctx)` and keep
  members whose type is a specialisation of a port template.
- *Names*: `identifier_of(member)`, placed in read-only storage with
  `define_static_string`.
- *Annotations* carry everything the declaration cannot express:

  ```cpp
  struct Driver {
      [[=rtt3::doc("Joint command"), =rtt3::event]]
      rtt3::input_port<JointCommand> command_in;

      [[=rtt3::name("status")]]                 // override the identifier
      rtt3::output_port<Status> status_out;

      [[=rtt3::property, =rtt3::unit("rad/s")]]
      double max_velocity = 1.0;

      [[=rtt3::operation(rtt3::own_thread), =rtt3::doc("Clear a latched fault")]]
      std::expected<void, rtt3::error> reset(int axis) noexcept;

      [[=rtt3::hidden]]                         // not part of the interface
      rtt3::output_port<Debug> trace;

      void update() noexcept;
  };
  ```

  Operation argument names come from `parameters_of`.
- *Data types*: member names and types, bases (`bases_of`) and enum value
  names (`enumerators_of`) are read from the type alone, so plain structs
  and enums need no typekit and no annotation. A type from another library
  is reflected the same way; if its public members are not the right view
  of it, it is wrapped in a struct that is.
- *Deployments*: nodes and their activities, as in `deployment.md`.
- Only public members are reflected. Private ports and properties are not
  part of a component's interface.

**Containing the dependency.** Requiring an experimental feature with no
fallback is the largest technical risk in the plan. Three measures limit it:

- All uses of `<meta>` are confined to one internal module, `rtt3/meta/`,
  whose only output is a small set of ordinary constexpr tables
  (`component_info`, `type_info`, `deployment_info`). The rest of the
  library, including validation, scheduling and `rtt3-reflect`, consumes
  those tables and never touches a reflection value. A change in the
  feature is then a change in one module.
- A reflection conformance test set exercises exactly the queries that
  module relies on, independent of the rest of `rtt3`. It runs first in CI
  and is the gate for moving to a new GCC release.
- The tables have a documented plain-C++ form. This is not a supported
  user API, but it means that if reflection had to be abandoned, a
  hand-written or generated description could be reintroduced without
  redesigning the layers above.

`<meta>` is costly to include; keeping it in one module also keeps that
cost out of most translation units. Compile time is tracked from stage S0.

**Not planned.** `define_aggregate` (generating new class types) until
there is a concrete need. Contracts, also experimental in GCC 16, are a
separate topic.
