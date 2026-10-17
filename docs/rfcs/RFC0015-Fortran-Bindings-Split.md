---
author: Alexis Montoison (amontoison)
created: 2026-07-20
modified: 2026-10-17
status: draft
---

# Distributing Fortran bindings across rocm-systems and rocm-libraries

## Summary

- **Where.** The Fortran bindings move out of `ROCm/hipfort` and next to the headers they mirror: HIP and ROCTx in one `rocm-systems` project, `projects/fortran/`, and each math library in a `fortran/` directory of its `rocm-libraries` project.
- **How.** They are generated from the C headers by a standalone tool (`rocm-fortran-generator`), committed, and kept in sync by a CI hook that suggests the regenerated diff on any header PR. Facts the headers cannot express (for example, whether a pointer argument is a scalar or an array) live in a `metadata/` directory next to each module, and the generator reads them.
- **What ships.** Per library: one module (`use rocblas`, `use hip`), a static `lib<lib>_fortran.a`, the compiled module interfaces (`.mod`) under `include/fortran/<compiler>/`, the generated `.F90` under `share/<lib>/fortran/`, and a CMake package `<lib>-fortran` exporting `roc::rocblas_fortran`, `hip::hip_fortran`, and so on. ROCm ships them precompiled for `amdflang`, its Fortran compiler; other compilers build them from the installed source.
- **How it builds.** Each project builds its own binding behind one guarded switch, `BUILD_FORTRAN_BINDINGS` (ON by default on Linux, skipped without a Fortran compiler); TheRock builds and packages them with the C libraries, using the `amdflang` it builds.
- **When.** The packaged bindings land in ROCm 10.3. `ROCm/hipfort` is maintained beside them through 10.x and removed at 11.0. The hand-written modules some libraries ship (`rocblas_module.f90`, `rocsparse.f90`, ...) are replaced at 10.3.
- **For users.** hipfort users rename their `use` lines and link targets; their calls do not change. The migration guides live in the hipfort documentation.

## Overview

`hipfort` provides Fortran interfaces to the HIP runtime and the ROCm math libraries, declared with `bind(C)`, Fortran's standard mechanism for calling C functions.
Today it lives in a separate GitHub repository, and the interfaces are partially written by hand.
Coverage is only ever as current as the last manual update: it drifts, and it stalls when no one maintains it.

This RFC co-locates the generated bindings in the same repository as the C headers they mirror, and, for each math library, in the same project.
They live in a dedicated `fortran/` directory, not in the `include/` tree, so a change to a `.h` can flow to its `.F90` in the same repository.
The HIP runtime and ROCTx bindings go to `rocm-systems`, in one standalone project, `projects/fortran/`, built against the installed HIP and ROCTx.
Each math library's bindings go to `rocm-libraries`, in a `fortran/` directory of that project.
The generator (`rocm-fortran-generator`) is a standalone build-time tool, in its own repository.
CI keeps everything in sync: when a C header changes, it regenerates the affected module and suggests the diff on the PR, like `clang-format`.

For ROCm 7.14 / 10.0 / 10.1 / 10.2, the goal is narrow: repair the existing hipfort without breaking anything, bringing its bindings current with the headers (they lag by one release).
The real change lands in ROCm 10.3: packaging returns, and the bindings split across `rocm-systems` and `rocm-libraries`.
From then on, the co-located, generated bindings this RFC describes (the *packaged track*) are the deliverable.
hipfort, under its current name and layout (the *compatibility track*), is maintained beside them through the 10.x series and removed at ROCm 11.0.
This document fixes the direction and is the source of truth for the design; the roadmap holds the schedule.

## Background

hipfort is a thin Fortran-to-C translation layer.
Its modules declare `interface ... bind(C)` against the ROCm C symbols, plus convenience wrappers (the *ergonomic* forms, for example ones that take a Fortran array instead of a raw pointer).
It holds no GPU compute.

Everything in this section is the current situation, the hipfort of today.
The RFC changes or removes much of it (the separate repository, the `hipfort_` prefix, the split modules, and the CUDA backend); the roadmap says when.

```
   ┌───────────────────────────────────────────────────────┐
   │  Your Fortran application                             │
   ├───────────────────────────────────────────────────────┤
   │  hipfort: Fortran bind(C) interfaces (*.mod)          │
   │           + generic runtime wrappers (libhipfort*.a)  │
   ├───────────────────────────────────────────────────────┤
   │  ROCm math libraries (.so): librocblas, libhipfft, ...│
   │  HIP runtime         (.so): libamdhip64               │
   ├───────────────────────────────────────────────────────┤
   │  AMD GPU driver / hardware                            │
   └───────────────────────────────────────────────────────┘
```

The sources (`lib/hipfort/*.F90`) are one module per library plus a few base modules:

| File                                                           | Role                                 |
| :------------------------------------------------------------- | :----------------------------------- |
| `hipfort.F90`                                                  | HIP runtime (device, stream, launch) |
| `hipfort_types.F90`, `hipfort_enums.F90`                       | HIP types and enumerations           |
| `hipfort_check.F90`                                            | return-code checking helpers         |
| `hipfort_hipmalloc` / `hipmemcpy` / `hiphostregister`          | generic memory wrappers              |
| `hipfort_{roc,hip}blas` / `fft` / `solver` / `sparse` / `rand` | one module per math library          |
| `hipfort_roctx.F90`, `hipfort_cuda_errors.F90`                 | profiling, CUDA-backend errors       |

After `make install`, under `$CMAKE_INSTALL_PREFIX`:

```
├── lib/fortran/<compiler> (HIPFORT_MULTITOOLCHAIN_LAYOUT=ON)
│   ├── libhipfort-amdgcn.a              ← AMD backend  (default)
│   └── libhipfort-nvptx.a               ← NVIDIA backend (deprecated; compat track only)
├── lib/cmake/hipfort/                   ← find_package(hipfort): hipfort::rocblas ...
├── include/fortran/<compiler>/hipfort/{amdgcn,nvptx}/*.mod ← read on `use hipfort...`
```

The install layout has a toolchain dimension, because a `.mod` (the compiled interface file that a Fortran `use` statement reads) is specific to the compiler, and even the compiler version, that produced it.
Two Fortran compilers cannot share one `.mod`.
`HIPFORT_MULTITOOLCHAIN_LAYOUT` selects the layout.
When ON (the default), each compiler gets its own `<compiler>` subdirectory (for example `lib/fortran/<compiler>/`), so several toolchains coexist under one prefix, as in the diagram above.
When OFF, one `.mod` tree and the `.a` go directly under `lib/`, so a second compiler's install would conflict.
The backend (`amdgcn` for AMD, `nvptx` for NVIDIA) is a second, independent axis, because compiling for CUDA also produces a different `.mod` (see One source set, two backends).
`find_package(hipfort)` resolves to the subdirectory matching the consuming project's compiler and the AMD backend.
A presence check reports which pieces are installed for the user's compiler, so a user can confirm the Fortran extension is available before building.
Under the split, this becomes a per-library check with per-library targets (`find_package(rocblas-fortran)`, `roc::rocblas_fortran`); see Building the bindings.

The root cause is that Fortran has no portable ABI: `.mod` files and compiled `.a` archives are specific to the compiler and its version.
The only stable ABI is C's, exposed by the vendor `.so`, and the `bind(C)` layer calls through it.
That constraint drives the whole packaging design (ship source, plus a precompiled set for `amdflang`), set out in full in the appendix on `.mod`/`.a` incompatibility.

Both `.mod` and `.a` are needed, for different phases.

```
   lib/hipfort/*.F90
         │  compile
         ▼
   ┌────────────────────────────┬────────────────────────────────┐
   │  *.mod  (declarations)     │   *.o  (only for real bodies)  │
   │  interfaces, enums, types  │   generic wrappers             │
   └─────────────┬──────────────┴───────────────┬────────────────┘
   COMPILE phase │                              │ ar → archive
   your_app.F90 ─┘ reads *.mod          libhipfort-amdgcn.a
         │                                      │
         ▼ compile                              ▼ LINK phase
   your_app.o ── unresolved symbols ──► libhipfort-amdgcn.a  (rocblas_dgemm_, rocblas_dgemm_rank_)
                                       ► librocblas.so       (rocblas_dgemm)
                                       ► libamdhip64.so      (HIP runtime)
```

The `.mod` is a compile-time artifact.
It carries declarations.
The `.a` is a link-time artifact and it carries compiled bodies.
A math-library call has two forms.
The raw `bind(C)` interface (`rocblas_dgemm_`) is a pure declaration: it sits in the `.mod`, adds nothing to the `.a`, and its symbol is resolved from the vendor `.so`.
The ergonomic variants (`rocblas_dgemm_rank_` taking Fortran arrays) are real wrappers: they call `c_loc` (which returns the C address of a Fortran variable) and forward to the raw form, so their bodies are compiled into the `.a`.
These wrappers are numerous, around 1900 for rocBLAS alone, alongside the HIP runtime wrappers (`hipMalloc`, `hipMemcpy`, `hipCheck`).
So the `.a` is not just the memory wrappers: every math library with ergonomic variants contributes bodies to it.
Both the `.mod` and the `.a` must ship.

**Why a static archive, not a shared library.**
The `.a` holds thin forwarding shims (each wrapper does `c_loc` and calls the raw `bind(C)` form), so a shared library buys nothing:

- there is nothing substantial to share in memory;
- the caller is bound to the module at compile time through the `.mod`, so changing a wrapper means recompiling anyway;
- a prebuilt `.so` is compiler- and version-locked exactly like a `.mod`.

A `.so` would only add a runtime dependency, and a PLT indirection that blocks inlining a one-line shim.

**Why not fuse with the vendor library.**
We keep the Fortran `.a` separate from the vendor `.so` rather than fusing them: fusing would tie the binding's version to the vendor release and make that `.so` carry Fortran symbols, for no gain (the wrapper object is well under half a percent of `librocblas.so`).
With a separate static `.a`, the linker pulls in only the members that are used, so unused wrappers never reach the binary, and there is no C-side clash, since the C and Fortran objects compile in different directories.

**One source set, two backends.**
The same sources compile twice under the CUDA switch (`-DUSE_CUDA_NAMES`), producing a per-backend archive and `.mod` (`libhipfort-amdgcn.a` for AMD, `libhipfort-nvptx.a` for NVIDIA); a compiled `.mod`/`.o` is backend-specific, so the two cannot share one archive.
The packaged track keeps only the AMD backend.

The hand-written model lags for a simple reason.
A new function, enum, or struct field in a header must be copied by hand, later, by someone who is not the header's author.
The result is delay and silent drift: a missing `_64` variant, a stale struct layout.
It stops entirely when there is no maintainer.

Worse, the hand-written work is already duplicated.
Several math libraries ship their own Fortran module in their source tree.
rocBLAS ships `rocblas_module.f90`, hipBLAS ships `hipblas_module.f90`, rocSPARSE ships `rocsparse.f90` (over 7000 lines) plus `rocsparse_enums.f90`, and hipSPARSE and hipSOLVER ship `hipsparse.f90` (plus `hipsparse_enums.f90`) and `hipsolver_module.f90`.
rocRAND and hipRAND ship a third variant, the deprecated `rocrand_m` / `hiprand_m` wrappers with a small `hipfor` HIP module, gated by `BUILD_FORTRAN_WRAPPER`.
Five of these define a module named exactly after the library (`module rocblas`, `hipblas`, `rocsparse`, `hipsparse`, `hipsolver`).
Three of those (rocBLAS, hipBLAS, rocSPARSE) install it as a public source file; hipSPARSE compiles its module only into its own samples, and hipSOLVER's install guard is never defined.
The packaged track's unprefixed modules therefore supersede and remove them rather than coexist (see What changes for a library repository).
Each is maintained independently of hipfort's binding for the same library.
So `rocblas_dgemm` has two separate Fortran interfaces, in two repositories, kept current by two different people.
Either copy can drift from the C header and from the other.
Co-located generation collapses these parallel copies into one generated source of truth per library, next to the header they both used to track by hand.

rocSPARSE goes further.
It also ships fourteen Fortran example programs (`clients/samples/example_fortran_*.f90`) written against its own module.
Yet its online documentation is entirely C/C++ and surfaces none of them, pointing Fortran users to hipfort instead.
So the same library carries a second Fortran binding and a second example set, maintained apart from hipfort and invisible in its own docs.
Generating the binding in place, with the examples alongside it, folds that duplicated, undiscoverable work back into the library.

## Design principles

**P1. Co-locate bindings with their headers.** The binding for `rocblas_dgemm` should live in the same repo, and change in the same PR, as `rocblas.h`.
Proximity removes the lag.
The binding regenerates when, and where, the C API changes.

**P2. Each library builds its own binding; TheRock packages them.** The deliverable is per-library: each library ships its own `.F90`, `.mod`, `lib<lib>_fortran.a`, and CMake target (`roc::rocblas_fortran`, `hip::hip_fortran`, ...), built by that library's own CMake from its `fortran/` directory, in the same build as the C library.
HIP and ROCTx are the exception: `rocm-systems`' `projects/fortran/` builds them against the installed runtime.
There is no separate job that gathers the bindings from every repository: TheRock, which already builds every project, packages the Fortran artifacts with the C ones (see Packaging in TheRock).
Fusing the pieces into one package, or building only a subset, stays possible, but the default is per-library.
That fixes the sync problem without adding a pipeline: the sources move next to the headers, and the existing build and packaging carry them.

**P3. Keep the generator standalone.**
It is build-time tooling, not a runtime dependency.
It lives in its own repository, `ROCm/rocm-fortran-generator`, which becomes public before the CI hook is enabled, since a public CI job has to run it (see Rollout).
Where it ultimately belongs is recorded in Open questions.
This RFC does commit to one thing: the generator's *output* is checked in as reviewable Fortran in the library repositories, so reading and building the bindings never depends on reaching the generator.
Wherever it lives, it is owned and maintained by the Fortran-bindings team, not the compiler developers, and it is not part of the compiler build.
Library repositories invoke it as a standalone tool.
They do not vendor it.

**P4. CI, not hand-copying, keeps the generated surface current.** A C-library developer should not have to learn Fortran to keep a binding current.
When a header changes, CI regenerates the `.F90` and proposes the diff as a suggested commit, like an auto-formatter, so the generated surface is never stale after merge.
The one manual part is the curated metadata, in each `fortran/metadata/`: most changes need none, and when a new function needs a classification CI flags it for the Fortran-bindings team, who add the one line.

## Non-goals

To keep the scope sharp, the following are explicitly out of scope for this RFC.

- **Windows.** The target is Linux HPC. ROCm's existing Fortran clients are already gated off on Windows in-tree (rocBLAS, hipBLAS, hipSOLVER, and rocSPARSE all condition Fortran on `NOT WIN32` / `UNIX`), so a Windows Fortran product is not a goal.
- **Fortran under sanitizers.** The ASan and coverage build variants ship no Fortran, since Fortran linking is unsupported under ROCm's sanitizer configuration. Today the projects still build the bindings under ASan and only skip their tests; TheRock's shipping build is what turns them off (see Packaging in TheRock).
- **A binding for compilers limited to Fortran 77.** Legacy Fortran 77 code built with a current compiler uses the bindings; only a compiler that stops below Fortran 90 cannot. See Fortran 77 code, under Language standard, for what that means and what such a binding would take.
- **The generator as a runtime or shipped product.** `rocm-fortran-generator` (written in Julia, using Clang.jl to parse headers) is build-time tooling; none of its code reaches a consumer, the compiler build, or the deliverable.
- **Non-CMake consumer tooling as a first-class path.** CMake `find_package` is the supported integration; a `pkg-config` file or a flags script is a convenience, not a guarantee.
- **A new build system or test framework.** This RFC defines where the bindings live and how they stay in sync, reusing the monorepo's existing CMake, CODEOWNERS, and CI substrate rather than introducing a parallel one.

## The generated bindings

The generated modules are what you `use` and link.
This section describes their surface: the module names, how you call them, the Fortran standard they need, and the CUDA option.
The generator that produces them is described next.

### Naming and module structure

The compatibility track keeps today's names and layout: the `hipfort_` prefix and the split into separate interface, type, and enum modules per library.
So `use hipfort_rocblas` keeps working, and the prefix avoids clashing with the vendor's own `rocblas` module.
The packaged track modernizes both: it drops the prefix, and it collapses each library's interfaces, types, and enums into a single module.
Dropping the prefix matches NVIDIA's convention (`use cublas`, not `nvfort_cublas`).
It does not clash with the libraries' own same-named modules (`rocblas_module.f90`, `rocsparse.f90`, ...), because the split removes those, so the generated module is the only `rocblas` in the tree.
The HIP runtime module becomes `hip` (so `use hip`, not `use hipfort`).
The cross-cutting modules `hipfort_check` and `hipfort_auxiliary` disappear; their symbols come from the library module you already `use` (`hipCheck` from `hip`, `rocblasCheck` from `rocblas`; handles are `type(c_ptr)`, see Ambiguous or unmappable C).
Cross-library sharing still works: a library that needs another's types simply `use`s that library's module (rocSOLVER uses rocBLAS).
The generator (`ROCm/rocm-fortran-generator`) emits the packaged, collapsed, unprefixed set committed next to the headers; `ROCm/hipfort` keeps its split, `hipfort_`-prefixed layout until 11.0.
The two schemes coexist at no extra cost.

### Calling styles

One rule applies: each entry point is a single generic (Fortran's overloading mechanism) under its public name.
The compiler picks the form from the actual arguments, so a caller passing a Fortran array never calls `c_loc` itself.

- Raw `type(c_ptr)`: today's hipfort surface, and what OpenMP target data needs.
  Existing code keeps working, nothing breaks.
  As in hipfort, every routine `X` is a generic, and its raw `bind(C)` implementation (the *specific*) is named `X_`, so the raw form also stays callable by that name, and adding an ergonomic form later never changes what `X` or `X_` is.
- Ergonomic forms under the same name: a plain Fortran array, a scalar passed by reference, an optional argument (and, for ROCTx, a `character(kind=c_char)` string) just work.

The ergonomic array forms carry a contract, because they take a Fortran array and forward it with `c_loc`:

- The actual argument must be **contiguous** and have the `TARGET` attribute; otherwise the compiler may pass a temporary copy.
- The rocBLAS/rocSPARSE math APIs are **stream-ordered** (they enqueue and return before the device is done), so for any argument the device reads or writes after the call returns, the **raw `type(c_ptr)` surface is canonical**: a temporary would dangle past the call.
- The generator therefore never emits an ergonomic wrapper that passes `c_loc` of wrapper-local storage (a materialized shape or stride array, a null-terminated string copy) for an argument the device touches after return.

For a device-resident array (`hipMalloc` + `c_f_pointer`) a non-contiguous section is worse than a lifetime bug: packing it is a host read of device memory, which faults on GPUs without XNACK (GPU page-fault retry) enabled, so contiguity there is a correctness requirement, not a performance one.

### Language standard

The floor is Fortran 2003, and two features raise it in well-defined steps:

- **Fortran 2003 (`iso_c_binding`)** is the minimum: `bind(C)`, `type(c_ptr)`, `c_loc`, `VALUE`, interoperable derived types, and `enum, bind(c)`. The raw `type(c_ptr)` interface surface needs nothing more, so a site restricted to F2003 can use it as-is.
- **Fortran 2008** is needed for the ergonomic array forms: assumed-shape array dummies and passing a Fortran array through `c_loc`.
- **Fortran 2018** is needed only for the opt-in assumed-rank variants (`-DFORTRAN_ARRAY_INTERFACES=assumed-rank`, `dimension(..)`).

Minimum compilers: gfortran, amdflang, and Cray CCE; each `fortran/` directory also carries toolchain files for Intel (`ifx`, `ifort`) and NVIDIA HPC (`nvfortran`).
See the appendix for why a `.mod` cannot be shared across compilers, or even across versions of the same compiler.

#### Fortran 77 code

The question comes up often, so here is the answer in one place.

- **Fortran 77 code can use the bindings.**
  What matters is the compiler, not the age or the form of the source.
  A Fortran 77 code base built with a current compiler (gfortran, amdflang, ifx, ...) calls the bindings like any other code: the file that calls them adds one `use rocblas` line, and fixed-form source, implicit typing and the rest of the code are untouched.
- **Only a compiler limited to Fortran 77 cannot.**
  The bindings are Fortran modules built on `iso_c_binding` (Fortran 2003), and a compiler that stops below Fortran 90 has no modules and no `use` statement, so it has nothing to read them with.
- **Nothing is taken away.**
  hipfort never offered a path for such a compiler either.
- **What such a path would take.**
  A separate artifact: a C shim library with Fortran 77 linkage, plus an include file holding the constants as `PARAMETER` values.
  It is feasible, and its cost is bounded, because it would be another emitter of the same generator, reusing the same parsed C declarations and the same curated metadata.
  It is still a project of its own: the emitter, the tests, the packaging and the per-compiler symbol-name conventions are new, the tested surface roughly doubles, and it serves only callers that cannot use the modules.
- **How it would be taken up.**
  If an application needs it, in its own RFC, scoped to that application's entry points rather than the full API; nothing decided here would need to change.
  Leaving it out of this RFC is a matter of scope, not a judgement on the need.

### CUDA backend

The generated modules are `.F90` (uppercase), so the compiler runs the C preprocessor and resolves the CUDA switch.
Compiled with `-DUSE_CUDA_NAMES`, the same sources bind to CUDA instead of HIP.
The generator cannot infer CUDA names from the AMD headers, so hand-written data files feed the `#ifdef` branches.
The main one is a per-library `hipName cuName` map.

The NVIDIA backend lives only in the compatibility track (`ROCm/hipfort`), kept there for HIP-versus-CUDA comparison until ROCm 11.0.
The packaged track does not carry it: not co-located, not packaged, not shipped in ROCm.
A `hip*` library configured for its CUDA backend (hipBLAS, hipSPARSE, hipSOLVER, hipFFT, hipRAND) defaults `BUILD_FORTRAN_BINDINGS` to `OFF`; forcing it on builds the ROCm-generated module with a warning that it is not tested against the CUDA-backed library.

## The generator (rocm-fortran-generator)

`rocm-fortran-generator` is a Julia program that regenerates the whole binding set from the current C headers, using [Clang.jl](https://github.com/JuliaInterop/Clang.jl) (Julia bindings to libclang) to parse them.
It keeps hipfort's routine, enum, and check-routine names, so existing hipfort calls compile unchanged.
Using the modules needs no Julia.
Only regenerating does.

```
   Header sources (one of)
     a rocm-systems checkout         (HIP, ROCTx)
     a rocm-libraries checkout       (the math libraries, HIP from an installed ROCm)
     an installed ROCm               (/opt/rocm/include)
            │  parse as C  (clang -x c, via Clang.jl)
            ▼
   Generator (Julia)
     generate_all.jl  →  common/fortran_gen.jl, common/hip_generics.jl,
                         <lib>/generator.jl + <lib>.toml
                         + the library's fortran/metadata/ from the checkout
            │  emit
            ▼
   fortran/  (committed, packaged track: one module per library, no prefix)
     rocm-systems projects/fortran/:  hip.F90, roctx.F90
     rocm-libraries projects/<lib>/fortran/:  rocblas.F90, hipblas.F90, rocsparse.F90,
       hipsparse.F90, rocfft.F90, hipfft.F90, hipfftw.F90, rocsolver.F90,
       hipsolver.F90, rocrand.F90, hiprand.F90
            →  hip runtime + roctx + 11 libraries, ~5800 functions
```

The engine parses each header as C and emits, per library, the interface blocks, the enum and `#define` constants, the derived types, and the check routine.
The generator handles each library in a separate process.
Each monorepo's CI runs the generator on that monorepo's checkout only: `rocm-systems` with no ROCm installed, `rocm-libraries` with the HIP headers of an installed ROCm.
The headers a build generates (version, config and export headers) are synthesized from the checkout's templates, and any Clang error fails the run.

The profiling binding (`roctx`) sits beside HIP in `rocm-systems`' `projects/fortran/`, moving there from `projects/rocprofiler-sdk/fortran/` (rocm-systems#12700).
It is built against the installed `rocprofiler-sdk-roctx` and exports `roc::roctx_fortran` from a `roctx-fortran` package.
It is a dozen `bind(C)` interfaces with no array overloads, so the array option `FORTRAN_ARRAY_INTERFACES` (see Building the bindings) changes nothing for it.

Regeneration is cheap.
That is what makes the CI-as-formatter model practical, and it means a release's final headers can be picked up at the last moment, not weeks ahead.
The generator is also deterministic: the same headers and metadata produce byte-identical output.

The generator is about 4.2k lines of Julia, plus its tests.
Only the emit stage is Fortran-specific.
The front end, which parses headers with Clang.jl into a language-agnostic description of functions, enums, and structs, could be retargeted to another language by swapping the emitter.

### Free side-effect: C-header hygiene

The generator parses every public header as C (`clang -x c`), because a `bind(C)` interface mirrors the C ABI.
So "the headers are valid C" is a hard prerequisite, checked for free on every run.
ROCm headers are meant to be C-includable, but in practice they are only pulled from C++.
C-only breakage therefore hides for releases.
Typical cases are a C++ keyword used as an identifier, a missing `extern "C"`, or a C++-only include.
The generator fails on such a header, and the CI hook flags the PR that introduced it.
This also benefits the C libraries themselves.
Several such issues have already been caught this way.

### Ambiguous or unmappable C

Not every valid C construct maps cleanly to a `bind(C)` interface.
Where a construct is ambiguous or has no Fortran equivalent, the generator falls back to a defined, ABI-correct form rather than dropping it or guessing.

- **Bit-fields, unions, anonymous members.** A struct that is not field-mappable is emitted as an opaque byte blob of the exact C size and alignment.
  Structs that embed it keep their field offsets, so nothing downstream breaks.
  The cost is that the opaque struct's own members cannot be accessed individually.
  This is how `hipDeviceProp_t` survives, even though it embeds the bit-field struct `hipDeviceArch_t`.
  A struct with an incomplete or zero-size layout is genuinely skipped.
- **Opaque handles.** A pointer to an incomplete struct, such as `rocblas_handle`, is a `type(c_ptr)`, as it was in hipfort.
- **Opaque workspace pointers.** A typed pointer that is really scratch memory is always bound as a `type(c_ptr)` passed by value, never as a Fortran array.
- **C strings.** `char*` is bound as the C API spells it, a `type(c_ptr)`, in every role: an input string, a `const char*` return value (the status-to-string helpers, `hipGetErrorString`), and a caller-supplied output buffer.
  This is hipfort's surface, so code written against hipfort compiles unchanged.
  A library can opt in to a typed form for its input strings.
  ROCTx does: its routines also accept a `character(kind=c_char)` string, passed without a copy, so the caller appends the terminating NUL (`"zone"//c_null_char`).
- **Null pointers as a mode selector.** Across rocSPARSE, rocBLAS, and rocSOLVER, `NULL` is sometimes not an error but a mode selector (the buffer-size query/execute idiom: call once with `nullptr` to get the size, again with the buffer).
  A Fortran array argument cannot be "absent", and `OPTIONAL` (Fortran's marker for an argument the caller may omit) is illegal on the raw surface: F2018 constraint C865 forbids `OPTIONAL` with `VALUE`, and every raw pointer is passed by value.
  So the `<lib>.optional` metadata records which by-reference arguments accept `NULL`.
  For those, the generator adds a plain (non-`bind(C)`) procedure to the same generic, where `OPTIONAL` is legal; it passes `c_null_ptr` for an omitted argument, so omitting it from a call passes `NULL` to C.
  The raw specific `X_` stays callable by name.
- **Complex numbers.** A rocBLAS or rocSPARSE complex is a two-float C struct, which the C type system does not distinguish from any other two-float struct, so mapping it to a Fortran intrinsic `complex` is a curated fact, not something inferred.
  It rests on an ABI assumption, to be checked by the size-and-alignment gate (see the backlog appendix): the struct is passed the same way as `complex(c_float_complex)` on the supported platforms.
  The alternative, a `bind(C)` derived type, would break every existing user and the shipped examples that pass a plain `complex`.
- **Macros and vector types.** Function-like and internal (`__`-prefixed) macros are skipped, while `#define` constants are extracted as parameters.
  The low-level HIP vector types (`float4`, `int4`, and the rest) are skipped, and functions taking them stay `type(c_ptr)`.

Two ambiguities the C type system cannot resolve on its own are left to the curated metadata (see Curated metadata): a scalar host value versus a data buffer (`const double* alpha` versus `double* x`, identical in C), and device-side output scalars.
Invalid C, as opposed to merely ambiguous C, is a separate case: the generator fails on it (see C-header hygiene).

### Curated metadata (the manual surface)

The generator emits everything the C ABI carries: signatures, enums, and structs.
A few Fortran-side facts are not present in the C headers and cannot be inferred.
They live in small per-library files that a maintainer curates, in a `metadata/` directory next to the library's module: `projects/fortran/metadata/` in `rocm-systems`, `projects/<lib>/fortran/metadata/` in `rocm-libraries`.
When the generator runs on a checkout, it reads that directory, so a header change and the metadata line it needs travel in the same PR (P1).
This is the one exception to "the generator removes the hand-copying": the bindings are generated, but this thin layer of per-argument intent stays manual, because the C API does not carry it.
Several files concern the *pointer mode*, a per-handle setting that says whether scalar arguments such as `alpha` and `beta` are read from host or device memory.

| File               | What it encodes                                                                     | Why the headers cannot give it                                        |
| ------------------ | ----------------------------------------------------------------------------------- | --------------------------------------------------------------------- |
| `<lib>.argkinds`   | which pointer args are scalars (`alpha`, `beta`, an output) vs data buffers         | `const double* alpha` and `double* x` are identical in C              |
| `<lib>.ptrmode`    | the subset of those scalars the library reads per its pointer mode (host or device) | the pointer mode is runtime handle state                              |
| `<lib>.devonly`    | scalars the device writes (hipSOLVER's `devInfo`), which get no host form           | the C type does not say which side writes                             |
| `<lib>.cptrargs`   | pointers kept as pass-through `type(c_ptr)` in the array overloads (a dot `result`) | pointer-mode outputs live on the device, so arrayifying them is wrong |
| `<lib>.native`     | which functions get the array (`_rank_*`) overloads, and the max rank               | rank is a Fortran-side choice, absent from the ABI                    |
| `<lib>.optional`   | by-reference arguments the C API accepts as `NULL`                                  | a nullable pointer and a required one are identical in C              |
| `<lib>.byvalue`    | arguments passed by value although their C type is a pointer (HIP's `extra`)        | the C type does not say how the API reads it                          |
| `<lib>.funptrargs` | arguments that are function pointers declared `void*`                               | the C type erased the function type                                   |
| `<lib>.arrayargs`  | pointers to a derived type that are arrays (hipFFTW's `fftw_iodim`)                 | a pointer to one element and to an array are identical in C           |

A library carries only the files it needs: none for ROCTx, one to three for the FFT and RAND libraries, four to six for HIP and the BLAS, SOLVER and SPARSE libraries.
The generator configuration (`<lib>.toml`: name prefix, opaque handles, own headers, check overrides) is project convention rather than header fact; it stays with the generator, which owns the code that reads it.
The CUDA map files (`<lib>.txt`, `<lib>_enums_cuda.txt`, and the per-type `.cuda.finc`) belong to `ROCm/hipfort`, which keeps its CUDA backend until 11.0; they are not part of the packaged track.
The HIP binding's pull request carries its `metadata/` (ROCTx needs none); the `rocm-libraries` pull requests add theirs before they leave draft, and until then the generator keeps a copy of each.

The maintenance cost is bounded, because these files were bootstrapped and are tool-validated, not written from scratch.
Every `.argkinds` was derived from the original hipfort sources in one run, and a script derives the `.ptrmode` and `.devonly` files from `.argkinds`, minus a short per-library list of host-only arguments.
The generator's `apply_argkinds` step warns about scalar-looking arguments that are still unclassified.

The residual manual work is small, and it is maintenance rather than authoring.
A new ROCm function whose scalar pointer argument the bootstrap did not cover must be classified by hand (the warning points at it).

**Keeping the classifications correct.**
The `apply_argkinds` warning catches an *unclassified* argument, but not a *stale* one: an entry that stays well-formed after a function is renamed or its signature changes.
So on each run the generator checks every entry against the parsed C declarations and reports any entry that matches no current function.
Today that report is a warning; once the gate is enforcing (see Rollout), it **fails generation**.
That turns "nothing merges *unclassified*" into "nothing merges *unmatched*".
That leaves one case no such check can catch: an entry that matches but is wrong.
It is covered two ways: a one-time audit of the classifications bootstrapped from hipfort, and runtime tests that exercise **both host-pointer and device-pointer modes** for a representative set of entry points.
Those tests prioritize the pointer-mode-ambiguous families (`alpha`/`beta`/`result`), since a wrong classification there turns into a fault or a silently wrong answer.

### Testing the generator

The generator is validated at three levels:

- unit tests over its pure transforms;
- golden-output comparisons plus the CI compile gates (the committed modules must compile under amdflang and gfortran, in each array configuration);
- a runtime run of the bindings on an AMD GPU.

That runtime run must cover both pointer modes for the `argkinds`-classified entry points, since a misclassified scalar pointer compiles and links cleanly and no compile-only gate can see it (see Keeping the classifications correct).
The full test inventory (file names, gates, and the end-to-end golden) is in the appendix.

## Proposal

```
   BEFORE  (super-repo, always downstream)
     rocm-systems (HIP headers) and rocm-libraries (rocBLAS, ...)
            │  headers copied downstream
            ▼
     hipfort  (separate repo, updated by hand, stalls when unmaintained)
            │
            ▼
     packaging

   AFTER  (co-located sources, built with each library)
     rocm-systems/    projects/hip/*.h, projects/rocprofiler-sdk/.../roctx.h
                      projects/fortran/       hip.F90, roctx.F90, metadata/
     rocm-libraries/  projects/rocblas/*.h  + projects/rocblas/fortran/rocblas.F90, metadata/
                      projects/hipfft/*.h   + projects/hipfft/fortran/hipfft.F90, metadata/
            │  the generator regenerates each .F90 in the SAME PR as its header
            ▼
     each project's build (BUILD_FORTRAN_BINDINGS)
            fortran/<lib>.F90  →  <lib>.mod + lib<lib>_fortran.a
                               →  <lib>-fortran package (roc::rocblas_fortran, hip::hip_fortran, ...)
            ▼
     TheRock: builds every project with amdflang, packages the Fortran
              artifacts with the C ones (P2)
```

Where each piece goes:

| Piece                                                                   | Repo                                        | Why                                                  |
| :---------------------------------------------------------------------- | :------------------------------------------ | :--------------------------------------------------- |
| HIP runtime module `hip` (absorbing HIP types, enums, check, auxiliary) | `rocm-systems`, `projects/fortran/`         | HIP lives here (P1)                                  |
| ROCTx module `roctx`                                                    | `rocm-systems`, `projects/fortran/`         | ROCTx lives here (P1)                                |
| Each math library's `<lib>` module (`rocblas`, `hipfft`, ...)           | `rocm-libraries`, `projects/<lib>/fortran/` | next to its C headers (P1)                           |
| Curated metadata (`argkinds`, `native`, `optional`, ...)                | the module's `fortran/metadata/`            | manual, but moves with the header (P1)               |
| Generator configuration (`<lib>.toml`)                                  | the generator                               | project convention, read by the generator's code     |
| CUDA maps (`<lib>.txt`, `<lib>_enums_cuda.txt`, `*.cuda.finc`)          | `ROCm/hipfort` (compatibility track)        | CUDA is not in the packaged track (see CUDA backend) |
| The generator (`rocm-fortran-generator`)                                | its own repository                          | build-time Fortran tooling (P3)                      |
| Build (per-library `.mod`/`lib<lib>_fortran.a`, CMake package)          | each project's `fortran/CMakeLists.txt`     | per-library deliverable (P2)                         |
| Packaging                                                               | TheRock                                     | already builds and packages every project (P2)       |

The CI hook runs the generator on the changed headers only:

```
   Developer opens a PR editing  projects/rocblas/.../rocblas.h
            │
            ▼
   CI hook: run the generator on the changed header(s) only
            │
            ├─ generated .F90 unchanged  → OK, check passes, nothing to do
            │
            ├─ generated .F90 differs    → bot pushes a SUGGESTED CHANGE:
            │                                the regenerated rocblas.F90 diff
            │                                (same UX as clang-format-diff)
            │                                dev clicks "commit suggestion" → in sync
            │
            └─ header no longer valid C   → FAIL, generation stops and flags the
                                             C-header regression (see C-header hygiene)
```

The hook is path-filtered.
It runs only when a PR touches a library's public C headers (`*.h`, the files the generator parses) or its `fortran/` directory (the generated binding and the curated metadata), so unrelated PRs never pay for it.
For `rocm-systems`' `projects/fortran/`, the headers are HIP's and ROCTx's, in their own projects.
The `.hpp` C++ headers are not parsed, so they do not trigger it.
Some changes reach beyond one library: a shared header, such as the HIP runtime or the rocBLAS headers that rocSOLVER includes, can shift other libraries' bindings, and a change to the generator itself can shift all of them.
A nightly full regeneration backstops the per-PR hook and catches that cross-header drift.

This rolls out in two stages.
First *advisory*: the bot suggests the diff but does not block the merge.
Then *enforcing*: a required `bindings-in-sync` check, like a `clang-format` gate, fails a header change that lacks its regenerated binding.

### Maintaining the bindings under the split

A binding is more than its `.F90`, and the split keeps all of it next to the library.

- **Generated `.F90`.** Derived from the header.
  The CI hook regenerates it and suggests the diff, and no one edits it by hand (P4).
- **Curated metadata.** The manual part, the intent the C API cannot express (see Curated metadata).
  Co-located in `fortran/metadata/`, which the generator reads when it runs on the checkout, so a header change carries its metadata tweak in the same PR (P1).
  Most header changes need none; a still-unclassified scalar pointer argument is flagged on the PR, so it cannot merge silently.
  A classification that matches its function but is wrong is caught instead by the runtime tests and CODEOWNERS review of the metadata (see Keeping the classifications correct).
- **Tests.** A library's Fortran tests live in its `fortran/test/`, all on one convention (see Tests and documentation).
  Coverage is uneven.
  HIP, ROCTx, rocSOLVER, hipSOLVER, rocSPARSE, hipSPARSE, rocFFT, hipFFT, hipFFTW, rocRAND and hipRAND carry a `fortran/test/`, most of it ported from hipfort.
  rocBLAS and hipBLAS are exercised for now only through their existing `clients/` (the gtest Fortran shims and samples, which now link the generated binding).
  Filling that gap is part of the rollout, not a precondition for co-location.
  The per-PR hook compile-checks them without a GPU; the packaging or validation job runs them on an AMD host.
- **Docs and examples.** The libraries' existing Fortran examples (the rocBLAS, hipBLAS, rocSPARSE and hipSPARSE samples) move to the generated module.
  The packaged modules carry no docstrings yet (see Tests and documentation).
  The current hipfort documentation can likewise be split across the libraries it covers.

Each project builds its binding in its own build (HIP and ROCTx in `rocm-systems`' `projects/fortran/`).
TheRock, which builds every project nightly and on release branches, compiles them in the C dependency order for the AMD backend.
Each build produces, per library, a `.mod` set and a static archive `lib<lib>_fortran.a` (`librocblas_fortran.a`, `libhip_fortran.a`, ...) with its CMake target (`roc::rocblas_fortran`, `hip::hip_fortran`, ...), installed under `lib/fortran/<compiler>/` and `include/fortran/<compiler>/`.
The deliverable is per-library: `find_package(rocblas-fortran)` then gives the Fortran binding and, through it, the C library.
Fusing everything into one package, or shipping only a subset, is an option (P2), not the default.
The packaged track does not build the nvptx backend (see CUDA backend); that backend stays source-only in the compatibility track until ROCm 11.0.
Because the package has no CUDA backend, the `.mod` files land directly in `include/fortran/<compiler>/`, without the compatibility track's inner `hipfort/<backend>/` nesting.

### Building the bindings

The Fortran sources live in each library's `fortran/` directory, and a dedicated, self-contained `fortran/CMakeLists.txt` builds them.
It does `enable_language(Fortran)`, compiles the modules, and produces the `.mod` and `.a`.
It requires a recent CMake, and the versions in use already suffice:

- the in-scope `rocm-libraries` projects declare between 3.13 (rocSOLVER) and 3.25.2 (hipSPARSE, and the monorepo root);
- TheRock requires 3.25, and its builders run 3.31;
- all of these handle Fortran module dependency ordering and `Fortran_MODULE_DIRECTORY`;
- a build using the Ninja generator wants CMake >= 3.20 for reliable Fortran module dyndep (dynamic dependency scanning).

Every project that carries a binding (each in-scope `rocm-libraries` project, and `projects/fortran/` in `rocm-systems`) exposes one uniform switch, `BUILD_FORTRAN_BINDINGS`, **ON by default on Linux but guarded**:

- its default is `${UNIX}` (Windows ships no Fortran compiler; see Non-goals);
- it builds the bindings when a Fortran compiler is present;
- it silently skips them when one is not, so a C-only site with no Fortran compiler still configures and builds.

The guard reuses a probe the same configure already ran:

- in `rocm-libraries`, `ROCM_LIBS_HAVE_FORTRAN` from the root `CMakeLists.txt`;
- in `rocm-systems`, which has no root probe, `ROCM_HAVE_FORTRAN` from `shared/cmake/ROCmFortran.cmake` (added with the HIP binding and included by `projects/fortran/`);
- in a standalone per-project build, its own `check_language(Fortran)`, preferring the `amdflang` found beside the C/HIP compiler or under `ROCM_PATH` when neither `CMAKE_Fortran_COMPILER` nor `FC` is set.

In `rocm-libraries` the project root adds `fortran/` (`rocm-systems`' `projects/fortran/` is a project of its own), and `fortran/CMakeLists.txt` declares the options, runs the guard, and calls `enable_language(Fortran)` only when the switch is ON and a compiler exists; otherwise it prints a `STATUS` line and moves on, never a fatal error.
That is how ON-by-default stays compatible with P4: no C build is ever *forced* to acquire a Fortran compiler, it just builds the bindings wherever it already can, and a C developer who has a Fortran compiler but does not want them passes `-DBUILD_FORTRAN_BINDINGS=OFF`.
Each switch also has a per-library form, `<LIB>_BUILD_FORTRAN_BINDINGS` (for example `ROCSPARSE_BUILD_FORTRAN_BINDINGS`), which overrides the global one so a single project can differ from the rest of a monorepo build.

Two more options complete the set, identical in every project:

- `FORTRAN_ARRAY_INTERFACES` (`none`, `assumed-shape`, `assumed-rank`; default `assumed-shape`) selects the array tier, replacing hipfort's two booleans `HIPFORT_USE_FPOINTER_INTERFACES` and `HIPFORT_ASSUMED_RANK` (see Open questions for why assumed-rank replaces rather than adds). ROCTx has no array overloads, so it does not take the option.
- `BUILD_FORTRAN_CLIENTS` (default `ON`, per-library `<LIB>_BUILD_FORTRAN_CLIENTS`) builds the binding's own tests in `fortran/test/`.
  rocBLAS, hipBLAS and rocSPARSE already used that name for the Fortran part of `clients/`, and it keeps that meaning; asking for clients with the bindings off skips them, never an error.
  In-tree, the binding tests also follow the project's own test switch (the library's `BUILD_CLIENTS` / `BUILD_CLIENTS_*`, `BUILD_TEST` for rocRAND and hipRAND, `BUILD_TESTING`), and hipSPARSE's Fortran samples stay under its existing `HIPSPARSE_ENABLE_FORTRAN`.
  `projects/fortran/` in `rocm-systems` has no other test switch: `BUILD_FORTRAN_CLIENTS` alone gates its tests.

Each `fortran/` is also a self-contained CMake project (`project(<lib>-fortran LANGUAGES NONE)` when top-level), so a binding builds against an installed ROCm without rebuilding the C library.
`rocm-systems`' `projects/fortran/` is only ever built that way: it finds the installed HIP (`find_package(hip)`) and ROCTx, and no other project adds it.

```bash
cmake -S projects/rocblas/fortran -B build/rocblas-fortran \
  -DCMAKE_Fortran_COMPILER=gfortran -DCMAKE_PREFIX_PATH=/opt/rocm
cmake --build build/rocblas-fortran
# rocm-systems: HIP (and ROCTx), against the installed runtime
cmake -S projects/fortran -B build/fortran -DCMAKE_PREFIX_PATH=/opt/rocm
```

A build-tree result variable `<LIB>_HAVE_FORTRAN_BINDINGS` (for example `ROCSPARSE_HAVE_FORTRAN_BINDINGS`) is true only when this library's bindings were actually built.
It is deliberately distinct from the root's `ROCM_LIBS_HAVE_FORTRAN`, which only reports that a Fortran compiler exists: a compiler can be present while a given library has `BUILD_FORTRAN_BINDINGS=OFF`.
TheRock, which always has amdflang, builds every library this way.

The guard is small, but each project's `fortran/CMakeLists.txt` currently repeats it rather than sharing it.
`rocm-systems` gets `shared/cmake/ROCmFortran.cmake` for the compiler probe with the HIP binding (rocm-systems#11923).
In `rocm-libraries`, hipSPARSE, rocSOLVER and hipSOLVER carry the amdflang preference in a `fortran/cmake/ROCmFortranCompiler.cmake`, while the other projects inline it.
Folding the probe, the guard, and `add_subdirectory(fortran)` into one helper (in rocm-cmake or a shared module) is implementation backlog; its logic is:

```cmake
option(BUILD_FORTRAN_BINDINGS "Build the generated Fortran bindings" "${UNIX}")
if(DEFINED <LIB>_BUILD_FORTRAN_BINDINGS)
  set(BUILD_FORTRAN_BINDINGS ${<LIB>_BUILD_FORTRAN_BINDINGS})
endif()
if(BUILD_FORTRAN_BINDINGS)
  # reuse the probe the root already ran, otherwise probe locally (amdflang first)
  if(NOT DEFINED ROCM_HAVE_FORTRAN AND NOT DEFINED ROCM_LIBS_HAVE_FORTRAN)
    include(CheckLanguage)
    check_language(Fortran)
  endif()
  if(CMAKE_Fortran_COMPILER)
    enable_language(Fortran)
    add_subdirectory(fortran)
    set(<LIB>_HAVE_FORTRAN_BINDINGS TRUE)  # e.g. ROCSPARSE_HAVE_FORTRAN_BINDINGS
  else()
    message(STATUS "BUILD_FORTRAN_BINDINGS=ON but no Fortran compiler; skipping")
  endif()
endif()
```

For discovery, the Fortran target ships as its own per-library config package, `<lib>-fortran`, so a consumer writes `find_package(rocblas-fortran)`, which `find_dependency()`s the C library.
The `<lib>-fortran-config.cmake` filename is reused from rocRAND and hipRAND, which already install it beside their C config (today their C config `include()`s it; it is not found on its own).
The config file sits beside the C one (`<libdir>/cmake/rocblas/rocblas-fortran-config.cmake`), with a forwarder in `<libdir>/cmake/rocblas-fortran/` so that `find_package` resolves the hyphenated name; HIP and ROCTx install theirs directly in `<libdir>/cmake/<lib>-fortran/`.
Keeping it beside the C config is what allows the next step: once rocm-cmake grows an `include(<lib>-fortran-config.cmake OPTIONAL)` hook in the generated C config, `find_package(rocblas)` alone will expose `roc::rocblas_fortran` when the binding is installed and ignore it otherwise. Until then the `-fortran` package is the entry point, and consumer code written against it keeps working afterwards.
The same is possible for HIP and ROCTx, whose C configs come from clr and rocprofiler-sdk rather than from rocm-cmake: one optional line in each (`include("${CMAKE_CURRENT_LIST_DIR}/../hip-fortran/hip-fortran-config.cmake" OPTIONAL)`, run only when the consumer has enabled Fortran) would let `find_package(hip)` expose `hip::hip_fortran`.
It touches those two projects, which the bindings otherwise stay out of, so it is a possibility to propose to their owners later, not part of this RFC; until then `find_package(hip-fortran)` and `find_package(roctx-fortran)` are the entry points.
Each library's Fortran target sits in its own C library's existing CMake namespace: `roc::rocblas_fortran` and `roc::hipsolver_fortran`, but `hip::hip_fortran`, `hip::hipfft_fortran`, and `hip::hiprand_fortran`. This is per-library, not a hip-versus-roc rule (hipBLAS, hipSPARSE, and hipSOLVER are `roc::`); the full list is in Migration and compatibility. There is no per-library `<lib>::` namespace and deliberately no umbrella `rocm-fortran::` one, since the bindings have no shared deliverable beyond the per-library ones.
The consumer-facing presence flag keeps the variable name already in the tree, `<pkg>_FORTRAN_FOUND` (rocRAND and hipRAND set `rocrand_FORTRAN_FOUND` / `hiprand_FORTRAN_FOUND` today, from `find_package(<lib>)`); it is now defined by the Fortran config package, and it is `FALSE` when the package is installed but holds nothing for the consumer's compiler.
The config requires Fortran to be enabled before `find_package`, and when no `.mod` matches the consumer's compiler it fails with a message naming the compilers that are installed and the `<LIB>_FORTRAN_COMPILER_DIR` override, so a user can tell a missing rebuild from a wrong path.
Each per-library Fortran package is versioned with its C library (`SameMajorVersion`), so a binding resolves to the same version as the C library it was generated from; a static archive carries no soname, so this package version is the version handle.
This generalizes what hipSOLVER already does: it defaults the switch to `${UNIX}` today, so the default is unchanged there.
The design adds the compiler guard so that default is safe everywhere, and redefines the target as the generated binding: the hand-written `hipsolver_module.f90` is removed under the split, and its shared `libhipsolver_fortran.so` becomes the static `libhipsolver_fortran.a` under the same `roc::hipsolver_fortran` name.
It also retires hipSOLVER's `EXPORT_FORTRAN_BINDINGS` and `BUILD_FORTRAN_MODULE`, with the module they gated.
One consequence for hipSOLVER users: `roc::hipsolver_fortran` was exported by `find_package(hipsolver)` (through `EXPORT_FORTRAN_BINDINGS`, ON by default) and no longer is; a consumer moves to `find_package(hipsolver-fortran)` until the rocm-cmake hook lands.
The reference build of the shipped bindings is TheRock's, which builds every project, and so every `fortran/`, in dependency order.

Cross-module dependencies are handled the same way the C libraries already handle them.
`rocsolver` uses `rocblas`, because `rocsolver-functions.h` includes `rocblas.h` and the rocSOLVER API takes `rocblas_handle` and rocBLAS enums.
That is the existing C dependency graph, now in Fortran, not a new problem.
TheRock compiles rocBLAS before rocSOLVER; a standalone per-repo build reuses the `roc::rocblas_fortran` target when both are configured in one tree, and otherwise resolves it with `find_package(rocblas-fortran)`, exactly as the C build depends on `librocblas`. The rocBLAS `.mod` must come from the same compiler, so a non-amdflang site builds the two bindings in that order.
Because each library is a single module under the packaged track, `rocsolver` simply `use`s `rocblas`; the dependency is kept as-is, matching the C graph.

#### Options and tests at a glance

Every project that carries a binding takes the same options (the per-library spelling uses the project's prefix, `ROCSPARSE_`, `HIPFFT_` and `HIPFFTW_`, `HIP_`, ...):

| Option                         | Default                                             | Effect                                                                                 |
| :----------------------------- | :-------------------------------------------------- | :------------------------------------------------------------------------------------- |
| `BUILD_FORTRAN_BINDINGS`       | `${UNIX}` (`OFF` on a hip\* library's CUDA backend) | build the binding; skipped, with a `STATUS` line, without a Fortran compiler           |
| `<LIB>_BUILD_FORTRAN_BINDINGS` | unset                                               | per-library override of `BUILD_FORTRAN_BINDINGS`                                       |
| `FORTRAN_ARRAY_INTERFACES`     | `assumed-shape`                                     | array overloads: `none`, `assumed-shape` (Fortran 2008), `assumed-rank` (Fortran 2018) |
| `BUILD_FORTRAN_CLIENTS`        | `ON`                                                | build and register `fortran/test/`; in-tree, also needs the project's test switch      |
| `<LIB>_BUILD_FORTRAN_CLIENTS`  | unset                                               | per-library override of `BUILD_FORTRAN_CLIENTS`                                        |
| `<LIB>_FORTRAN_COMPILER_DIR`   | from the compiler ID                                | the `<compiler>` install subdirectory, to keep a second version of one compiler apart  |
| `<LIB>_HAVE_FORTRAN_BINDINGS`  | (result)                                            | true when this library's binding was built                                             |

A few projects add a guard of their own: rocSPARSE builds no binding under `BUILD_ROCSPARSE_ILP64`, rocSPARSE installs nothing under `BUILD_CLIENTS_ONLY`, hipSPARSE nothing under `HIPSPARSE_ENABLE_HOST=OFF`, and `projects/fortran/` takes its install component from `HIP_FORTRAN_INSTALL_COMPONENT` (default `dev`).

To build a binding with its tests and run them, standalone against an installed ROCm:

```bash
cmake -S projects/rocsparse/fortran -B build/rocsparse-fortran -DCMAKE_PREFIX_PATH=/opt/rocm
cmake --build build/rocsparse-fortran
ctest --test-dir build/rocsparse-fortran -L gpu    # on an AMD GPU; -LE gpu runs the rest elsewhere
```

The tests of a `rocm-libraries` binding need the HIP binding installed (`find_package(hip-fortran)`); put its prefix on `CMAKE_PREFIX_PATH`.

### Layout after the split

**Source tree (committed).** Each math library owns a `fortran/` directory in its `rocm-libraries` project, and `rocm-systems` has one `projects/fortran/` for HIP and ROCTx.
Each holds the generated modules, their curated metadata, their build files, and their tests.
The generator *code* is never vendored here (P3); only the metadata it reads is co-located, in `metadata/` (see Curated metadata).

```
rocm-systems/  projects/fortran/        a standalone project, built against the installed HIP and ROCTx
   hip.F90                       generated: module hip (runtime + types/enums/check/auxiliary)
   roctx.F90                     generated: module roctx
   metadata/                     curated: hip.argkinds, hip.optional, hip.byvalue, hip.funptrargs
   CMakeLists.txt                self-contained build → libhip_fortran.a + hip.mod, libroctx_fortran.a + roctx.mod
   hip-fortran-config.cmake.in   find_package export → hip::hip_fortran
   roctx-fortran-config.cmake.in find_package export → roc::roctx_fortran
   cmake/toolchains/             example toolchain files (amdflang, gnu, cray, intel, intel-classic, nvhpc)
   test/                         Fortran tests

rocm-libraries/  projects/rocblas/fortran/
   rocblas.F90                   generated: module rocblas (one module)
   metadata/                     curated: rocblas.argkinds, .cptrargs, .native, .ptrmode
   CMakeLists.txt                → librocblas_fortran.a + rocblas.mod
   rocblas-fortran-config.cmake.in
   cmake/toolchains/             example toolchain files
   test/                         Fortran tests (use hip: need the HIP binding)
```

HIP and ROCTx share `projects/fortran/` rather than living under `projects/hip/` and `projects/rocprofiler-sdk/`: the bindings stay out of those projects' builds, and the directory has the same shape as a library's `fortran/` in `rocm-libraries`.
HIP's binding is in review in rocm-systems#11923; ROCTx's (rocm-systems#12700) moves there from `projects/rocprofiler-sdk/fortran/`.
hipFFT's `fortran/` holds two bindings and installs two packages from one build, each with its own `HIPFFT_*` / `HIPFFTW_*` overrides:

- `hipfft-fortran`: `hip::hipfft_fortran`, `libhipfft_fortran.a`;
- `hipfftw-fortran`: `hip::hipfftw_fortran`, `libhipfftw_fortran.a`.

Both configs sit in `<libdir>/cmake/hipfft/`, with forwarders in `<libdir>/cmake/hipfft-fortran/` and `<libdir>/cmake/hipfftw-fortran/`.

The CUDA metadata (`<lib>.txt`, `<lib>_enums_cuda.txt`, `*.cuda.finc`) does **not** live here: the packaged track has no CUDA backend, so those files belong to `ROCm/hipfort`, which keeps its CUDA backend until 11.0.
The CODEOWNERS entry is not a file in `fortran/` either; in `rocm-libraries` it is a line in the project's own `.github/CODEOWNERS` (each project keeps one, and a merge job assembles the monorepo root file from them), and in `rocm-systems`, whose projects keep none, a line in the root `.github/CODEOWNERS`.

**Install tree.** Because a `.mod` is compiler- and version-specific (see appendix), artifacts install under a per-compiler subdirectory, one archive and one module set per library:

```
<prefix>/
   <libdir>/fortran/<compiler>/        librocblas_fortran.a, libhip_fortran.a, libroctx_fortran.a, ...
   <includedir>/fortran/<compiler>/    rocblas.mod, hip.mod, roctx.mod, ...
   <datadir>/rocblas/fortran/          rocblas.F90          ← the generated source
   <datadir>/hip/fortran/              hip.F90
   <datadir>/roctx/fortran/            roctx.F90
   <libdir>/cmake/rocblas/             rocblas-fortran-config.cmake  → roc::rocblas_fortran
   <libdir>/cmake/rocblas-fortran/     forwarder, so find_package(rocblas-fortran) resolves
   <libdir>/cmake/hip-fortran/         hip-fortran-config.cmake      → hip::hip_fortran
   <libdir>/cmake/roctx-fortran/       roctx-fortran-config.cmake    → roc::roctx_fortran
```

`<libdir>`, `<includedir>` and `<datadir>` are the GNUInstallDirs values (`lib64` on Fedora, RHEL and SUSE); a stock ROCm install gives `/opt/rocm/include/fortran/amdflang/` and `/opt/rocm/lib/fortran/amdflang/`.
There is no `hipfort/<backend>/` nesting: the packaged track has a single backend, so the `.mod` files sit directly under `include/fortran/<compiler>/`.
The `rocm-libraries` configs sit beside their C library's, so that the C config can later pick them up (see Building the bindings); HIP's and ROCTx's come from `projects/fortran/`, a project of their own, and stay in their own directories rather than inside another project's.
Every file installs in the project's development component (`devel` in `rocm-libraries`; `dev` for `projects/fortran/`, HIP's name for it), the one that holds the C library's headers and CMake config, so the bindings land in the same development package as the library they bind.
`projects/fortran/` installs both HIP and ROCTx in that one component, so both land in its own development package rather than in HIP's or `rocprofiler-sdk-roctx`'s.
That gives one project, one TheRock artifact and one package; the artifact can still be split by path if ROCTx ever needs its own package.
`<compiler>` is the name a user types, derived from the compiler ID (`amdflang`, `gfortran`, `ftn`, `ifx`, `ifort`, `nvfortran`, else the lowercased basename), so several compilers can share one prefix.
The name ignores the compiler version; `<LIB>_FORTRAN_COMPILER_DIR` (for example `gfortran-13`) gives a second version of one compiler its own directory.
`find_package(rocblas-fortran)` resolves the subdirectory matching the consuming project's compiler, so CMake users never write these paths; a non-CMake build passes them as `-I` and `-L`.

**The generated `.F90` installs too, under `share/<lib>/fortran/`** (`share/hipfft/fortran/` holds both `hipfft.F90` and `hipfftw.F90`).
A user on a compiler other than the one ROCm ships bindings for has to build the binding themselves, and shipping the source means they do not have to clone a monorepo to get it.
It also removes a failure mode: source obtained separately can be generated from a different ROCm than the installed `.so`, which surfaces as an undefined symbol at link time or, worse, as an interface that differs silently from the one the caller meant.
Installed source matches the installed library by construction.

The source goes to `share/<lib>/fortran/` rather than `include/`, for three reasons.

- **It is not in the compiler-partitioned tree.** Every entry under `include/fortran/` answers "which compiler is this for?", and the `.F90` is the one artifact that is compiler-*independent*. Placing it there is a category error, with a concrete symptom: a config that enumerates `include/fortran/*` to report which compilers are installed would list `rocblas.F90` as if it were a compiler, and the user would be invited to select it.
- **Room to grow.** If the Fortran side ever ships more than the module (a second module, an example), `share/<lib>/fortran/` holds them together. `include/<lib>/` would scatter them among C headers.
- **Packaging.** `include/` is the development-headers namespace that tooling walks; a Fortran source in it is noise for anything scanning the include path. `share/` is defined as architecture-independent read-only data, which is what a source file is.

This is a deliberate break from where the hand-written modules used to live (`include/rocsparse/rocsparse.f90`, `include/hipblas/hipblas_module.f90`).
Those were headers-by-convention that a user compiled into their own build; the generated module is a source artifact of the library, and the filenames change in the move anyway, so there is no drop-in path to preserve.

### Tests and documentation

**Tests.** hipfort's test suite splits across the libraries it covers: each library's Fortran tests move to its `fortran/test/`, next to the binding they exercise (rocBLAS and hipBLAS are not there yet; see Maintaining the bindings under the split).
Every math library's `fortran/test/` follows one convention:

- one Fortran program per test (`<name>.f03`), each its own CTest test `<lib>_fortran_<name>`, labelled `gpu` so `ctest -LE gpu` leaves them out on a host without a GPU;
- built under `BUILD_FORTRAN_CLIENTS` (and, in-tree, the project's own test switch), skipped under ASan;
- linked against the binding, the C library and the HIP binding: the tests `use hip`, found with `find_package(hip-fortran)`, so until the HIP binding (rocm-systems#11923) is installed they configure as skipped with a `STATUS` message;
- a test that needs what the host lacks (a second GPU, a feature of the device) can exit with code 77, which CTest reports as skipped.

rocRAND and hipRAND are no exception.
Their former Fortran tests (`test/fortran/`, built with the FRUIT framework against the deprecated `rocrand_m` / `hiprand_m` wrappers) move into `fortran/test/` like everyone else's, ported to the generated module: the generator-by-distribution programs from hipfort, and the FRUIT suite kept as one more CTest test of the same kind.
Their one test that touches no device, the library version check, is their only test without the `gpu` label, and it does not `use hip`.

The `rocm-systems` tests in `projects/fortran/test/` differ only where they must: HIP's link the in-tree `hip_fortran` rather than finding it, and ROCTx's single test needs no GPU.

The per-PR hook compile-checks them without a GPU (this is what catches interface drift), and the packaging or validation job runs them on an AMD host.
None of the projects installs its Fortran test executables yet; they reach a library's `test` component when it does (see Packaging in TheRock).
A test that spans libraries (a rocSOLVER test that `use`s rocBLAS) follows the same dependency order as the bindings, resolved the same way: build order in TheRock, `find_package` for a standalone build.

**Documentation.** The packaged modules carry no docstrings today.
A Fortran routine has the same name, arguments and semantics as the C function it binds, so the library's C reference documents it.
Each library's documentation should state how the Fortran API maps onto it (module name, generic forms, link target), as the rocSPARSE and hipSPARSE pull requests start doing in their usage pages.
The headers' Doxygen comments can be emitted as `!>` docstrings, as hipfort's modules carry them; turning that on for the packaged modules, and publishing a Fortran reference from them, is open (see Open questions).

**Do the docs get their own subfolder in `fortran/`?** No. Any docstrings live in the `.F90`, and the *doc build* is a whole-library concern (the C API and the Fortran API belong in one site), so it stays at the library's existing top-level `docs/`, which would add `fortran/*.F90` to its Doxygen input. We deliberately do not stand up a second doc-build system inside `fortran/`: co-locate the content (P1), not a parallel toolchain. So `fortran/` holds `test/`, but no `docs/`.

**Dependency impact on the doc build.** If the packaged modules carry docstrings (see Open questions), cross-references are where library dependencies reach the docs: a rocSOLVER docstring that mentions `rocblas_handle` resolves only if rocBLAS's Doxygen tag file (the index a Doxygen build exports so other projects can link to its symbols) or Sphinx inventory is available when rocSOLVER's docs are built.
So the doc build carries the same dependency graph as the code, expressed as tag-file and inventory dependencies rather than `.mod` dependencies.
That is the argument for assembling the site in one aggregating job (the doc job that already has every project's output) rather than building each library's Fortran doc in isolation: an isolated build cannot resolve a cross-project reference without the other project's tag file.
The remaining mechanics (portal integration, the exact tag-file wiring) need the ROCm documentation team; see Open questions.

## What changes for a library repository

For a `rocm-systems` or `rocm-libraries` project, the delta is small and bounded.

- A `fortran/` subdirectory holds the generated binding, its curated metadata (`fortran/metadata/`), its build, and its Fortran tests (`fortran/test/`).
  In `rocm-systems`, HIP and ROCTx share one such directory, `projects/fortran/`, instead of one per project.
- A path-filtered CI check runs the generator, but only when a PR touches that library's `*.h` or its `fortran/` directory.
  Unrelated PRs are untouched.
- A CODEOWNERS entry gives the Fortran-bindings team ownership of `fortran/`.
- C and C++ developers do not write Fortran.
  The bot proposes regenerated bindings, and the bindings team owns the manual metadata.
- In the advisory stage (see Rollout and success criteria) a Fortran failure never blocks a C-only PR.
  In the enforcing stage the gate only fails a header change that skips its regenerated binding.
- Any Fortran module the library ships today (for example `rocblas_module.f90`) is superseded by the generated binding.
  Its removal is coordinated with the bindings team, not forced on the C developers.
  The removal covers the module under `library/` and its `<lib>_enums` companion, with no compatibility shim (`use rocsparse_enums` becomes `use rocsparse`).
  For rocRAND and hipRAND it also covers `BUILD_FORTRAN_WRAPPER` and the wrappers it built, and their own Fortran tests move to `fortran/test/`.
  The Fortran test shims in `clients/` stay, but link the generated target instead of compiling the hand-written source.
- The generated module follows the C API more closely than the hand-written one, so a few call sites change for users of the hand-written module: output scalars are typed and passed by reference instead of through `c_loc`, enum arguments use the enum's integer kind, and handle creation takes the handle by reference.
  The last one needs care: the old `rocblas_create_handle(c_loc(handle))` still compiles but leaves the handle unset.
  Each library's CHANGELOG lists these changes.

## Dependencies

The generator needs Julia, Clang.jl, and the C headers.
It is a standalone, build-time tool that regenerates the committed `.F90`; it is never part of a consumer build or of the compiler build.
No Julia dependency reaches the compiler, ROCm, or any consumer: using the committed `.F90` needs only a Fortran compiler.
Consumers are unchanged: they still link `libamdhip64` and the relevant math-library `.so`.

The coupling to Clang is deliberately shallow.
The generator parses public headers at the C API surface (`clang -x c`), not Clang's internal AST, so it needs Clang only to parse C, not a specific Clang version.

Across repositories, `rocm-systems` and `rocm-libraries` gain a `fortran/` subdirectory and a CI hook that invokes the generator.
TheRock builds both but never runs the generator.

## Version targeting and roadmap

The bindings target whichever ROCm version's headers the generator reads: the `develop` branch of the `rocm-systems` and `rocm-libraries` monorepos, or a ROCm install.

The immediate step is bringing hipfort's committed bindings current with the ROCm 7.14, 10.0, 10.1 and 10.2 headers.
Packaging targets 10.3.
This RFC fixes the destination and the order of steps, not a packaging date.

The transition runs on two tracks, both generated from the same headers: the compatibility track and the packaged track.
The compatibility track is `ROCm/hipfort` with its current layout: the `hipfort_` prefix, the CUDA backend, and the split modules (interfaces, types, enums).
It is kept as source, not packaged, so existing users who build it themselves see no break through ROCm 7.14, 10.0, 10.1, and 10.2.
It stays **maintained beside the packaged track through the whole 10.x series**, so that from 10.3 to 11.0 a user can migrate on their own schedule, one file at a time.

A packaged binding is versioned with its library and holds for a major ROCm series: a binding built for 10.3 is valid on 10.3 and any later 10.x, and its CMake package accepts any version with the same major number (`SameMajorVersion`).
A user revisits their build at a major release, not at every point release.

The compatibility track covers only one of the four ways users get hipfort today; the table lists all four.

| How you get hipfort today                                                           | What the transition does to you                                                                                                     |
| :---------------------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------------------------------------- |
| You build it from source                                                            | Nothing. The compatibility track keeps its layout, module names, and CUDA backend.                                                  |
| You install a package                                                               | Nothing: ROCm 10.0 ships no hipfort package, `.mod`, or archive, so there is nothing to lose. Packaging *returns* at 10.3.          |
| You use a library's own Fortran module (`rocblas_module.f90`, `rocsparse.f90`, ...) | A real break at 10.3, when the generated module supersedes it. See What changes for a library repository.                           |
| You build one code on several ROCm versions, some before 10.3 (for example 6.x)     | Nothing until your oldest system reaches 10.3: use the hipfort release tagged for each ROCm (`rocm-<version>`). See the note below. |

For the last row, a thin wrapper module can also select `use hipfort_<lib>` or `use <lib>` per system at compile time, and the packaged bindings can be generated for an older release's headers on request.

Only the third row is a break, and it is planned and coordinated with each library, not a side effect.
If a distribution still publishes its own hipfort package, that package continues through the transition, and any removal is announced together with that distributor.
The packaged track lands in ROCm 10.3: packaging returns, and the bindings split across `rocm-systems` and `rocm-libraries`, dropping the `hipfort_` prefix, collapsing each library to a single Fortran module, and dropping the CUDA backend from the package.
Keeping both is cheap, because both are generated rather than written by hand.
After 10.3 the compatibility track remains as source only, and it is retired at ROCm 11.0, leaving the packaged track as the sole deliverable.

## Migration and compatibility

Existing hipfort users face two mechanical edits, and nothing else.
Routine names, argument lists, and calling styles are unchanged, so this is find-and-replace rather than a rewrite: what breaks is the `use` line and the link target, not the call sites.

**1. Module names.** The `hipfort_` prefix is dropped and each library collapses to a single module.

| Old                                                | New           |
| :------------------------------------------------- | :------------ |
| `use hipfort` (+ `hipfort_types`, `hipfort_enums`) | `use hip`     |
| `use hipfort_rocblas` (+ `_enums`)                 | `use rocblas` |
| `use hipfort_<lib>` (+ `_enums`, `_types`)¹        | `use <lib>`   |

¹ `_types` exists only for rocRAND and hipFFTW.

The library's own enum modules fold in the same way: `use rocsparse_enums`, `use hipsparse_enums`, and `use rocblas_enums` become `use <lib>`.

The HIP memory helpers (`hipfort_hipmalloc`, `hipfort_hipmemcpy`, `hipfort_hiphostregister`) fold into `hip`, and the cross-cutting modules (`hipfort_check`, `hipfort_auxiliary`) fold into the library module the caller already `use`s: `hipCheck` comes from `hip`, `rocblasCheck` from `rocblas`, and handles stay `type(c_ptr)`.

**2. Link target.** The single `libhipfort-*.a` becomes one archive and one CMake target per library, in the C library's own namespace.
`find_package(<lib>-fortran)` finds it and also brings in the C library; once rocm-cmake has the optional include, `find_package(<lib>)` alone will do (see Building the bindings):

```cmake
# before
find_package(hipfort REQUIRED)
target_link_libraries(app PRIVATE hipfort::rocblas)

# after
find_package(rocblas-fortran REQUIRED)
target_link_libraries(app PRIVATE roc::rocblas roc::rocblas_fortran)
```

The namespace follows each C library rather than a hip-versus-roc rule: `hip::` for the HIP runtime, hipFFT, hipFFTW, and hipRAND; `roc::` for ROCTx, rocBLAS, hipBLAS, rocSPARSE, hipSPARSE, rocFFT, rocSOLVER, hipSOLVER, and rocRAND (matching the `roc::hipsolver_fortran` that standalone hipSOLVER already exports).
Without CMake, a build passes the per-compiler directories itself, for example `-I/opt/rocm/include/fortran/amdflang -L/opt/rocm/lib/fortran/amdflang -lrocblas_fortran -lrocblas -lamdhip64`.

**Both can be installed at once**, so a codebase migrates one file at a time: the module names differ, the `.mod` files land in different subpaths, the archives have different filenames, and the object symbols are mangled per module name (`__hipfort_rocblas_MOD_...` against `__rocblas_MOD_...`).
The single rule is not to `use` both bindings for the same library in one scope, since they export the same public names and the reference becomes ambiguous.
That clash is source-level and scope-local, which is exactly why a file-by-file migration works.

One capability does not carry over: the CUDA (`nvptx`) backend is not built for the packaged track (see CUDA backend), and keeps working on the compatibility track until 11.0.
`roctx` does carry over, as `use roctx` and `roc::roctx_fortran`, from `rocm-systems`' `projects/fortran/`.

The user-facing migration guides live in the hipfort documentation ([ROCm/hipfort#548](https://github.com/ROCm/hipfort/pull/548)) rather than in this RFC, so they track the packaging as it lands.
They give the full per-library table, the compile and link lines, when a user has to build a binding, and a scripted rename.
There are two, because the populations differ: one for hipfort users (a rename, with an overlap until 11.0) and one for users of a library's own module (call-site changes, at 10.3).

## Rollout and success criteria

The rollout is staged, and each stage has a clear exit.

1. **Bring current (7.14 / 10.0 / 10.1 / 10.2).** The committed bindings match the current headers.
   The CI hook is not implemented yet. It needs two small steps: making the generator repository public, so that a public CI job can run it, and adding a path-filtered workflow to each monorepo.
   The hook starts in *advisory* mode, suggesting regenerated diffs without blocking a merge.
   Until it runs, a binding is refreshed by regenerating it and committing the result by hand, which is how the co-location PRs were produced.
1. **Co-locate and package (10.3).** Each in-scope library has a `fortran/` directory, each project builds its per-library `.mod` and `lib<lib>_fortran.a` and TheRock packages them, and the `bindings-in-sync` gate becomes *enforcing*.
1. **Retire the compatibility track (11.0).** `ROCm/hipfort` under its current name is removed, and the packaged track is the sole deliverable.

The rollout is done when:

- `projects/fortran/` (`hip`, `roctx`) and the `fortran/` of the ten in-scope `rocm-libraries` projects (eleven modules) carry generated source, curated metadata where needed, and tests;
- a header PR that changes a binding fails the enforcing gate until the regenerated `.F90` is committed;
- `find_package(<lib>-fortran)` exposes `<ns>::<lib>_fortran` when built, where `<ns>` is the C library's namespace (and `find_package(<lib>)` too, once rocm-cmake has the optional include);
- TheRock builds and packages them (see Packaging in TheRock);
- an external consumer compiles and links against the installed package with at least amdflang and gfortran.

The co-location step is open as one pull request per library: the HIP binding (rocm-systems#11923), ROCTx (rocm-systems#12700), and one per `rocm-libraries` project, in draft until the HIP binding lands.
Every `rocm-libraries` binding test uses `hip`, except rocRAND's and hipRAND's version checks, so these tests need the HIP binding (rocm-systems#11923) installed.
Until it is, they configure as skipped, and the `rocm-libraries` pull requests stay in draft.
All these pull requests share one shape: the options of Building the bindings, a static `<lib>_fortran` in the C library's namespace, the `<lib>-fortran` package beside the C one with its forwarder, the install tree of Layout after the split, and the test convention of Tests and documentation.
What remains before they leave draft:

- each `rocm-libraries` `fortran/` gains its `metadata/` (only the HIP binding carries one so far; ROCTx needs none);
- the committed modules are regenerated with the generator release the CI hook will run, so the hook's first run on each library is a no-op (most committed modules predate parts of the surface described here: the `X` / `X_` generics, ROCTx's typed strings, the solvers' array overloads);
- ROCTx moves into `projects/fortran/` beside HIP;
- hipBLAS drops the temporary `hip` stub module its clients build until the HIP binding is installed;
- the few remaining differences in how projects wire `fortran/` are aligned: where the root adds it (rocSOLVER adds it inside `if(BUILD_LIBRARY)`, the others unconditionally), and whether the amdflang preference is inlined or in `fortran/cmake/ROCmFortranCompiler.cmake`.

rocBLAS's and hipBLAS's `fortran/test/` come after, as part of the rollout.

### Packaging in TheRock

TheRock already builds every project above as a subproject, so it builds their `fortran/` with them, and nothing about the bindings needs a separate pipeline.
It needs a short list of changes in TheRock itself, none of which changes the design.
Part of it is already open as TheRock#3928 (flang and OpenMP throughout ROCm), which:

- adds `FORTRAN_OPTIONAL` / `FORTRAN_REQUIRED` to `therock_cmake_subproject_declare`;
- emits `CMAKE_Fortran_COMPILER` for the subprojects that opt in;
- marks rocBLAS, hipBLAS, rocSOLVER, hipSOLVER, rocRAND and hipRAND `FORTRAN_OPTIONAL`;
- drops the hipSOLVER `.so` entries.

The bindings build on it: the remaining in-scope `rocm-libraries` subprojects (rocSPARSE, hipSPARSE, rocFFT, hipFFT) opt in the same way, and `rocm-systems`' `projects/fortran/` becomes one more subproject, Linux-only and `FORTRAN_REQUIRED`, after `hip-clr` and `rocprofiler-sdk`, whose installed HIP and ROCTx it builds against.
The items below are written against that mechanism rather than a second one.

1. **One Fortran compiler for every subproject: the `amdflang` TheRock builds.**
   `compiler/amd-llvm` already builds flang on Linux (`LLVM_ENABLE_PROJECTS` includes `flang`, and `flang-rt` is added to the runtimes), and `base/aux-overlay` exposes it as `bin/amdflang`.
   But the toolchain file TheRock generates for each subproject (`cmake/therock_subproject.cmake`) sets `CMAKE_C_COMPILER`, `CMAKE_CXX_COMPILER` and, for `amd-hip`, `CMAKE_HIP_COMPILER`, and no `CMAKE_Fortran_COMPILER`.
   Each project's probe then picks a compiler on its own.
   The `fortran/` probe looks for `amdflang` beside the C compiler first.
   But a project root that probes earlier with a plain `check_language(Fortran)` takes whatever is on `PATH`: on a TheRock builder, with the CMake 3.31 it pins, that is `gfortran` (the environment check requires it).
   Newer CMake already prefers the flang beside the C compiler.
   The fix, which TheRock#3928 makes, is to emit `CMAKE_Fortran_COMPILER` (the toolchain's flang, which `bin/amdflang` points to) in that generated file for every subproject that opts in.
   Every subproject then compiles its binding with the compiler that ships, so the shipped `.mod` files match the shipped `amdflang` exactly, version included.
   It also keeps the dependent bindings consistent.
   On `develop`, rocSOLVER and hipSOLVER default `CMAKE_Fortran_COMPILER` to `gfortran` before `project()` (hipBLAS does so in its clients project).
   The co-location pull requests narrow that default to builds of the test and benchmark clients, and prefer `amdflang` otherwise.
   But TheRock builds those clients when testing is on, so without this change it would build `rocsolver.mod` with gfortran against an `amdflang`-built `rocblas.mod` that gfortran cannot read, and the rocSOLVER binding would be skipped.
   A compiler set by the toolchain overrides that default.
   The projects keep gfortran for their clients on purpose: the clients link a host LAPACK, and a gfortran-built LAPACK needs the gfortran runtime. This does not apply in TheRock, whose host LAPACK is OpenBLAS built with `C_LAPACK=ON` (`third-party/host-blas`) and needs no Fortran runtime, so forcing `amdflang` is safe for the clients too.
1. **Artifact descriptors.**
   The default `dev` component already collects `**/*.a`, `**/include/**` and `**/cmake/**`, so the `.mod` files, the archives and the config packages land in `dev` with no change.
   Three edits remain:
   - Declare the `projects/fortran/` subproject and give it an artifact descriptor: `dev` for the `.mod` files, archives, config packages and `share/`, `test` for its tests, once they are installed.
   - Add `share/<lib>/fortran/**` to the `dev` component of each library (for HIP and ROCTx, to the `dev` component of the new `projects/fortran/` artifact), since `share/` is not a default `dev` path and the installed `.F90` is what users of other compilers build from.
     HIP's existing `share/hip/**` rule sits in the `lib` component of `core/artifact-core-hip.toml`; the Fortran files come from their own subproject, so they do not fall under it.
   - Fix the existing Fortran entries in `math-libs/BLAS`.
     hipSOLVER's `lib/libhipsolver_fortran.so*` (in `artifact-solver.toml`) goes away once rocm-libraries#12581 lands, because the binding becomes a static archive; the `-DEXPORT_FORTRAN_BINDINGS=OFF` passed to hipSOLVER retires with that option.
     hipBLAS's `lib/libhipblas_fortran.so` entries (in `artifact-blas.toml`) already match nothing, since the clients' Fortran helper is a static library that is not installed, so they are removed; `libhipblas_fortran.a` is now the binding, under `lib/fortran/<compiler>/`.
1. **Dependency order.**
   Already expressed: rocSOLVER depends on rocBLAS, and every library depends on `hip-clr`, so the Fortran build order follows the C one.
   The `rocm-libraries` binding tests `use hip`, so they need the HIP binding installed first: `projects/fortran/` builds after `hip-clr` and before the math libraries that test against it.
1. **Tests.**
   Where a library has binding tests (`fortran/test/`, under `BUILD_FORTRAN_CLIENTS`), they are CTest tests labelled `gpu`, and join each library's existing `test` component and test runner once the project installs them.
   None of the co-location pull requests installs them yet (rocRAND and hipRAND used to install their Fortran wrapper test), so they reach TheRock's test artifacts only once each project does.
1. **Fortran runtime.**
   An `amdflang`-built test executable links flang's runtime (`flang-rt`) rather than `libgfortran`; the test artifacts depend on the `amd-llvm` runtime libraries, as the HIP ones already do. A user's application gets the runtime from the compiler that links it.

Windows builds keep `BUILD_FORTRAN_BINDINGS` off (see Non-goals).
The ASan and coverage variants will pass `-DBUILD_FORTRAN_BINDINGS=OFF` and skip the `projects/fortran/` subproject: the projects themselves build the bindings under ASan and only skip their tests, so it is TheRock that keeps Fortran out of those variants.
Today TheRock compiles Fortran uninstrumented under ASan (`cmake/therock_sanitizers.cmake`, issue #1782); instrumenting it is the draft TheRock#2403.
Once these land, TheRock's nightly builds are the first place a user gets the bindings precompiled, ahead of a ROCm release; until then, a user tries a binding by building its `fortran/` from the pull request against a nightly install.

## Open questions and risks

This is a draft.
Several points are still open.

- **Cross-repo header dependencies (a risk, settled).** Some bindings depend on another library's headers (rocSOLVER on rocBLAS) or on the HIP runtime, so the modules have a compile order.
  This mirrors the existing C dependency graph and is resolved the same way: build order in TheRock, and `find_package` for isolated per-repo builds (see Building the bindings).
  The nightly full regeneration is the backstop for cross-header drift.
- **Assumed-rank interfaces (F2018): decided, with a residual risk.** The default is not assumed-rank.
  The default stays the explicit, per-rank variants, which work on older compilers, with a build option (`-DFORTRAN_ARRAY_INTERFACES=assumed-rank`) to select assumed-rank instead.
  One tri-state option rather than two booleans, because the two tiers are exclusive and two booleans could express the illegal combination.
  Enabling it *replaces* the per-rank variants rather than adding to them: an assumed-rank dummy is not distinguishable by rank from the per-rank specifics, so the two cannot legally coexist in one generic.
  F2018 assumed-rank is not uniformly mature across gfortran, flang, ifx, and Cray CCE, and HPC sites lag the standard; it compiles with amdflang today, but corner-cases are a risk.
  Turn it on by default later, once the compilers are proven.
  Assumed-rank would also shrink the generated sources and their build time (for example the ~24k-line `hipfort_hipmalloc` on today's hipfort).
- **Minimum compiler versions.** The minimum standard is settled at Fortran 2003 (see Language standard; for Fortran 77 code, see Fortran 77 code); the minimum compiler versions are not.
  The question worth asking the large HPC sites is which compiler versions they pin.
- **Where the generator ultimately lives.** It stays a standalone repository (P3), and it becomes public before the CI hook is enabled; that much this RFC settles.
  What is not settled is whether it eventually moves next to the Fortran toolchain that consumes its output.
- **Size of the generated files.** A library's binding is one generated file, and for the large APIs it is very large (about 46k lines for `rocsparse.F90`, 43k for `hipsparse.F90`, 58k for `rocblas.F90`), too large for GitHub to render in a diff and a concern raised in review.
  Two options:
  - split the output per C header: one `.F90` per header, still assembled into one module (Fortran `include` or submodules), which keeps diffs reviewable at the cost of more files;
  - keep one file, treated like any generated artifact: marked `linguist-generated` and reviewed through the generator and the metadata rather than line by line.
    The CI hook makes the second option safer, since a regenerated diff is then produced by the tool rather than by hand.
- **Docstrings and the documentation site (Sphinx + Doxygen).** The layout is decided (see Tests and documentation): `fortran/` has no `docs/` subfolder, and the Fortran API is documented from the library's existing top-level `docs/` (for HIP and ROCTx, HIP's and rocprofiler-sdk's).
  Still open:
  - whether the packaged modules carry the headers' docstrings (this would add a Doxygen pass to the in-sync gate, since hipfort's docs build runs with `WARN_AS_ERROR`);
  - if they do, how to assemble one coherent site from twelve doc sets;
  - how to wire cross-project references (a rocSOLVER docstring that references rocBLAS types needs rocBLAS's Doxygen tag file or Sphinx inventory at build time).
    Two shapes: each library builds its Fortran doc section, aggregated into the ROCm doc portal; or the doc job assembles a single combined site from all sources.
    This needs the ROCm documentation team.
- **A second error model (owned error objects).** rocSPARSE now ships, alongside the status code, an owned error object: several entry points take a `rocsparse_error*` out-parameter producing an opaque descriptor the caller must release with `rocsparse_destroy_error`, whose message is a borrowed `const char*` tied to the descriptor's lifetime.
  The status-code model the bindings inherit from hipfort cannot express an object the caller must destroy, or a string that stays valid only while that object lives.
  Today the generated `rocsparse.F90` binds it at the raw level: the descriptor is a `type(c_ptr)` passed as an optional out-argument, the caller releases it explicitly with `rocsparse_destroy_error`, and the message comes back as a raw `c_ptr`.
  Open: whether to add an idiomatic layer on top (a derived type with a finalizer, the message copied into a Fortran string), and how it states the lifetime of the borrowed message pointer.
- **Deprecation and removal policy.** A C `deprecated` attribute is a compiler diagnostic, not a docstring, so a Fortran caller of a deprecated entry point gets no warning where a C caller does.
  And because the generated surface tracks the header exactly, removing a C function would remove the Fortran routine in the same merge, with no release in which Fortran users were warned.
  Proposed, not yet agreed: capture the attribute as a generated comment marker (a docstring, if docstrings are turned on; Fortran has no deprecation attribute), and forbid an entry point from leaving the generated surface in the same release it leaves the C header (the in-sync gate fails a regeneration that drops a public Fortran name unless the PR acknowledges it).
- **Are dummy-argument names public API?** A Fortran caller can pass an argument by name (`call rocblas_dscal(n=n, ...)`), which makes every argument name part of the module's public contract.
  Those names come from the C parameter names, which are not API in C: renaming a C parameter is invisible to C callers, but it breaks a Fortran caller that passes that argument by name.
  Open: decide whether dummy names are supported API; if so, the in-sync gate fails on a dummy-name rename rather than suggesting it, and unnamed C parameters get a deterministic naming rule so they do not drift between generator versions.

## Alternatives considered

**A single bindings super-repo.** All bindings in one dedicated repository (today's hipfort, with generated rather than hand-written content) keeps a clean boundary and simple ownership, and it stays a reasonable option.
Its tradeoff is that the bindings sit downstream of the header change, so they update in a separate, later PR, and headers must be copied out to regenerate.
Co-location gives up that clean separation for updates that move in lockstep with the C API, which is the property this RFC prioritizes.

## Appendix: How NVIDIA does it (baseline)

The closest analog is NVIDIA's Fortran story for its GPU math libraries, and the contrast is instructive.

NVIDIA ships precompiled Fortran modules for cuBLAS, cuFFT, cuRAND, cuSOLVER, cuSPARSE, cuTENSOR, and more, inside the HPC SDK, next to `nvfortran`.
You write `use cublas` and link with `-cudalib=cublas`.
The decisive difference is that NVIDIA owns the only compiler its users use.
With a single supported compiler the "`.mod` incompatible across compilers" problem simply does not exist for them, so shipping prebuilt `.mod` is fine.
We cannot copy that: our users are on gfortran, amdflang, Cray CCE, ifx and others, so we ship the source for every compiler, plus a precompiled set for `amdflang`.

A few more contrasts, point by point:

- **Location.** NVIDIA's interfaces live in the compiler/SDK, not co-located in each library's source repo.
  Ours live next to the C headers in `rocm-systems` and `rocm-libraries`, and change in the same PR as the API.
- **Naming.** No prefix: `use cublas`, not `nvfort_cublas`.
  The module name is the library name.
- **Backward compatibility.** They keep the legacy and the v2 calling conventions side by side (`cublas` and `cublas_v2`).
- **Array handling.** They do not use assumed-rank.
  They use the CUDA Fortran `device` attribute: `nvfortran` distinguishes host from device arrays, and the interfaces declare device arrays, so host/device dispatch is resolved by the language.
  gfortran and flang do not know `device`, so our `bind(C)` plus `c_loc` plus generated variants is the portable substitute for what `nvfortran` does natively.
- **Scalar host-vs-device (alpha/beta).** Because `nvfortran` knows host from device, its non-v2 modules set and restore the library pointer mode implicitly, at a small overhead.
  We cannot rely on the compiler for this, so `.argkinds` records which scalar pointers are host values.
  One caveat static metadata cannot remove: the host-versus-device choice for `alpha`/`beta` is also *runtime* handle state (`rocblas_set_pointer_mode`), so the ergonomic host-scalar form is only correct when the handle is in host-pointer mode.
  The wrappers therefore must not silently set or restore that mode (it is shared, mutable handle state, so touching it would race other threads), and the raw `type(c_ptr)` surface stays the escape hatch for device-pointer mode.
- **Tooling.** Their tooling is the compiler driver: `-cudalib` plus an automatic module search path.
  There is no separate installation checker because the bindings are bundled with the compiler.
  Ours are not bundled with a single compiler, which is why explicit packaging and a presence check are worth having.

The through-line: most of the hard questions here (cross-compiler `.mod`, ABI across compilers, where the bindings live, generation) exist for us precisely because we do not own the one compiler our users run.
NVIDIA answers them by monoculture; we answer them by generation, source shipping, and co-location.

## Appendix: Fortran `.mod` and `.a` incompatibility across compilers and versions

This is the constraint that shapes the whole packaging design, so it is set out here in full.

Unlike C, Fortran has no standard ABI and no standard module format.
Two artifacts are affected.

- **The `.mod` file.** A `.mod` is a compiler-private serialization of a module's interface (names, kinds, generic-resolution tables).
  Its format is undocumented and specific to the compiler, and it is **not stable across versions of the same compiler**: gfortran, for instance, bumps its module-format version between releases, so a `.mod` written by one gfortran is rejected by another.
  There is no interoperability across compilers either: a gfortran `.mod` is meaningless to ifx, amdflang, or Cray, and vice versa.
- **The compiled `.a`.** The wrapper bodies compile to objects whose symbols are **name-mangled** by a compiler-specific scheme (trailing underscores, module-name decoration such as `__rocblas_MOD_`), so the archive is tied to the same compiler, and version, as the `.mod` it accompanies.

So there is exactly one portable thing to ship: **the Fortran source**.
A prebuilt `.mod`/`.a` set is usable only by the exact compiler, at essentially the exact version, that produced it.

One relaxation exists on the compiler we ship.
amdflang has an opt-in `-fmodule-mismatch-check=warn` (since ROCm 10.1) that downgrades the module-version check to a warning, so a newer amdflang can read a `.mod` written by an older one.
That softens the *version* axis for the precompiled amdflang set, which is the set we ship and the skew our users are most likely to meet (a ROCm-shipped `.mod` against a slightly different amdflang).
It does nothing for the *compiler* axis, which is the one that forces us to ship source: no flag makes a gfortran `.mod` meaningful to amdflang.

This is why the design:

- ships **source** so any site can compile with its own compiler, and additionally ships a **precompiled set for the default compiler (`amdflang`)** for the common case;
- keys the install tree by **compiler** under `lib/fortran/<compiler>/` and `include/fortran/<compiler>/` (a site keeping two versions of one compiler sets `<LIB>_FORTRAN_COMPILER_DIR`, for example `gfortran-13`);
- treats the only stable ABI as the **C one**, the vendor `.so`, which the `bind(C)` layer rides on and which every compiler agrees about.

NVIDIA escapes all of this by owning the single compiler (`nvfortran`) its users run, which is why it can ship prebuilt `.mod` and we cannot (see How NVIDIA does it).

## Appendix: Implementation backlog and open risks

The direction is set; the items below are implementation and CI work that does not change the design, recorded here so they are not lost. None is a blocker.

**CI and reproducibility.**

- Pin what the generator resolves (its revision, the Julia and Clang.jl versions, the include roots, the macro profile, and the C standard level it parses at), so a regeneration is reproducible, and a toolchain change reaches `hip`, `roctx` and the library modules only when someone moves the pin.
- Some public headers do not parse from a source checkout alone. The generator already synthesizes the version, config and export headers from their templates, but feature-gate headers (`*.h.in`) can still change the visible API without touching a tracked `.h`. Define the generator's input as a named, configured artifact per library, or run the hook nightly for the libraries whose headers need a configured or partly-built tree.
- The per-PR hook reads PR-head content, runs a parser over it, and needs a write-capable token to post the suggestion. Use the standard two-workflow split: an unprivileged job on the PR head produces the diff as an artifact, and a trusted job on the base ref posts it and never executes PR code.

**Release integrity.**

- Resolve every `bind(C, name=)` label in the generated sources against the symbol tables of the libraries actually shipped in a release, and fail on any unresolved name, so a feature-gated API cannot ship as a Fortran interface with no code behind it. Linking the built archive with `--whole-archive -Wl,--no-undefined` does the same for the wrapper bodies in the `.a`.
- Add the size-and-alignment gate: a CI check comparing each emitted struct's size and alignment against the C compiler's `sizeof`/`_Alignof` on the target, since those ABI facts are frozen into a committed artifact. This also covers the complex-number mapping, which maps a two-float C struct to a Fortran `complex` on an ABI assumption.

**Type mapping.**

- Integer scalars map to named `iso_c_binding` kinds, not host-measured ones. Document two assumptions: a site that builds with `-i8` (8-byte default `INTEGER`) must give integer literals an explicit kind (`1_c_int`), and the `iso_c_binding` kinds describe the C compiler paired with the Fortran compiler, which need not be the compiler that built the `.so`.
- Function-pointer and callback types map to `c_funptr` with an emitted `abstract interface` where the signature is mappable; a callback that must be a device function pointer is marked unsupported-from-Fortran in a generated comment rather than emitted as if usable.
- Compatibility track only (the packaged track has no NVIDIA backend): a few handle types differ across platforms (for example `hipfftHandle`, a 32-bit int on NVIDIA but a pointer on AMD). Either record a per-backend type substitution, or drop that library from the nvptx build.

**Cross-library and metadata consistency.**

- The opaque-handle list, the "own headers" marker, and the name prefix are global decisions, not per-library ones (rocSOLVER's whole API takes `rocblas_handle`). Promote them to one repository-level config, and gate on two modules claiming the same declaration, or a module referencing a type no module claims.
- Assert the 63-character Fortran identifier limit in the generator (the longest C symbol is 52 characters), and fail rather than truncate.

**Packaging and build variants.**

- The ASAN build variant ships no Fortran, since Fortran linking is unsupported under ROCm's ASAN configuration; the presence check reports that distinctly rather than as "not installed for your compiler". Same for the coverage variant.

**Consumer experience and policy.**

- Offer a non-CMake consumer path beyond the documented command lines (a `pkg-config` file, or a small flags script), since not every HPC site builds with CMake. The migration guide already gives one worked command line per compiler, and the installed `.F90` can be compiled by hand with `-DUSE_ASSUMED_SHAPE=1` (the preprocessor macro behind `FORTRAN_ARRAY_INTERFACES=assumed-shape`).
- Decide whether to ship a supported migration helper that rewrites `use hipfort_*` lines to the packaged module names. The migration guide documents the `sed` recipe such a tool would automate (delete the folded modules, rename the rest, leave `cuda_errors` alone), so the open question is support surface and the residue the recipe leaves to the user (fixed-form sources, continuation lines, `use hipfort_check` in a file that never used `hipfort`), not effort.
- Share the Fortran probe and guard across projects (one helper in rocm-cmake, or a `shared/cmake` module in each monorepo) instead of the copy each `fortran/CMakeLists.txt` carries today, and settle one name for the root's "a Fortran compiler exists" flag (`ROCM_HAVE_FORTRAN` in `rocm-systems`, `ROCM_LIBS_HAVE_FORTRAN` in `rocm-libraries`).
- Add the `include(<lib>-fortran-config.cmake OPTIONAL)` hook to rocm-cmake's generated config, so that `find_package(<lib>)` alone exposes the Fortran target.
- State a default-inclusion policy and who answers support tickets for a public Fortran API shipped under a library's name (bug reports arrive against rocBLAS, while the bindings team owns `fortran/`).

## Appendix: Testing the generator

The generator is tested at three levels: its own logic, the shape of its output, and the output at runtime.

Its logic has a unit-test suite in the generator repository.
It exercises the pure transforms, with no ROCm headers: C-type to Fortran mapping, name and spec helpers, the metadata loaders, the bind, wrapper, enum, and struct emitters, 132-column line wrapping, and the selection of header sources from checkouts; small fixture headers are parsed with the libclang Clang.jl ships.
The suite also includes:

- a byte-for-byte golden test, which asserts the emitters still produce the same output as before;
- `compile_golden.jl`, which runs `gfortran -fsyntax-only` on regenerated fixtures to prove the output is valid Fortran;
- `array_ifaces.jl`, which checks that each `FORTRAN_ARRAY_INTERFACES` setting yields the same API as the guarded module compiled under its macro.

The suite runs in the generator repository's CI on every supported Julia version.

The output is also gated in CI, and those gates double as regression tests.
The committed modules must compile under amdflang and gfortran, in each array configuration.
Each library's Fortran tests (`fortran/test/`, where present) must still compile against the regenerated interfaces, which catches interface drift without a GPU.

These gates are compile-only, by design, so they need no GPU.
The bindings themselves are exercised at runtime by those tests on an AMD GPU, as GPU-labelled CTest tests of each library.
That runtime suite must cover both pointer modes (host and device) for the `argkinds`-classified entry points: a misclassified scalar pointer compiles and links cleanly, so it is the one correctness property no compile-only gate can see (see Keeping the classifications correct).

Because the transforms are unit-tested, the trust the enforcing `bindings-in-sync` gate needs is largely in place: a C developer who accepts a suggested `.F90` relies on a tool whose logic is checked, not only on its output.
