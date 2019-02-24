# Process Lifecycle

## Rust main\(\) Entry Point

A Deno process starts with the `main` function in `src/main.rs`:

{% code-tabs %}
{% code-tabs-item title="src/main.rs" %}
```rust
fn main() {
  log::set_logger(&LOGGER).unwrap();
  let args = env::args().collect();
  let (flags, rest_argv, usage_string) =
    flags::set_flags(args).unwrap_or_else(|err| {
      eprintln!("{}", err);
      std::process::exit(1)
    });

  if flags.help {
    println!("{}", &usage_string);
    std::process::exit(0);
  }

  log::set_max_level(if flags.log_debug {
    log::LevelFilter::Debug
  } else {
    log::LevelFilter::Warn
  });

  let state = Arc::new(isolate::IsolateState::new(flags, rest_argv));
  let snapshot = snapshot::deno_snapshot();
  let isolate = isolate::Isolate::new(snapshot, state, ops::dispatch);
  tokio_util::init(|| {
    isolate
      .execute("denoMain();")
      .unwrap_or_else(print_err_and_exit);
    isolate.event_loop().unwrap_or_else(print_err_and_exit);
  });
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

After processing the flags, Deno first creates a V8 isolate \(an isolated V8 VM instance with its own heap\). We also pass in the snapshot here, which contains serialized heap data about TypeScript compiler and frontend code. This allows much faster startup speed.

Immediately afterwards, we setup Tokio, and ask the isolate to execute a function called `denoMain`. Only after it has finished running this function does Deno starting its event loop. Once the event loop stops, the Deno process dies.

## denoMain\(\)

`denoMain` is located in `js/main.ts`.

{% code-tabs %}
{% code-tabs-item title="js/main.ts" %}
```typescript
export default function denoMain() {
  libdeno.recv(handleAsyncMsgFromRust);

  const startResMsg = sendStart();
  setLogDebug(startResMsg.debugFlag());

  const compiler = Compiler.instance();
  
  // ... code omitted

  const cwd = startResMsg.cwd();
  
  // ... code omitted

  const runner = new Runner(compiler);

  if (inputFn) {
    runner.run(inputFn, `${cwd}/`);
  } else {
    replLoop();
  }
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

First, we tell `libdeno`, which is the exposed API from the middle-end, that whenever there is a message sent from the Rust side, please forward the buffer to a function called `handleAsyncMsgFromRust`. Then, a `Start` message is sent to Rust side, signaling that we are starting and receiving information including the current working directory, or `cwd`. We then decide whether the user is running a script, or using the REPL. If we have an input file name, Deno would then try to let the `runner`, which contains the TypeScript `compiler`, to try running the file \(going deep into its definition, you'll eventually find a secret `eval/globalEval` called somewhere\). `denoMain` only exits when the `runner` finish running the file.

## isolate.event\_loop\(\)

With `denoMain` running to its end, a set of asynchronous calls might have been made without any of them being responded. Deno would then call the `isolate.event_loop()` to start the loop.

Code for `event_loop` is located in `src/isolate.rs`:

{% code-tabs %}
{% code-tabs-item title="src/isolate.rs" %}
```rust
pub fn event_loop(&self) -> Result<(), JSError> {
    // Main thread event loop.
    while !self.is_idle() {
      match recv_deadline(&self.rx, self.get_timeout_due()) {
        Ok((req_id, buf)) => self.complete_op(req_id, buf),
        Err(mpsc::RecvTimeoutError::Timeout) => self.timeout(),
        Err(e) => panic!("recv_deadline() failed: {:?}", e),
      }
      self.check_promise_errors();
      if let Some(err) = self.last_exception() {
        return Err(err);
      }
    }
    // Check on done
    self.check_promise_errors();
    if let Some(err) = self.last_exception() {
      return Err(err);
    }
    Ok(())
  }
```
{% endcode-tabs-item %}
{% endcode-tabs %}

This is literally a loop. Every single time, we check if there is still some events we haven't responded, with `self.is_idle()`. If yes, we will wait for `self.rx`, which is the receiving end of a message channel, to receive some message, or wait for a timer, created by `setTimeout`, to expire.

If `self.rx` received some message, this means that one of the task in the Tokio thread pool has completed its execution. We would then process its execution result, which is a pair of `req_id` \(the task id\) and `buf` \(the buffer containing serialized response\), by sending them to `self.complete_op`, which forwards the result based on request id and invoke the code that is waiting for this output, running to the end. Notice that the code invoked here might schedule for more async operation. 

If timeout happens, code scheduled to execute by `setTimeout` would similarly be invoked and running to the end.

The event loop would continue to wait for output and run more code, until `self.is_idle` becomes `true`. This happens when there are no more pending tasks. Then, the event loop exits and Deno quits.

It is possible that an error might happen when executing code, and Deno would exit if such error exists. If it is a promise error \(e.g. uncaught rejection\), Deno would invoke `self.check_promise_errors` to check.

## Promise Table

In the previous section, we mentioned that `req_id` is provided to `self.complete_op` as a task id. Actually, it is used to retrieve the correct promise created by the past async operation from a promise table, which holds all the promises that are pending resolve.

The promise table is located in `src/dispatch.ts` 

{% code-tabs %}
{% code-tabs-item title="src/dispatch.ts" %}
```typescript
const promiseTable = new Map<number, util.Resolvable<msg.Base>>();
```
{% endcode-tabs-item %}
{% endcode-tabs %}

Promise table is populated by `dispatch.sendAsync`, which is used to forward async requests to Rust.

{% code-tabs %}
{% code-tabs-item title="src/dispatch.ts" %}
```typescript
// @internal
export function sendAsync(
  builder: flatbuffers.Builder,
  innerType: msg.Any,
  inner: flatbuffers.Offset,
  data?: ArrayBufferView
): Promise<msg.Base> {
  const [cmdId, resBuf] = sendInternal(builder, innerType, inner, data, false);
  util.assert(resBuf == null);
  const promise = util.createResolvable<msg.Base>();
  promiseTable.set(cmdId, promise);
  return promise;
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

The `cmdId` here is essentially the `req_id` on the Rust side. It is returned by `sendInternal`, which calls into V8 Send binding and increments a global request id counter. For each async operation, we create a deferred/resolvable and place it into the promise table with `cmdId` as the key.

From the `denoMain` section, we saw that we attach `handleAsyncMsgFromRust` to `libdeno` as the receive callback whenever a new message arrives. Here is its source code:

```typescript
export function handleAsyncMsgFromRust(ui8: Uint8Array) {
  // If a the buffer is empty, recv() on the native side timed out and we
  // did not receive a message.
  if (ui8.length) {
    const bb = new flatbuffers.ByteBuffer(ui8);
    const base = msg.Base.getRootAsBase(bb);
    const cmdId = base.cmdId();
    const promise = promiseTable.get(cmdId);
    util.assert(promise != null, `Expecting promise in table. ${cmdId}`);
    promiseTable.delete(cmdId);
    const err = errors.maybeError(base);
    if (err != null) {
      promise!.reject(err);
    } else {
      promise!.resolve(base);
    }
  }
  // Fire timers that have become runnable.
  fireTimers();
}
```

Notice that it grabs the `cmdId` from the response and find the corresponding Promise in the table. The promise is then resolved, allowing code awaiting the response to be executed as microtasks.

## Lifecycle Example

For a file looks like the following:

```typescript
// RUN FILE
import * as deno from 'deno';

const dec = new TextDecoder();

(async () => {
    // prints "A"
    console.log(dec.decode(await deno.readFile('a.c')));
    // prints "B"
    console.log(dec.decode(deno.readFileSync('b.c')));
    // prints "C"
    console.log(dec.decode(await deno.readFile('c.c')));
})();

// prints "D"
console.log(dec.decode(deno.readFileSync('d.c')));

// END FILE
```

The Deno process lifecycle control flow graph looks like this:

```text
            Rust             TypeScript (V8)

            main
              |
        create isolate
              |
              |    execute
              |--------------> denoMain
                                   |
                    startMsg       |
              |<-------------------|
              |------------------->|
                   startResMsg     |
                                   |
                                RUN FILE
                  readFile('a.c')  |
(thread pool) |<-------------------|
                                   |
                                   |
                readFileSync('d.c')|
              |<-------------------|
              |
           blocking
           execution
              |
              |------------------->|
                 (Uint8Array) "D"  |
                             console.log("D")
                                   |
                    END FILE       |
              |....................|   # no more sync code to exec
              |
          event loop
              |
          more tasks
              |
              | 'a.c' done reading
(thread pool) |------------------->|
                 (Uint8Array) "A"  |
                            console.log("A")
                                   |
                readFileSync('b.c')|
              |<-------------------|
              |
           blocking
           execution
              |
              |------------------->|
                 (Uint8Array) "B"  |
                            console.log("B")
                                   |
                  readFile('c.c')  |
(thread pool) |<-------------------|
                                   |
              |....................|   # no more sync code to exec
              |
          more tasks
              |
              | 'c.c' done reading
(thread pool) |------------------->|
                 (Uint8Array) "C"  |
                            console.log("C")
                                   |
                                   |
              |....................|   # no more sync code to exec
              |
           no task
              |
              |
              v
             EXIT
```



