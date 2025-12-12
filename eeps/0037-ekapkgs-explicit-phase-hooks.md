---
EEP: 37
Title: Explicit Phase Hooks
Author: jonringer
Status: Provisional
Type: Standards Track
Topic: Packaging
Created: 2025-12-12
Resolution:
---

# Motivation

Setup hooks are a quality of life features which interact with the stdenv builder
in ways which generally alter the build to be more reflective of the expected
tool. For example, including `cmake` will then automatically change the
`configurePhase` from the `./configure || true` default configure phase to running
the equivalent cmake commands to do a `cmake` configure phase. The problem arises
when these tools are not the entrypoint to a build, the major ecosystem being
python where some builds will want to invoke meson or cmake, but not have it
dominate the build logic. The implicit nature of the configure phase being
replaced is also not intuitive for many users.

To reduce the "weirdness budget" of including a dependency, make these hooks
be specified explicitly.

# Detailed implementation / Specification

See [corepkgs#47](https://github.com/ekala-project/corepkgs/pull/47) for an example for `cmake`. But this would apply to all tools
which require a configure step.

Example replacement:
```diff
  nativeBuildInputs = [
    cmake
+   cmake.configurePhaseHook
  ];
```

# Prior art (optional)

Unknown

# Future work

- Similar treatment for other tools besides meson and CMake.

