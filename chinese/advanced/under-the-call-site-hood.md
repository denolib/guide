---
描述: >-
  在本节中，我们将深入源代码，了解TypeScript函数是如何与Rust实现进行通信的。
---

# 从调用的角度

接下来，我们将会使用`readFileSync`和`readFile`来揭示底层的原理。

## TypeScript调用

在Deno，我们可以像下面这样读取文件：

```typescript
import * as deno from "deno";

async function readFileTest() {
  // 异步读取文件
  const data1 = await deno.readFile("test.json");

  // 同步读取文件
  // 需要注意的是，我们不需要`await`关键字。
  const data2 = deno.readFile("test.json");
}
```

上面的代码是非常简单和直观的，但是究竟Typescript是如何读取文件的呢？没有了`libuv`，到底Deno是如何实现异步调用的呢？

事实上，需要进行系统调用的Deno APIs会像下面这样，被转换为消息处理：

```typescript
function readFileSync(filename: string): Uint8Array {
  return res(dispatch.sendSync(...req(filename)));
}

async function readFile(filename: string): Promise<Uint8Array> {
  return res(await dispatch.sendAsync(...req(filename)));
}
```

`sendSync`和`sendAsync`方法底层都调用了`libdeno.send`方法，这个方法是添加到V8的C++扩展。通过这样的处理，函数调用信息被传递给了C++。

除了分发消息，TypeScript端也需要注册回调函数来从Rust端获得异步回调的结果：

```typescript
// js/main.ts denoMain()

libdeno.recv(handleAsyncMsgFromRust);
```

`libdeno.recv`也是一个注入到V8的C++扩展函数，他用来注册接受消息的回调函数。了解更多的信息，请查看[下一节](https://github.com/denolib/guide/chinese/advanced/process-lifecycle.md)。

让我们看看C++是如何处理接受到的的消息的。

## C++转换器

注意：下面不是真实的`libdeno.send`代码，只是为了方便我们更好的理解而已，真实的代码不是这个样子的。

```cpp
// libdeno/binding.cc

void Send(const pseudo::Args args) {
  // 获取调用信息，比如调用函数名字等。
  pseudo::Value control = args[0];
  // 获取带有许多数据的参数。
  pseudo::Value data;
  // 获取请求id
  int32_t req_id = isolate->next_req_id_++;

  if (args.Length() == 2) {
    data = args[1];
  }

  isolate->current_args_ = &args;

  isolate->recv_cb_(req_id, control, data);
}
```

`Send`函数获取参数然后调用`recv_cb_`。需要注意的是`recv_cb_`是定义在Rust代码中的。

`libdeno.send`的返回值将会被`deno_respond`设置：

```cpp
// libdeno/api.cc
// 这也不是真实的代码

int deno_respond(DenoIsolate* d, int32_t req_id, deno_buf buf) {
  // 同步响应。
  if (d->current_args_ != nullptr) {
    // 获取作为Uint8Array类型的返回值
    auto ab = deno::ImportBuf(d, buf);
    // 设置返回值
    d->current_args_->GetReturnValue().Set(ab);
    d->current_args_ = nullptr;
    return 0;
  }

  // 异步响应。
  // 获取定义在TypeScript中的回调函数。
  auto recv_ = d->recv_.Get(d->isolate_);
  if (recv_.IsEmpty()) {
    d->last_exception_ = "libdeno.recv_ has not been called.";
    return 1;
  }

  pseudo::Value args[1];
  // 获取作为Uint8Array类型的返回值
  args[0] = deno::ImportBuf(d, buf);
  // 调用TypeScript函数
  auto v = recv_->Call(context, context->Global(), 1, args);

  return 0;
}
```

`deno_respond`函数是被`recv_cb_`调用的，并且当被执行的时候，会区分同步和异步。当调用是同步的，`deno_respond`直接返回结果。当调用是异步的，函数是被异步触发的，然后调用定义在TypeScript中的函数。

## Rust执行器

函数调用之旅，进入到了最后的阶段。Rust代码映射不同类型的函数调用到相应的处理操作，并且同步和异步调用是基于事件循环的。对于事件循环的更多内容，请查看[下一节](https://github.com/denolib/guide/chinese/advanced/process-lifecycle.md)

