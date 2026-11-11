# rtt3: toolchain and build

Part of the `rtt3` plan; see [README.md](README.md) for the index, the stages and how they depend on each other.

Stages: S0.

- CMake 3.28 or later, target-based, exported targets, `CMakePresets.json`
  including an `aarch64-linux-gnu` cross preset.
- Compiler: GCC 16 with libstdc++ 16, on both x86-64 and AArch64. No older
  GCC is supported. GCC 16 defaults to `-std=gnu++20`, so the build always
  sets the standard explicitly.
- One build mode: `-std=c++26 -freflection`, extensions off. The flags are
  exported as usage requirements of the `rtt3::core` CMake target, so
  component projects get them by linking it.
- The C++23 library and the C++26 additions in libstdc++ 16 can be used
  throughout, including in the libraries themselves: `std::expected`,
  `std::move_only_function`, `std::print`, `std::flat_map`, deducing
  `this`, pack indexing, expansion statements, and (once confirmed on the
  real toolchain) `std::inplace_vector` for fixed-capacity storage.
- The exact GCC 16 point release is pinned in the toolchain image, and
  moving to a newer one is a reviewed change gated by the reflection
  conformance tests (`reflection.md`).
- **ARM toolchain.** Distribution and vendor cross compilers will lag
  behind GCC 16, so the project provides its own: a container image with an
  `aarch64-linux-gnu` GCC 16 cross compiler and a sysroot taken from the
  target. Deployments link libstdc++ statically, so the board needs no new
  runtime library; only the sysroot's glibc version has to match the target.
- **Clang** cannot build `rtt3` at all until it implements reflection. No
  CI job depends on it, and clangd and clang-tidy will not parse `rtt3`
  headers or component code in the meantime.
- CI on GitHub Actions, Linux only: GCC 16 on x86-64 and AArch64, with
  ASan, UBSan and TSan jobs.
- New test suite in a current framework (question M9). The existing
  Boost.Test files are not ported, since they exercise an API that no
  longer exists; they are read as a specification when writing new tests.
- A benchmark harness from the start: wake-up latency, port write and read
  latency, operation call cost, start-up time, binary size, compile time.
  RTT 2.x numbers are recorded first for comparison.
