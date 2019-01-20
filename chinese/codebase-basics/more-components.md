# 更多的模块

在这一部分，我们将会简短的讨论Deno依赖的其他重要的模块。

## V8

[V8](https://v8.dev/)是Google出品的JavaScript/WebAssembly引擎。他是由C++编写的，被Google Chrome浏览器 和 Node.js所使用。

V8并不是原生支持TypeScript。实际上，所有你运行在Deno上的TypeScript代码会被保存为快照形式的TS 编译器编译成JavaScript代码，同时生成的文件会被缓存在`.deno`目录。除非用户更新代码，否则缓存的js文件在编译后会被一直使用。

## FlatBuffers

[Flatbuffers](https://google.github.io/flatbuffers/)是一个由Google开发的，高效的跨平台的序列化库。他允许在不同的语言之间传递和接受消息，这其中没有打包和解包的性能损耗。

对于Deno来说，FlatBuffers被用来在进程内部的“特权方”和“没有特权方”之间进行消息通信。很多公开的Deno APIs内部，在TypeScript前台创建了包含序列化数据的buffers，然后这些buffers传递到Rust，使得Rust可以处理这些请求。在处理完这些请求后，Rust后台也创建了包含序列化数据的buffers回传给TypeScript，其中，反序列化这些buffers使用的是FlatBuffers编译器产生的文件。

相比较于Node，Node对于每一个特权调用创建了很多v8的扩展，而借助于FlatBuffers，Deno仅仅需要暴露出在TypeScript和Rust之间进行消息发送和消息接受的方法。

在Go原型中，Flatbuffers被用来取代[Protocol Buffers](https://developers.google.com/protocol-buffers/)，来避免序列化的性能损耗。具体可以查看[this thread](https://github.com/denoland/deno/issues/269)来获得更多的信息。

### 例子：通过FlatBuffers传递readFile数据

TypeScript 代码:

{% code-tabs %}
{% code-tabs-item title="read\_file.ts" %}
```typescript
import * as msg from "gen/msg_generated";
import * as flatbuffers from "./flatbuffers";
// dispatch被用来传递一个消息给Rust
import * as dispatch from "./dispatch";

export async function readFile(filename: string): Promise<Uint8Array> {
  return res(await dispatch.sendAsync(...req(filename)));
}

function req(
  filename: string
): [flatbuffers.Builder, msg.Any, flatbuffers.Offset] {
  // 用来序列化消息的Builder
  const builder = flatbuffers.createBuilder();
  const filename_ = builder.createString(filename);
  msg.ReadFile.startReadFile(builder);
  // 文件名被存入底层的ArrayBuffer
  msg.ReadFile.addFilename(builder, filename_);
  const inner = msg.ReadFile.endReadFile(builder);
  return [builder, msg.Any.ReadFile, inner];
}

function res(baseRes: null | msg.Base): Uint8Array {
  // ...
  const inner = new msg.ReadFileRes();
  // ...
  // Taking data out of FlatBuffers
  const dataArray = inner.dataArray();
  // ...
  return new Uint8Array(dataArray!);
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

Rust 代码:

{% code-tabs %}
{% code-tabs-item title="ops.rs" %}
```rust
fn op_read_file(
  _config: &IsolateState,
  base: &msg::Base,
  data: libdeno::deno_buf,
) -> Box<Op> {
  // ...
  let inner = base.inner_as_read_file().unwrap();
  let cmd_id = base.cmd_id();
  // 从序列化buffer中取出filename
  let filename = PathBuf::from(inner.filename().unwrap());
  // ...
  blocking(base.sync(), move || {
    // 真正的fs调用
    let vec = fs::read(&filename)?;
    // 序列化输出然后回传给TypeScript
    let builder = &mut FlatBufferBuilder::new();
    let data_off = builder.create_vector(vec.as_slice());
    let inner = msg::ReadFileRes::create(
      builder,
      &msg::ReadFileResArgs {
        data: Some(data_off),
      },
    );
    Ok(serialize_response(
      cmd_id,
      builder,
      msg::BaseArgs {
        inner: Some(inner.as_union_value()),
        inner_type: msg::Any::ReadFileRes,
        ..Default::default()
      },
    ))
  })
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

## Tokio

[Tokio](https://tokio.rs/) 是一个Rust编写的异步运行时。他被用来创建和处理事件。他允许Deno在一个内部的线程池创建任务，然后当任务完成后，接收到输出的消息通知。

Tokio依赖于Rust的`Future`特性，这个特性类似于JavaScript的Promises。

### 例子: 产生异步任务

在上面的`readFile`例子中，有一个`blocking`函数。他被用来决定一个任务被生成在主线程中运行还是交给Tokio的线程池。

{% code-tabs %}
{% code-tabs-item title="ops.rs" %}
```rust
fn blocking<F>(is_sync: bool, f: F) -> Box<Op>
where
  F: 'static + Send + FnOnce() -> DenoResult<Buf>,
{
  if is_sync {
    // 在主线程中运行任务
    Box::new(futures::future::result(f()))
  } else {
    // 把任务交给Tokio
    Box::new(tokio_util::poll_fn(move || convert_blocking(f)))
  }
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}



