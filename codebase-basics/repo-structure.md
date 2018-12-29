# Repo Structure

In this section, we will explore the current Deno repository structure, such that you won't feel lost.

Here is the link to the repo: [https://github.com/denoland/deno](https://github.com/denoland/deno)

We will list some of the important directories/files you might need to interact with, and briefly introduce each of them:

```text
# DIRECTORIES
build_extra/: Some customized rules of building Deno
js/: TypeScript Frontend
libdeno/: C/C++ Middle-end
src/: Rust Backend
tests/: Integration Tests
third_party/: Third party code
tools/: Deno tools for testing and building

# FILES
BUILD.gn: Main Deno build rules
Cargo.toml: Dependency info for Rust end
package.json: Node dependencies (mostly for TypeScript compiler and Rollup)
build.rs: Cargo build rules
```

## build\_extra/

Some customized rules for building Deno, consisting of rules for FlatBuffers and Rust crates at the moment.

One important file you might want to pay attention to is `build_extra/rust/BUILD.gn`. When adding a new Rust crate in `third_party/`, we need to add an entry into this file. For example, this is the rule for building a crate called `time`:

{% code-tabs %}
{% code-tabs-item title="build\_extra/rust/BUILD.gn" %}
```
rust_crate("time") {
  # Where is this crate?
  source_root = "$registry_github/time-0.1.40/src/lib.rs"
  # What are its dependencies?
  extern = [
    ":libc",
    ":winapi",
  ]
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

\(`rust_crate` GN template is defined in `build_extra/rust/rust.gni`\)

## js/

This is where code for TypeScript frontend is located. Most APIs are defined in its own file, accompanied with a test file. For example, `readFile` is defined in `read_file.ts`, while its unit tests are in `read_file_test.ts`.

Besides files for API, here are some other special files:

1. `main.ts`: `denoMain` is defined in this file, which is invoked by Rust backend as the entry for bootstrapping TypeScript code.
2. `globals.ts`: All globals that do not need imports are attached here.
3. `deno.ts`: All `deno` namespace APIs are exported here.
4. `libdeno.ts`: Type definition for customized V8 API exposed to TypeScript.
5. `unit_tests.ts`: All unit tests are imported here and executed.
6. `compiler.ts`: Customized TypeScript compilation logic for Deno.

## libdeno/

The C/C++ libdeno middle-end lives here.

1. `api.cc`: libdeno APIs that are exposed and callable from the Rust side
2. `binding.cc`: code that handles interaction with V8 and where V8 C++ bindings are added. These APIs are exposed Rust and TypeScript side both.
3. `snapshot_creator.cc`: logic for creating V8 snapshot during build.

## src/

The Rust backend lives here.

1. `libdeno.rs`: exposes libdeno APIs from `libdeno/api.cc` to Rust
2. `isolate.rs`: extracts V8 isolate \(an isolated instance of V8 engine\) and event loop creation logic using libdeno APIs
3. `deno_dir.rs`: contains logic for Deno module file resolution and caching
4. `ops.rs`: where each Deno Op \(operation\) is handled. This is where the actual file system and network accesses are done.
5. `main.rs`: contains main function for Rust. This is the **ACTUAL** entry point when you run the Deno binary.
6. `msg.fbs`: FlatBuffers definition file for message structure, used for message passing. Modify this file when you want to add or modify a message structure.

## tests/

Integration tests for Deno.

`*.js`or `*.ts` files define the code to run for each test. `*.test` defined how should this test be executed. Usually, `*.out` defines the expected output of a test \(the actual name is specified in `*.test` files\) 

See [tests/README.md](https://github.com/denoland/deno/blob/master/tests/README.md) for more details.

## third\_party/

Third party code for Deno. Contains actual node modules, v8, Rust crate code and so on. It is placed in [https://github.com/denoland/deno\_third\_party](https://github.com/denoland/deno_third_party).

## tools/

Mostly Python scripts for a range of purposes.

1. `build.py`: Builds Deno
2. `setup.py`: Setup and necessary code fetching
3. `format.py`: Code formatting
4. `lint.py`: Code linting
5. `sync_third_party.py`: Sync third\_party code after user modifies `package.json`, `Cargo.toml` , etc.

## BUILD.gn

Contains the most important Deno build logic. We will visit this file later in the example of adding a new Deno call.

## build.rs

Defines `cargo build` logic. Internally invokes `BUILD.gn`.

