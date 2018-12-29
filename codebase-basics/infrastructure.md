# Infrastructure

Deno is composed of mainly 3 separate parts:

## 1. TypeScript Frontend

Public interfaces, APIs and important most functionalities that do not directly require syscalls are implemented here. An automatically generated TypeScript API doc is available at [https://deno.land/typedoc](https://deno.land/typedoc) at this moment \(which might be changed in the future\).

TypeScript/JavaScript is considered as the "unprivileged side" which does not, by default, have access to the file system or the network \(as they are run on V8, which is a sandboxed environment\). These are only made possible through message passing to the Rust backend, which is "privileged".

Therefore, many Deno APIs \(especially file system calls\) are implement on the TypeScript end as purely creating buffers for data, sending them to the Rust backend through `libdeno` middle-end bindings, and waiting \(synchronously or asynchronously\) for the result to be sent back.

## 2. C++ libdeno Middle-end

`libdeno` is a thin layer between the TypeScript frontend and Rust backend, serving to interact with V8 and expose only necessary bindings. It is implemented with C/C++.

`libdeno` exposes Send and Receive message passing APIs to both TypeScript and Rust sides. It is also used to initiate V8 platforms/isolates and create/load V8 snapshot \(a data blob that represents serialized heap information. See [https://v8.dev/blog/custom-startup-snapshots](https://v8.dev/blog/custom-startup-snapshots) for details\). Worth noting, the snapshot for Deno also contains a complete TypeScript compiler. This allows much shorter compiler startup time.

## 3. Rust Backend

Currently, the backend, or the "privileged side" that has file system, network and environment access, is implemented in Rust.

For those who are not familiar with the language, Rust is a systems programming language developed by Mozilla, focusing on memory and concurrency safeties. It is also used in projects like [Servo](https://servo.org/).

The Rust backend is migrated from Go, which served to create the original Deno prototype present in June 2018. Reasons for the switch is due to concerns about double GC. Read more in [this thread](https://github.com/denoland/deno/issues/205).

