# 例子: 贡献一个新的api给Deno

在这个部分，我们将会演示一个简单的例子，这个例子给Deno增加了产生某个范围内的一个随机数的功能。

\(这仅仅是一个例子，你完全可以用JavaScript的`Math.random`api完成。\)

## 定义消息

正如在前面几节我们提到的，Deno依靠消息传递在TypeScript和Rust之间进行通信来完成所有的工作。因此，我们必须首先定义要发送和接收的消息。

打开`src/msg.fbs`文件，然后做如下操作：

{% code-tabs %}
{% code-tabs-item title="src/msg.fbs" %}
```text
union Any {
    // ... 其他消息，忽略
    // 添加如下几行
    RandRange,
    RandRangeRes
}

// 添加下面的tables
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

通过这些代码，我们告诉FlatBuffers我们想要两个新的消息类型，`RandRange`和`RandRangeRes`。`RandRange`包含`from`到`to`的范围。`RandRangeRes`包含一个值，表示我们从Rsut中产生的一个随机数。

现在，运行`./tools/build.py`来更新相应的序列化和反序列化代码。自动生成的代码被保存在`target/debug/gen/msg_generated.ts`和`target/debug/gen/msg_generated.rs`文件，如果你感兴趣，可以看看这两个文件。

## 添加前台代码

进入`js/`目录。我们将会新增TypeScript前台接口。

创建一个新文件，`js/rand_range.ts`。增加下面的imports。

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

我们将会使用`dispatch.sendAsync`和`dispatch.sendSync`来分发异步或者同步操作。

因为我们处理的是消息，所以我们想要做一些序列化和反序列化的工作。对于请求，我们需要提供`from`和`to`给`RandRange`表；对于响应，我们需要从表`RandRangeRes`中提取`result`。

因此，让我们定义两个函数，`req`和`res`，来完成这些工作：

{% code-tabs %}
{% code-tabs-item title="js/rand\_range.ts" %}
```typescript
function req(
  from: number,
  to: number,
): [flatbuffers.Builder, msg.Any, flatbuffers.Offset] {
  // 获取一个builder来创建一个序列化的buffer
  const builder = flatbuffers.createBuilder();
  msg.RandRange.startRandRange(builder);
  // 把数据存入buffer！
  msg.RandRange.addFrom(builder, from);
  msg.RandRange.addTo(builder, to);
  const inner = msg.RandRange.endRandRange(builder);
  // 我们返回这三个信息。
  // dispatch.sendSync/sendAsync将会需要这些参数！
  // （可以把这些操作当做是模板）
  return [builder, msg.Any.RandRange, inner];
}

function res(baseRes: null | msg.Base): number {
  // 一些检查
  assert(baseRes !== null);
  // 确保我们确实得到正确的响应类型
  assert(msg.Any.RandRangeRes === baseRes!.innerType());
  // 创建RandRangeRes模板
  const res = new msg.RandRangeRes();
  // Deserialize!
  // 反序列化！
  assert(baseRes!.inner(res) !== null);
  // 提取出result
  return res.result();
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

非常棒！定义了`req`和`res`，我们可以非常容易的定义实际的同步/异步APIs：

{% code-tabs %}
{% code-tabs-item title="js/rand\_range.ts" %}
```typescript
// 同步
export function randRangeSync(from: number, to: number): number {
  return res(dispatch.sendSync(...req(from, to)));
}

// 异步
export async function randRange(from: number, to: number): Promise<number> {
  return res(await dispatch.sendAsync(...req(from, to)));
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

## 暴露出接口

进入`js/deno.ts`然后增加下面一行：

{% code-tabs %}
{% code-tabs-item title="js/deno.ts" %}
```typescript
export { randRangeSync, randRange } from "./rand_range";
```
{% endcode-tabs-item %}
{% endcode-tabs %}

同样, 进入`BUILD.gn`，然后把我们的文件添加到`ts_sources`：

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

## 增加后台代码

进入，`src/ops.rs`。这是唯一一个Rust后台代码文件，我们需要进行修改来增加这个功能的。

在`ops.rs`，让我们首先导入产生随机数的工具。非常幸运的，Deno已经包含了一个叫做`rand`的Rust crate，他可以用来完成这个任务。可以查阅这个API[here](https://docs.rs/rand/0.6.1/rand/)。

让我们导入他：

{% code-tabs %}
{% code-tabs-item title="src/ops.rs" %}
```rust
use rand::{Rng, thread_rng};
```
{% endcode-tabs-item %}
{% endcode-tabs %}

现在我们可以创建一个函数\(通过一些模板代码\)来完成随机数范围逻辑：

{% code-tabs %}
{% code-tabs-item title="src/ops" %}
```rust
fn op_rand_range(
  _state: &IsolateState,
  base: &msg::Base,
  data: libdeno::deno_buf,
) -> Box<Op> {
  assert_eq!(data.len(), 0);
  // 把消息解码为RandRange
  let inner = base.inner_as_rand_range().unwrap();
  // 获取 command id, 用来响应异步调用。
  let cmd_id = base.cmd_id();
  // 从buffer中获取`from`和`to`
  let from = inner.from();
  let to = inner.to();

  // 把我们的慢代码和响应代码包在这里
  // 基于dispatch.sendSync和dispatch.sendAsync，
  // base.sync()将会是true或false。
  // 如果是true，blocking()将会在主线程产生任务
  // 否则，blocking()将会在Tokio线程池产生任务
  blocking(base.sync(), move || -> OpResult {
    // 真正的随机数生成代码！
    let result = thread_rng().gen_range(from, to);
    
    // 准备响应消息序列化
    // 现在把这些当做是模板化的代码
    let builder = &mut FlatBufferBuilder::new();
    // 将消息类型设置为RandRangeRes
    let inner = msg::RandRangeRes::create(
      builder,
      &msg::RandRangeResArgs {
        result, // 把我们的结果放在这里
      },
    );
    // 把消息序列化
    Ok(serialize_response(
      cmd_id, // 用来回应给TypeScript，如果这是一个异步调用
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

在完成我们的实现后，我们需要告诉Rust我们该何时调用他。

我们注意到在`src/ops.rs`中由一个`dispatch`函数。在这个函数里面，我们可以找到一个对于消息类型的`match`语句来设置对应的处理操作。让我们设置我们的函数作为`msg::Any::RandRange`类型的处理函数，插入下面这一行：

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
        /* 增加下面一行！ */
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

好了！这就是你需要为这个调用添加的所有代码！

运行`./tools/build.py`来编译你的项目！现在你已经能够调用`deno.randRangeSync`和`deno.randRange`。

在终端中尝试运行！

```bash
$ ./target/debug/deno
> deno.randRangeSync(0, 100)
96
> (async () => console.log(await deno.randRange(0, 100)))()
Promise {}
> 74
```

祝贺！

## 增加测试

现在让我们为我们的代码增加基本的测试。

新建一个文件`js/rand_range_test.ts`：

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

把这个测试文件添加到`js/unit_tests.ts`：

{% code-tabs %}
{% code-tabs-item title="js/unit\_tests.ts" %}
```typescript
// 增加这一行
import "./rand_range_test.ts";
```
{% endcode-tabs-item %}
{% endcode-tabs %}

现在，运行`./tools/test.py`。对于所有的测试，你需要确认你的测试是通过的：

```text
test randRangeSync_permW0N0E0R0
... ok
test randRange_permW0N0E0R0
... ok
```

## 提交䘝PR!

让我们假设Deno确实需要这个功能，并且您决心为Deno做出贡献。你可以提交一个PR！

记得运行下面的命令来确保你的代码是可以提交的！

```bash
./tools/format.py
./tools/lint.py
./tools/build.py
./tools/test.py
```



