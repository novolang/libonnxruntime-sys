# Changelog

All notable changes to libonnxruntime-sys are recorded here. The format
is [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.1 — 2026-09-24

The documentation and comments in plain prose; no declaration changed.

## 0.1.0 — 2026-09-16

The first release: two entry points, one `@ffi` declaration each, and
no logic. Two is how many plain C symbols the ONNX Runtime shared
library exports.

### Added

- `libonnxruntime` — both symbols.
  - `ort_get_api_base`, over `OrtGetApiBase`. It answers the address of
    the `OrtApiBase` structure, which holds two eight-byte function
    pointers: `GetApi` at offset 0 and `GetVersionString` at offset 8.
  - `ort_session_options_append_execution_provider_cpu`, over
    `OrtSessionOptionsAppendExecutionProvider_CPU`, the one exported
    symbol of the CPU provider factory.
- `tests/libonnxruntime_tests.nv` — two tests over the signatures. They
  load no model, read and write no file, and need no privileges.

### Why there are two, which is this package's whole finding

The ONNX Runtime C API is not a set of symbols. It is a **table of
function pointers**. A C caller writes
`OrtGetApiBase()->GetApi(ORT_API_VERSION)`, which answers the address of
an `OrtApi` structure, and then reaches every call in the interface —
`CreateEnv`, `CreateSession`, `CreateTensorWithDataAsOrtValue`, `Run`,
`ReleaseSession`, roughly three hundred of them — as a **field of that
structure**, loaded and called indirectly.

A novo-lang `@ffi` declaration names a symbol for the linker to resolve.
There is no symbol behind any of those calls, and there is no novo-lang
expression that calls an address. So the honest surface is:

- `OrtGetApiBase`, which is a symbol and is here;
- `OrtSessionOptionsAppendExecutionProvider_CPU`, which is a plain C
  function in the CPU provider factory, is a symbol, and is here —
  although nothing in this package can produce the `OrtSessionOptions`
  it takes, because that comes from a field of the table;
- and nothing else.

This package declares no entry point for a table field. A declaration
whose symbol the loader cannot resolve is a link failure dressed as an
interface, and a reader who saw `ort_create_session` in the API page
would have been told something untrue about what the package binds.

The missing piece is C, not novo-lang. A helper compiled against
`onnxruntime_c_api.h` that resolves the table once and exports one
plain function per call closes the whole gap, and those exported
functions are names an `@ffi` declaration can bind in the ordinary way.
Such a helper carries C source, so it is a package of its own and not a
`sys` binding. The README describes it.

### The effect rows

`ort_get_api_base` is `[ffi]`: it answers the address of a structure the
library already holds and touches nothing else.
`ort_session_options_append_execution_provider_cpu` is `[io, ffi]`: it
configures a runtime object and allocates.

### Not a `0.0.x` interface release

An interface release is the shape whose every `pub fn` body is a
`todo()`. Every `pub fn` here is an `@ffi` declaration with no body, so
`novo pkg publish` reads the package as a release with bodies and
refuses a `0.0.x` version for it. The first release of a bindings
package is therefore `0.1.0`.

### Unverified

ONNX Runtime was not installed on the machine this package was written
on. `novo pkg build` type-checks the declarations without it and is
green; `novo test` stopped at `cannot find -lonnxruntime`, so the suite
has never been linked and neither assertion in it has ever been observed
to hold. In particular, that `OrtSessionOptionsAppendExecutionProvider_CPU`
resolves in a default CPU build is read from the header
`cpu_provider_factory.h` and has not been checked against a real
library. The declarations were checked against the ONNX Runtime C API
reference. Treat the whole package as unmeasured until someone runs it
against a real libonnxruntime.

### Named as missing

**The whole `OrtApi` function table.** Every call ONNX Runtime offers is
a field of a structure rather than an exported symbol. See above.

**`GetApi` and `GetVersionString`.** They are the two fields of
`OrtApiBase`. A caller can read their addresses with `ptr.read_word` and
cannot call either.

**The execution provider factories other than the CPU one.**
`OrtSessionOptionsAppendExecutionProvider_CUDA`, `_ROCM`, `_Tensorrt`,
`_OpenVINO`, `_Nnapi` and `_CoreML` are symbols only in a library built
with that provider. Declaring one would turn a missing provider into a
link failure for every user of the package.

**The training interface.** `OrtTrainingApi` is reached through
`GetTrainingApi`, a field of the table.

**The C++ interface.** `onnxruntime_cxx_api.h` is a header-only wrapper
over the same table. A header is not a shared library and has no symbols
to bind.
