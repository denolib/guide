# 进程生命周期

## Rust main\(\) 入口

一个Deno进程的启动入口是`src/main.rs`文件的`main`函数：

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

在处理了flags之后，Deno接着创建了一个V8 isolate\(一个独立的拥有自己的堆的V8虚拟机实例\)。我们同时也传入了snapshot，snapshot包含了序列化的堆数据，而堆数据则是关于TypeScript编译器和一些前台代码的。这让Deno的启动更加快。

紧跟着，我们启动Tokio，并且让isolate去执行一个叫做`denoMain`的函数。只有在完成了这个函数的运行之后，Deno开始了他的事件循环。一旦事件循环结束，Deno进程结束。


## denoMain\(\)

`denoMain`位于文件`js/main.ts`。

{% code-tabs %}
{% code-tabs-item title="js/main.ts" %}
```typescript
export default function denoMain() {
  libdeno.recv(handleAsyncMsgFromRust);

  const startResMsg = sendStart();
  setLogDebug(startResMsg.debugFlag());

  const compiler = Compiler.instance();
  
  // ... 被省略的代码

  const cwd = startResMsg.cwd();
  
  // ... 被省略的代码

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


首先，我们告诉`libdeno`，当从Rust端有消息发送过来，就把buffer转发给`handleAsyncMsgFromRust`函数，`libdeno`是从中台暴露出来的API。然后，一个`Start`消息被发送给Rust端，表明我们正在启动和接受包含当前工作目录`cwd`的信息。然后我们我们决定用户是运行脚本还是使用REPL。如果我们传入了一个文件名，Deno会尝试让`runner`运行这个文件， `runner`包含了TypeScript `compiler`\(当你深入该函数的定义，你会发现在某处一个神秘的`eval/globalEval`被调用\)。仅当`runner`运行完这个文件，`denoMain`才会结束。


## isolate.event\_loop\(\)

当`denoMain`运行结束，一系列的异步调用被启动，并且还没有得到响应。因此Deno接着调用`isolate.event_loop()`来开始事件循环。

`event_loop`的代码是在`src/isolate.rs`文件中：

{% code-tabs %}
{% code-tabs-item title="src/isolate.rs" %}
```rust
pub fn event_loop(&self) -> Result<(), JSError> {
    // 主线程事件循环
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
    // 结束后的检查
    self.check_promise_errors();
    if let Some(err) = self.last_exception() {
      return Err(err);
    }
    Ok(())
  }
```
{% endcode-tabs-item %}
{% endcode-tabs %}

这就是一个循环，每一次，我们调用`self.is_idle()`检查是否还有事件没有得到响应。如果是，我们将会等待`self.rx`来接受一些消息或者一个由`setTimeout`创建的定时器超时，`self.rx`是消息通道的接收端。

如果`self.rx`接受到一些消息，这意味着Tokio线程池里面有一个任务已经完成了。我们接下来将会把执行结果发送给`self.complete_op`来处理，执行结果是由`req_id`\(任务id\)和`buf`\(包含序列化响应的buffer\)两个组成的。`self.complete_op`根据`req_id`调用正在等待结果的代码，运行到结束。需要注意的是，这里被调用的代码，可能会加入更多的异步操作。

如果发生了超时，由`setTimeout`加入的代码将会被调用，然后运行到结束。

事件循环将会继续等待输出和运行更多的代码，直到`self.is_idle`返回`true`为止。当没有挂起的任务时，`self.is_idle`返回true，然后，事件循环退出，Deno结束运行。

当执行代码的时候，有可能会有错误发生，如果错误发生了，Deno将会退出。如果是一个promise错误\(比如未捕获的错误\)，Deno将会调用`self.check_promise_errors`来进行检查。

## Promise表

在上面，我们提到，`req_id`作为一个任务id提供给了`self.complete_op`。事实上，他会被用来从一个promise表中查找获取相应的promise，promise表中的promise是由异步操作创建的，promise表包含了所有等待resolve的promise。

promise表位于文件`src/dispatch.ts`

{% code-tabs %}
{% code-tabs-item title="src/dispatch.ts" %}
```typescript
const promiseTable = new Map<number, util.Resolvable<msg.Base>>();
```
{% endcode-tabs-item %}
{% endcode-tabs %}

Promise表是由`dispatch.sendAsync`函数填充的，`dispatch.sendAsync`被用来转发异步请求给Rust。

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

这里的`cmdId`其实就是Rust端的`req_id`。他是由`sendInternal`返回的。`sendInternal`调用了V8的扩展并且递增了一个全局的请求id计数。对于每一个异步操作，我们创建了一个deferred/resolvable，并以`cmdId`为key存入promise表中。

在`denoMain`部分，我们知道，`libdeno`绑定了`handleAsyncMsgFromRust`作为接受新消息的回调。下面是他的源码：

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

我们注意到，他从响应里面取出`cmdId`，然后从表里面取出相应的promise。这个promise接着被resolve，允许等待该任务响应的代码执行。

## 生命周期例子

对于像下面这样的例子：

```typescript
// RUN FILE
import * as deno from 'deno';

const dec = new TextDecoder();

(async () => {
    // 打印 "A"
    console.log(await deno.readFile('a.c');
    // 打印 "B"
    console.log(dec.decode(deno.readFileSync('b.c')));
    // 打印 "C"
    console.log(await deno.readFile('c.c');
})();

// 打印 "D"
console.log(dec.decode(deno.readFileSync('d.c')));

// END FILE
```

Deno进程生命周期控制流图看起来像这样：

```text
            Rust             TypeScript (V8)

            main
              |
         创建 isolate
              |
              |    执行
              |--------------> denoMain
                                   |
                    startMsg       |
              |<-------------------|
              |------------------->|
                   startResMsg     |
                                   |
                                RUN FILE
                  readFile('a.c')  |
      (线程池) |<-------------------|
                                   |
                                   |
                readFileSync('d.c')|
              |<-------------------|
              |
           堵塞的执行
              |
              |------------------->|
                 (Uint8Array) "D"  |
                             console.log("D")
                                   |
                    END FILE       |
              |....................|   # 没有更多的同步代码需要执行
              |
           事件循环
              |
          更多的任务
              |
              | 'a.c' 结束读取
      (线程池) |------------------->|
                 (Uint8Array) "A"  |
                            console.log("A")
                                   |
                readFileSync('b.c')|
              |<-------------------|
              |
           堵塞的执行
              |
              |------------------->|
                 (Uint8Array) "B"  |
                            console.log("B")
                                   |
                  readFile('c.c')  |
      (线程池) |<-------------------|
                                   |
              |....................|   # 没有更多的同步代码需要执行
              |
          更多的任务
              |
              | 'c.c' 结束读取
      (线程池) |------------------->|
                 (Uint8Array) "C"  |
                            console.log("C")
                                   |
                                   |
              |....................|   # 没有更多的同步代码需要执行
              |
           没有任务
              |
              |
              v
             退出
```



