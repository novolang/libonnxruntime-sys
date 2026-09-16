# libonnxruntime-sys

ONNX Runtime is an inference engine for machine learning models in the
ONNX format. It loads a model, chooses an execution provider for it —
the CPU, a CUDA card, a vendor accelerator — and runs it. Its C
interface is documented in the
[ONNX Runtime C API reference](https://onnxruntime.ai/docs/api/c/).
This package declares **two** entry points to novo-lang, because two is
how many plain C symbols the shared library exports.

**Status: a binding, not a port, and a very small one.** The ONNX
Runtime C API is not a set of symbols. It is a table of function
pointers, reached through one symbol, and a novo-lang `@ffi`
declaration resolves a name at link time and cannot call an address. So
`CreateEnv`, `CreateSession`, `CreateTensorWithDataAsOrtValue`, `Run`,
`ReleaseSession` and every other call in the interface are absent, not
because of an omission but because there is no name to declare. This
package binds what can be bound and states the rest plainly. The
section "What is not included" and the section "What a program that
wants inference has to do" say what that costs and what closes it.

## What it is

An **ONNX model** is a computation graph in a portable format, usually
a `.onnx` file. It is what a training framework exports.

An **execution provider** is the code that runs a graph on one kind of
hardware. The CPU provider is always present; a CUDA or a ROCm provider
is in a build made for it.

The **function table**, `OrtApi`, is a C structure whose every field is
a function pointer. There is one instance of it per API version, the
library owns it, and every call in the interface is reached through it.
The C header defines `OrtApi` and the C caller writes
`g_ort->CreateEnv(...)`, which is a load from a field followed by an
indirect call.

The **API base**, `OrtApiBase`, is the structure that hands the table
over. It has two fields: `GetApi`, which takes a version number and
answers the address of an `OrtApi` for it, and `GetVersionString`,
which answers the library's version as text.

`OrtGetApiBase` is the **one symbol** in the whole interface. It answers
the address of the `OrtApiBase`. Everything else the C caller does
begins there.

## Install

```
novo pkg add libonnxruntime-sys
```

Adding the package does not install the C library. ONNX Runtime
publishes the runtime as a tarball holding `include/` and
`lib/libonnxruntime.so`, on the
[releases page](https://github.com/microsoft/onnxruntime/releases). The
tarball is unpacked wherever the reader wants it, and the linker is told
where that is:

```
export LIBRARY_PATH=/path/to/onnxruntime/lib
export LD_LIBRARY_PATH=/path/to/onnxruntime/lib
```

ONNX Runtime installs no pkg-config file, so there is no
`libonnxruntime.pc` to ask and the link flag `-lonnxruntime` is written
into `novo.toml` instead.

## Example

What this package can do, and where it stops:

```novo ignore
use libonnxruntime

fn main() [io, ffi]
    // The one symbol.  It never fails.
    let base = libonnxruntime.ort_get_api_base()

    // Two eight-byte function pointers: `GetApi` and
    // `GetVersionString`.
    let get_api = ptr.read_word(base)
    let get_version_string = ptr.read_word(base + 8)
    println("the runtime loaded: GetApi is at ${get_api}")

    // And that is as far as a `sys` binding reaches.  Both words are
    // addresses of C functions, and there is no novo-lang expression
    // that calls an address.  Asking the version string, getting the
    // `OrtApi` table, creating an environment, loading a model and
    // running it all live behind these two pointers.
    println("GetVersionString is at ${get_version_string}, and cannot be called")
```

The example is fenced as an illustration rather than a compiled block
because `novo doc` compiles the blocks in documentation comments and not
the ones in this file. The same calls are in
`tests/libonnxruntime_tests.nv`.

## What the package contains

| Module | Contents |
| --- | --- |
| `libonnxruntime` | Both exported symbols: `OrtGetApiBase`, which answers the address of the structure that hands out the function table, and `OrtSessionOptionsAppendExecutionProvider_CPU`, the CPU provider factory. |

| Group | Entry points | What it does |
| --- | --- | --- |
| The API base | 1 | Answers the address of the two function pointers the interface begins at. |
| The CPU provider factory | 1 | Adds the CPU execution provider to a set of session options. |

## How to choose an entry point

`ort_get_api_base` is the one to call, and the only one a program can
call usefully. It reports that the shared library loaded and resolved,
and it gives the two addresses a C helper would go on to use.

`ort_session_options_append_execution_provider_cpu` is declared because
it is a symbol. It takes an `OrtSessionOptions`, which is made by
`CreateSessionOptions`, which is a field of the table, so nothing in
this package can produce an argument for it. A program reaches it after
it has a helper, and by then the helper can call it directly.

## The rules a user needs

1. **A pointer is an `Int`, and zero is null.**
2. **`OrtGetApiBase` never fails.** The structure belongs to the
   library and lives as long as it is loaded. There is nothing to
   release.
3. **The `OrtApiBase` structure is two eight-byte function pointers.**

   | Offset | Field | Signature in C |
   | --- | --- | --- |
   | 0 | `GetApi` | `const OrtApi *(uint32_t version)` |
   | 8 | `GetVersionString` | `const char *(void)` |

   Read them with `ptr.read_word`. Both are non-zero in a loaded
   library, and neither can be called from novo-lang.
4. **`GetApi` takes a version number and may answer 0.** The number is
   `ORT_API_VERSION`, a macro in `onnxruntime_c_api.h` whose value is
   the runtime's minor version: 17 for ONNX Runtime 1.17. A library
   older than the version asked for answers the null pointer rather
   than a table. A macro is not a symbol, so a program that needs the
   number reads it out of the header it compiled against.
5. **A call that answers an `OrtStatus *` answers 0 for success.** A
   non-zero answer is the address of a status object carrying a code and
   a message, and it is released through `ReleaseStatus`, which is a
   field of the table.
6. **The provider factories other than the CPU one are not in every
   build.** `OrtSessionOptionsAppendExecutionProvider_CUDA`,
   `_ROCM`, `_Tensorrt`, `_OpenVINO`, `_Nnapi` and `_CoreML` are
   symbols only in a library built with that provider, so declaring one
   would turn a missing provider into a link failure. They are not
   declared here.

## What a program that wants inference has to do

The interface is reachable from C and not from a linked symbol, so the
missing piece is C. A small helper, compiled against
`onnxruntime_c_api.h` and linked beside the runtime, does this:

1. calls `OrtGetApiBase()->GetApi(ORT_API_VERSION)` once and keeps the
   table in a static;
2. exports one plain C function per call a program needs —
   `nv_ort_create_env`, `nv_ort_create_session`, `nv_ort_run` and so on
   — each of which loads the field and calls through it;
3. flattens the arguments that are structures into integers, floats and
   addresses.

Those exported functions are names, so they are `@ffi` declarations in
the ordinary way. That helper is a package of its own and not this one:
a `sys` package binds a C library and carries no C source. Until it
exists, this package is a report that the runtime loaded.

## What is not included

- **The whole `OrtApi` function table, which is the whole interface.**
  Every call ONNX Runtime offers — `CreateEnv`, `CreateSessionOptions`,
  `SetIntraOpNumThreads`, `CreateSession`, `SessionGetInputCount`,
  `SessionGetInputName`, `CreateCpuMemoryInfo`,
  `CreateTensorWithDataAsOrtValue`, `GetTensorMutableData`, `Run`,
  `GetErrorMessage`, `ReleaseStatus`, `ReleaseValue`, `ReleaseSession`,
  `ReleaseEnv` and the roughly three hundred others — is a **field of a
  structure** rather than an exported symbol. A novo-lang `@ffi`
  declaration names a symbol for the linker to resolve. There is no
  symbol to name, and this package does not declare one it cannot
  resolve.
- **`GetApi` and `GetVersionString`.** They are the two fields of
  `OrtApiBase`. Their addresses are readable and neither is callable.
- **The execution provider factories other than the CPU one.** They are
  present only in a build carrying that provider.
- **The training interface**, `OrtTrainingApi`, which is reached through
  `GetTrainingApi`, a field of the table.
- **The C++ interface.** `onnxruntime_cxx_api.h` is a header-only
  wrapper over the same table, and a header is not a shared library.

## Related packages

`ml` is the novo-lang orbit that would load a model and run it. It is
the reason this row is on the shelf, and it is what would consume the C
helper described above rather than this package directly.

`gguf-nv` and `tokenizers-nv` are the neighbours for a program that runs
a language model from a file rather than an ONNX graph. `libcuda-sys`
and `libhip-sys` are the bindings for the device memory an inference
program moves data through, and unlike this package they bind runtimes
that do export their calls as symbols.

## Tests

`tests/libonnxruntime_tests.nv` holds two tests written against the
signatures. They call the C library, so `novo test` needs ONNX Runtime
installed and linkable:

```
novo test tests/libonnxruntime_tests.nv
```

`novo pkg build` type-checks the declarations and needs nothing
installed.

**Unverified: the suite has never linked on the staging machine.**
ONNX Runtime was not installed where this package was written, so `novo
test` stopped at `cannot find -lonnxruntime` and no assertion below has
ever been observed to hold. Both are written from the documented C API.
Treat the package as unmeasured until someone runs it against a real
libonnxruntime.

The suite loads no model, reads and writes no file, and needs no
privileges. The first test asserts that `OrtGetApiBase` answers a
non-zero address, that the two words in the structure are two different
non-zero function pointers, and — in its comments — that this is where
a `sys` binding stops. The second test names the CPU provider factory
in a branch that is never taken, because an `OrtSessionOptions` cannot
be made from here; the declaration is compiled and linked all the same.

## Implementation status

| Group | State |
| --- | --- |
| The API base | Complete. It is one symbol and it is here. |
| The CPU provider factory | Complete as a declaration. Not usable without the helper. |
| The `OrtApi` function table | Absent. Every call in it is a struct field, not a symbol. |
| The training interface | Absent, for the same reason. |
| The other execution providers | Absent. Not every build exports them. |

## Licence

Apache-2.0. See [LICENSE](LICENSE).

ONNX Runtime itself is distributed under the MIT licence, and installing
it is the reader's own step.
