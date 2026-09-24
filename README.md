# libonnxruntime-sys

ONNX Runtime is a cross-platform inference engine for machine learning
models in the ONNX format. It loads a model, runs it on an execution
provider such as the CPU, a CUDA device or a vendor accelerator, and
answers the outputs. Its C API is documented in the
[ONNX Runtime C API reference](https://onnxruntime.ai/docs/api/c/).
This package declares two entry points to novo-lang, because a CPU
build of the shared library exports two plain C symbols.

Every function here is a declaration of a function in libonnxruntime.
The ONNX Runtime C API is a table of function pointers, reached through
one symbol. A novo-lang `@ffi` declaration resolves a name at link time
and cannot call an address. `CreateEnv`, `CreateSession`,
`CreateTensorWithDataAsOrtValue`, `Run`, `ReleaseSession` and every
other call in the C API are therefore absent, because there is no name
to declare. The section "What is not included" and the section "What a
program that runs a model has to do" say what that costs and what
closes it.

## What it is

An **ONNX model** is a computation graph in a portable format, usually
a `.onnx` file. It is what a training framework exports.

An **execution provider** is the code that runs a graph on one kind of
hardware. The CPU provider is always present. A CUDA or a ROCm provider
is present only in a build made with it.

The **function table**, `OrtApi`, is a C structure whose every field is
a function pointer. There is one instance of it per API version, the
library owns it, and every call in the interface is reached through it.
The C header defines `OrtApi` and the C caller writes
`g_ort->CreateEnv(...)`, which is a load from a field followed by an
indirect call.

The **API base**, `OrtApiBase`, is the structure that hands the table
over. It has two fields. `GetApi` takes a version number and answers
the address of an `OrtApi` for it. `GetVersionString` answers the
library's version as text.

`OrtGetApiBase` is the entry point of the C API. It answers the address
of the `OrtApiBase`, and everything else a C caller does begins
there.

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
    // The entry point of the C API.
    let base = libonnxruntime.ort_get_api_base()

    // Two eight-byte function pointers, `GetApi` and
    // `GetVersionString`.
    let get_api = ptr.read_word(base)
    let get_version_string = ptr.read_word(base + 8)
    println("the runtime loaded: GetApi is at ${get_api}")

    // Both words are addresses of C functions, and no novo-lang
    // expression calls an address.  The version string, the `OrtApi`
    // table, the environment, the model and the run are all behind
    // these two pointers.
    println("GetVersionString is at ${get_version_string}, and cannot be called")
```

The example is not compiled, because it links against libonnxruntime
and the link fails where that library is not installed. The same calls are in
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

`ort_get_api_base` is the one a program can call usefully. It shows
that the shared library loaded and resolved, and it gives the two
addresses a C helper would go on to use.

`ort_session_options_append_execution_provider_cpu` is declared because
it is a symbol. It takes an `OrtSessionOptions`, which is made by
`CreateSessionOptions`, which is a field of the table, so nothing in
this package can produce an argument for it. A program reaches it after
it has a helper, and by then the helper can call it directly.

## The rules a user needs

1. **A pointer is an `Int`, and zero is null.**
2. **`OrtGetApiBase` answers a structure the library owns.** The
   structure lives as long as the library is loaded, and there is
   nothing to release.
3. **The `OrtApiBase` structure is two eight-byte function pointers.**

   | Offset | Field | Signature in C |
   | --- | --- | --- |
   | 0 | `GetApi` | `const OrtApi *(uint32_t version)` |
   | 8 | `GetVersionString` | `const char *(void)` |

   Read them with `ptr.read_word`. Both are non-zero in a loaded
   library, and neither can be called from novo-lang.
4. **`GetApi` takes a version number and may answer 0.** The number is
   `ORT_API_VERSION`, a macro in `onnxruntime_c_api.h` whose value is
   the runtime's minor version, 17 for ONNX Runtime 1.17. A runtime
   that does not support the version asked for, such as one older than
   the header, answers the null pointer rather than a table. A macro is not a symbol, so a program that needs the
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

## What a program that runs a model has to do

The interface is reachable from C and not from a linked symbol, so the
missing piece is C. A small helper, compiled against
`onnxruntime_c_api.h` and linked beside the runtime, does three things.

1. It calls `OrtGetApiBase()->GetApi(ORT_API_VERSION)` once and keeps
   the table in a static variable.
2. It exports one plain C function per call a program needs, such as
   `nv_ort_create_env`, `nv_ort_create_session` and `nv_ort_run`. Each
   loads the field and calls through it.
3. It flattens the arguments that are structures into integers, floats
   and addresses.

Those exported functions are names, so they are `@ffi` declarations in
the ordinary way. The helper carries C source, so it belongs in a
package of its own, and a binding package such as this one carries
none. No such package is published. Until one is, this package shows
that the runtime loaded and goes no further.

## What is not included

- **The whole `OrtApi` function table, which is the whole C API.**
  Every call ONNX Runtime offers is a field of a structure rather than
  an exported symbol. `CreateEnv`, `CreateSessionOptions`,
  `SetIntraOpNumThreads`, `CreateSession`, `SessionGetInputCount`,
  `SessionGetInputName`, `CreateCpuMemoryInfo`,
  `CreateTensorWithDataAsOrtValue`, `GetTensorMutableData`, `Run`,
  `GetErrorMessage`, `ReleaseStatus`, `ReleaseValue`, `ReleaseSession`
  and `ReleaseEnv` are among them, with roughly three hundred others. A
  novo-lang `@ffi` declaration names a symbol for the linker to
  resolve. There is no symbol to name, and this package does not
  declare one it cannot resolve.
- **`GetApi` and `GetVersionString`.** They are the two fields of
  `OrtApiBase`. Their addresses are readable and neither is callable.
- **The execution provider factories other than the CPU one.** They are
  present only in a build carrying that provider.
- **The training interface**, `OrtTrainingApi`, which is reached through
  `GetTrainingApi`, a field of the table.
- **The C++ interface.** `onnxruntime_cxx_api.h` is a header-only
  wrapper over the same table, and a header is not a shared library.

## Related packages

[onnx-nv](https://novo-lang.org/packages/onnx-nv) reads and writes the
ONNX model format as typed values in novo-lang, and it runs nothing. It
is published as an interface release. Every function in it is declared
and none has a body yet.

[gguf-nv](https://novo-lang.org/packages/gguf-nv) and
[tokenizers-nv](https://novo-lang.org/packages/tokenizers-nv) read a
GGUF model file and turn text into token identifiers, for a program
that runs a language model from a file rather than an ONNX graph. Both
are published as interface releases.

[libcuda-sys](https://novo-lang.org/packages/libcuda-sys) and
[libhip-sys](https://novo-lang.org/packages/libhip-sys) bind the device
memory calls of the CUDA and HIP runtimes. Those runtimes export their
calls as symbols, so their functions are declared directly.

## Tests

`tests/libonnxruntime_tests.nv` holds two tests, one for each entry
point. They call the C library, so `novo test` needs ONNX Runtime
installed and linkable:

```
novo test tests/libonnxruntime_tests.nv
```

`novo pkg build` type-checks the declarations and needs nothing
installed.

The suite has never been linked. ONNX Runtime was not installed on the
machines where this package was written and revised, so `novo test`
stopped at `cannot find -lonnxruntime`, and no assertion below has been
observed to hold. Both tests are written from the documented C API.
Treat the package as unmeasured until it runs against libonnxruntime.

The suite loads no model, reads and writes no file, and needs no
privileges. The first test asserts that `OrtGetApiBase` answers a
non-zero address, and that the two words in the structure are two
different non-zero function pointers. The second test names the CPU
provider factory in a branch that is never taken, because an
`OrtSessionOptions` cannot be made from here. The declaration is
compiled and linked all the same.

## Licence

Apache-2.0. See [LICENSE](LICENSE).

ONNX Runtime itself is distributed under the MIT licence, and installing
it is the reader's own step.
