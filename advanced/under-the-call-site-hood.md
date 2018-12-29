---
description: >-
  In this section, we'll dive into the source code to get an idea of how
  TypeScript functions communicate with Rust implementations.
---

# Under the call site's hood

In the following, we'll use the `readFileSync` and `readFile` to illustrate the underlying principles.

## TypeScript call site

In Deno, we can read files like this:

```typescript
import * as deno from "deno";

async function readFileTest() {
  // Read file asynchronously
  const data1 = await deno.readFile("test.json");

  // Read file synchronously
  // Note that we don't need `await` keyword.
  const data2 = deno.readFile("test.json");
}
```

The above code is very simple and straightforward, but how does Typescript read the file actually? And in the absence of `libuv`, how does Deno implement asynchronous calls?

In fact, the Deno APIs that need to use system calls are converted to messaging process like the following:

```typescript
function readFileSync(filename: string): Uint8Array {
  return res(dispatch.sendSync(...req(filename)));
}

async function readFile(filename: string): Promise<Uint8Array> {
  return res(await dispatch.sendAsync(...req(filename)));
}
```

Both the `sendSync` and `sendAsync` methods call the `libdeno.send` method on the underlying, and this method is a C++ binding function injected by V8. After this processing, the function call information is passed to C++. 

In addition to dispatching messages, TypeScript end also needs to register a callback function to get the asynchronous result sent back from the Rust side:

```typescript
// js/main.ts denoMain()

libdeno.recv(handleAsyncMsgFromRust);
```

`libdeno.recv` is also a binding function injected in V8 to register callback for receiving messages. For details on this function, please refer to [the next section](https://denolib.gitbook.io/guide/~/drafts/-LUtKIIrxNaQMXGyMSgw/primary/advanced/process-lifecycle#rust-main-entry-point).

Let's take a look at how C++ consumes received messages.

## C++ converter

Note: The following is pseudo `libdeno.send` code, just for the sake of easy understanding, the actual code is not like this.

```cpp
// libdeno/binding.cc

void Send(const pseudo::Args args) {
  // get the call inforamtion, such as function name etc.
  pseudo::Value control = args[0];
  // get args with a large amount of data.
  pseudo::Value data;
  // get the request id
  int32_t req_id = isolate->next_req_id_++;

  if (args.Length() == 2) {
    data = args[1];
  }

  isolate->current_args_ = &args;

  isolate->recv_cb_(req_id, control, data);
}
```

The `Send` function get args and invoke the `recv_cb_` . Notice the `recv_cb_` function here, which is defined in the Rust code.

The return value of the `libdeno.send` function will be set by `deno_respond`:

```cpp
// libdeno/api.cc

int deno_respond(DenoIsolate* d, int32_t req_id, deno_buf buf) {
  // Synchronous response.
  if (d->current_args_ != nullptr) {
    // get return value as Uint8Array
    auto ab = deno::ImportBuf(d, buf);
    // set return value
    d->current_args_->GetReturnValue().Set(ab);
    d->current_args_ = nullptr;
    return 0;
  }

  // Asynchronous response.
  // get receive callback function defined in TypeScript
  auto recv_ = d->recv_.Get(d->isolate_);
  if (recv_.IsEmpty()) {
    d->last_exception_ = "libdeno.recv_ has not been called.";
    return 1;
  }

  pseudo::Value args[1];
  // get return value as Uint8Array
  args[0] = deno::ImportBuf(d, buf);
  // call the TypeScript function
  auto v = recv_->Call(context, context->Global(), 1, args);

  return 0;
}
```



## Rust executor



