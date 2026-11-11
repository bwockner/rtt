# rtt3: error handling and macro policy

Part of the `rtt3` plan; see [README.md](README.md) for the index, the stages and how they depend on each other.

These conventions apply to every stage.

## Error handling

Rules:

- Every fallible `rtt3` function returns `std::expected<T, rtt3::error>`
  and is `noexcept` and `[[nodiscard]]`. `rtt3::error` is a small trivially
  copyable value (an error code plus a pointer to static context), so
  returning it never allocates.
- Outcomes that are part of normal operation are not errors. Reading a port
  returns a flow status (`no_data`, `old_data`, `new_data`); a full buffer
  on write returns a status according to the connection policy.
- Constructors do not fail. Components and library objects are cheap to
  construct; anything that can fail happens in `configure()` or in factory
  functions that return `expected`.
- No `rtt3` function throws, at configuration time or on the update path.

**Why not allow exceptions at configuration time as well?** With
`std::expected` available there is little left in their favour:

| Argument for exceptions | Assessment |
|---|---|
| Constructors cannot return a value | Solved by the configure step RTT already has, and by factory functions. |
| Less boilerplate when propagating errors through several layers | True, C++ has no propagation operator. The monadic operations (`and_then`, `transform`, `or_else`) cover most cases, and configuration call chains here are shallow. |
| Errors cannot be silently ignored | `[[nodiscard]]` with `-Werror=unused-result` gives the same guarantee at compile time. |
| The standard library throws anyway | True, and unavoidable; see below. It is a reason to tolerate exceptions, not to add more. |
| Familiar to users | Weak; a mixed model (throwing at configuration, status on the update path) is harder to teach than one rule. |

Against them: a throw has unbounded and allocation-dependent cost, so it
must never happen on a real-time path anyway; a second error channel makes
every function's contract two-part; and `expected` puts the failure modes
in the signature where both the reader and the compiler can see them.

**Where exceptions are still tolerated.** The libraries are compiled with
exceptions enabled, because two sources remain that `rtt3` does not control:

- The standard library (`std::bad_alloc`, `std::format` errors, container
  operations) during configuration. `rtt3` code is written so that these
  cannot occur for reasons other than memory exhaustion, which is treated
  as fatal.
- User code inside component hooks. By default the `rtt3::component`
  concept requires hooks to be `noexcept`, which makes the contract
  checkable at compile time; a throw then terminates the process
  deterministically. A deployment can instead opt a node into
  `rtt3::catch_exceptions`, which wraps its hooks in one `try`/`catch` and
  moves the component to its fault state, as RTT 2.x does today. This is
  for components that wrap third-party code that throws.

Reflection queries may signal errors with exceptions
during constant evaluation; that happens in the compiler and has no
run-time effect.

## Macro policy

A macro is allowed only where the language has no other mechanism, and
never in the API a component author writes against.

| RTT 2.x | rtt3 |
|---|---|
| `ORO_CREATE_COMPONENT`, `ORO_LIST_COMPONENT_TYPE`, `ORO_CREATE_COMPONENT_LIBRARY` | `rtt3::plugin::registrar<T>` (below) |
| `ORO_SERVICE_PLUGIN` family | The same registrar |
| `ORO_main` | Plain `main` calling `rtt3::run<App>()` |
| `OROBLD_*`, `OROPKG_*`, `OROCFG_*` configuration macros | Generated `rtt3/config.hpp` with `inline constexpr` values, used with `if constexpr` |
| `OROCOS_TARGET`, `fosi` function macros, `oro_atomic_*` | Deleted; `rtt3::os` and `std::atomic` |
| Token pasting and stringising helpers | Deleted with their users |
| Preprocessor repetition for signatures up to N arguments | Variadic templates |
| Deprecation and unused-argument macros | Standard attributes |
| Logger file and line macros | `std::source_location` default arguments |

**Registration without macros.** Needed only for runtime loading. A
component library names its exported types with one line per type at
namespace scope:

```cpp
inline const rtt3::plugin::registrar<Sensor> sensor_registration;
```

The registrar adds a typed factory, named from the reflected type, to a
registry owned by `librtt3-reflect`; the loader opens the library and reads
what was added. There are no `extern "C"` entry points. The exact spelling
is settled with a test that registration survives static archives and
`--gc-sections`.

**Macros that remain**, all internal to library headers: include guards or
`#pragma once`; one symbol-visibility macro from a generated export header;
the test framework's assertion macros in tests. CI fails on any new
`#define` outside an allow-list.
