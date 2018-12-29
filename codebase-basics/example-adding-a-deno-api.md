# Example: Adding to Deno API

In this example, we will walk through a simple example that adds the functionality of generating a random number inside of a range.

\(This is just an example. You can definitely implement with JavaScript `Math.random` if you want.\)

## Define Messages

As mentioned in previous sections, Deno relies on message passing to communicate between TypeScript to Rust and do all the fancy stuff. Therefore, we must first define the messages we want to send and receive.

Go to `src/msg.fbs` and do the following:

{% code-tabs %}
{% code-tabs-item title="src/msg.fbs" %}
```text
union Any {
    // ... Other message types, omitted
    // ADD THE FOLLOWING LINES
    RandRange,
    RandRangeRes
}

// ADD THE FOLLOWING TABLES
table RandRange {
  from: int32;
  to: int32;
}

table RandRangeRes {
  result: int32;
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

With such code, we tell FlatBuffers that we want 2 new message types, `RandRange` and `RandRangeRes`. `RandRange` contains the range `from` and `to` we want. `RandRangeRes` contains a single value `result` that represents the random number we will get from the Rust side.

Now, run `./tools/build.py` to update the corresponding serialization and deserialization code. Such auto-generated code are store in `target/debug/gen/msg_generated.ts` and `target/debug/gen/msg_generated.rs`, if you are interested.

## Add Frontend Code

Go to `js/` folder. We will now create the TypeScript frontend interface.

Create a new file, `js/rand_range.ts`. Add the following imports:

{% code-tabs %}
{% code-tabs-item title="js/rand\_range.ts" %}
```typescript
import * as msg from "gen/msg_generated";
import * as flatbuffers from "./flatbuffers";
import { assert } from "./util";
import * as dispatch from "./dispatch";
```
{% endcode-tabs-item %}
{% endcode-tabs %}

We will be using `dispatch.sendAsync` and `dispatch.sendSync` to dispatch the request as async or sync operation.

Since we are dealing with messages, we need to do some serialization and deserialization work. For request, we need to put user provided `from` and `to` to the table `RandRange` \(as defined above\); for responses, we need to extract the `result` from the table `RandRangeRes`.

Therefore, let's define 2 functions, `req` and `res`, that does the work:

{% code-tabs %}
{% code-tabs-item title="js/rand\_range.ts" %}
```typescript
function req(
  from: number,
  to: number,
): [flatbuffers.Builder, msg.Any, flatbuffers.Offset] {
  // Get a builder to create a serialized buffer
  const builder = flatbuffers.createBuilder();
  msg.RandRange.startRandRange(builder);
  // Put stuff inside the buffer!
  msg.RandRange.addFrom(builder, from);
  msg.RandRange.addTo(builder, to);
  const inner = msg.RandRange.endRandRange(builder);
  // We return these 3 pieces of information.
  // dispatch.sendSync/sendAsync will need these as arguments!
  // (treat such as boilerplate)
  return [builder, msg.Any.RandRange, inner];
}

function res(baseRes: null | msg.Base): number {
  // Some checks
  assert(baseRes !== null);
  // Make sure we actually do get a correct response type
  assert(msg.Any.RandRangeRes === baseRes!.innerType());
  // Create the RandRangeRes template
  const res = new msg.RandRangeRes();
  // Deserialize!
  assert(baseRes!.inner(res) !== null);
  // Extract the result
  return res.result();
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

Great! With `req` and `res` defined, we can very easily define the actual sync/async APIs:

{% code-tabs %}
{% code-tabs-item title="js/rand\_range.ts" %}
```typescript
// Sync
export function randRangeSync(from: number, to: number): number {
  return res(dispatch.sendSync(...req(from, to)));
}

// Async
export async function randRange(from: number, to: number): Promise<number> {
  return res(await dispatch.sendAsync(...req(from, to)));
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

## Expose the Interface

Go to `js/deno.ts` and add the following line:

{% code-tabs %}
{% code-tabs-item title="js/deno.ts" %}
```typescript
export { randRangeSync, randRange } from "./rand_range";
```
{% endcode-tabs-item %}
{% endcode-tabs %}

Also, go to `BUILD.gn` and add our file to `ts_sources`:

{% code-tabs %}
{% code-tabs-item title="BUILD.gn" %}
```text
ts_sources = [
  "js/assets.ts",
  "js/blob.ts",
  "js/buffer.ts",
  # ...
  "js/rand_range.ts"
]
```
{% endcode-tabs-item %}
{% endcode-tabs %}

## Add Backend Code

Go to `src/ops.rs`. This is the only Rust backend file we need to modify to add this functionality.

Inside `ops.rs`, let's first import the utility for generating random numbers. Fortunately, Deno already includes a Rust crate called `rand` that could handle the job. See more about its API [here](https://docs.rs/rand/0.6.1/rand/).

Let's import it:

{% code-tabs %}
{% code-tabs-item title="src/ops.rs" %}
```rust
use rand::{Rng, thread_rng};
```
{% endcode-tabs-item %}
{% endcode-tabs %}

Now, we can create a function \(with some boilerplate code\) that implements the random range logic:

{% code-tabs %}
{% code-tabs-item title="src/ops" %}
```rust
fn op_rand_range(
  _state: &IsolateState,
  base: &msg::Base,
  data: libdeno::deno_buf,
) -> Box<Op> {
  assert_eq!(data.len(), 0);
  // Decode the message as RandRange
  let inner = base.inner_as_rand_range().unwrap();
  // Get the command id, used to respond to async calls
  let cmd_id = base.cmd_id();
  // Get `from` and `to` out of the buffer
  let from = inner.from();
  let to = inner.to();

  // Wrap our potentially slow code and respond code here
  // Based on dispatch.sendSync and dispatch.sendAsync,
  // base.sync() will be true or false.
  // If true, blocking() will spawn the task on the main thread
  // Else, blocking() would spawn it in the Tokio thread pool
  blocking(base.sync(), move || -> OpResult {
    // Actual random number generation code!
    let result = thread_rng().gen_range(from, to);
    
    // Prepare respond message serialization
    // Treat these as boilerplate code for now
    let builder = &mut FlatBufferBuilder::new();
    // We want the message type to be RandRangeRes
    let inner = msg::RandRangeRes::create(
      builder,
      &msg::RandRangeResArgs {
        result, // put in our result here
      },
    );
    // Get message serialized
    Ok(serialize_response(
      cmd_id, // Used to reply to TypeScript if this is an async call
      builder,
      msg::BaseArgs {
        inner: Some(inner.as_union_value()),
        inner_type: msg::Any::RandRangeRes,
        ..Default::default()
      },
    ))
  })
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

After completing our implementation, let's tell Rust when we should invoke it.

Notice there is a function called `dispatch` in `src/ops.rs`. Inside, we could find a `match` statement on message types to set corresponding handlers. Let's set our function to be the handler of the `msg::Any::RandRange` type, by inserting this line:

{% code-tabs %}
{% code-tabs-item title="src/ops.rs" %}
```rust
pub fn dispatch(
    // ...
) -> (bool, Box<Op>) {
    // ...
    let op_creator: OpCreator = match inner_type {
        msg::Any::Accept => op_accept,
        msg::Any::Chdir => op_chdir,
        // ...
        /* ADD THE FOLLOWING LINE! */
        msg::Any::RandRange => op_rand_range,
        // ...
        _ => panic!(format!(
            "Unhandled message {}",
            msg::enum_name_any(inner_type)
          )),
    }
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

Done! This is all the code you need to add for this call!

Run `./tools/build.py` to build your project! You would now be able to call `deno.randRangeSync` and `deno.randRange` .

Try it out in the terminal!

```bash
$ ./target/debug/deno
> deno.randRangeSync(0, 100)
96
> (async () => console.log(await deno.randRange(0, 100)))()
Promise {}
> 74
```

Congrats!

## Add Tests

Now let's add a basic test for our code.

Create a file `js/rand_range_test.ts`:

{% code-tabs %}
{% code-tabs-item title="js/rand\_range\_ts.ts" %}
```typescript
import { test, assert } from "./test_util.ts";
import * as deno from "deno";

test(function randRangeSync() {
  const v = deno.randRangeSync(0, 100);
  assert(0 <= v && v < 100);
});

test(async function randRange() {
  const v = await deno.randRange(0, 100);
  assert(0 <= v && v < 100);
});
```
{% endcode-tabs-item %}
{% endcode-tabs %}

Include this test in `js/unit_tests.ts`:

{% code-tabs %}
{% code-tabs-item title="js/unit\_tests.ts" %}
```typescript
// ADD THIS LINE
import "./rand_range_test.ts";
```
{% endcode-tabs-item %}
{% endcode-tabs %}

Now, run `./tools/test.py`. From all the tests, you will be able to find that your tests are passing:

```text
test randRangeSync_permW0N0E0R0
... ok
test randRange_permW0N0E0R0
... ok
```

## Submit a PR!

Let's suppose that Deno really needs this functionality and you are resolved to contribute to  Deno. You can submit a PR!

Remember to run the following commands to ensure your code is ready for submit!

```bash
./tools/format.py
./tools/lint.py
./tools/build.py
./tools/test.py
```



