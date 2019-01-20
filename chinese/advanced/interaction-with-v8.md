---
描述: >-
  在这一部分，我们将会探索libdeno是如何用来跟Deno底层的JavaScript引擎V8交互的。
---

# 跟V8的交互

## 创建 V8 Platform

从`src/main.rs`文件，我们可以找到如下两行代码：

{% code-tabs %}
{% code-tabs-item title="src/main.rs" %}
```rust
let snapshot = snapshot::deno_snapshot();
let isolate = isolate::Isolate::new(snapshot, state, ops::dispatch);
```
{% endcode-tabs-item %}
{% endcode-tabs %}

这里是V8 isolate被创建的地方。进入到文件`src/isolate.rs`中`Isolate::new`的定义：

{% code-tabs %}
{% code-tabs-item title="src/isolate.rs" %}
```rust
pub fn new(
    snapshot: libdeno::deno_buf,
    state: Arc<IsolateState>,
    dispatch: Dispatch,
  ) -> Self {
    DENO_INIT.call_once(|| {
      unsafe { libdeno::deno_init() };
    });
    let config = libdeno::deno_config {
      will_snapshot: 0,
      load_snapshot: snapshot,
      shared: libdeno::deno_buf::empty(),
      recv_cb: pre_dispatch, // 当Rust接受到消息，需要执行的回调
    };
    let libdeno_isolate = unsafe { libdeno::deno_new(config) };
    // 这个通道处理发送异步消息回运行时
    let (tx, rx) = mpsc::channel::<(i32, Buf)>();

    Self {
      libdeno_isolate,
      dispatch,
      rx,
      tx,
      ntasks: Cell::new(0),
      timeout_due: Cell::new(None),
      state,
    }
  }
```
{% endcode-tabs-item %}
{% endcode-tabs %}

在这里，我们找到了`libdeno`上的两个调用：`deno_init`和`deno_new`。跟其他的函数不同，这两个函数是在文件`libdeno/api.cc`中定义的，感谢FFI 和 `libc`，Rust中可以调用这两个C/C++函数。他们的Rust接口是由`src/libdeno.rs`提供的。

进入`libdeno/api.cc`文件：

{% code-tabs %}
{% code-tabs-item title="libdeno/api.cc" %}
```c
void deno_init() {
  auto* p = v8::platform::CreateDefaultPlatform();
  v8::V8::InitializePlatform(p);
  v8::V8::Initialize();
}

Deno* deno_new(deno_config config) {
  // ... 忽略的代码
  deno::DenoIsolate* d = new deno::DenoIsolate(config);
  // ... 忽略的代码
  v8::Isolate* isolate = v8::Isolate::New(params);
  // ... 忽略的代码

  v8::Locker locker(isolate);
  v8::Isolate::Scope isolate_scope(isolate);
  {
    v8::HandleScope handle_scope(isolate);
    auto context =
        v8::Context::New(isolate, nullptr, v8::MaybeLocal<v8::ObjectTemplate>(),
                         v8::MaybeLocal<v8::Value>(),
                         v8::DeserializeInternalFieldsCallback(
                             deno::DeserializeInternalFields, nullptr));
    if (!config.load_snapshot.data_ptr) {
      // 如果没有提供snapshot，我们将会用空的main source和source map初始化context
      deno::InitializeContext(isolate, context);
    }
    d->context_.Reset(isolate, context);
  }

  return reinterpret_cast<Deno*>(d);
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

从这两个函数，我们可以发现，`deno_init`是用来初始化V8 platform，而`deno_new`是用来在该platform上创建一个新的独立的虚拟机实例的。\(一些V8嵌入APIs在这里被使用。为了了解更多的V8嵌入，查看[https://v8.dev/docs/embed](https://v8.dev/docs/embed)相关概念和[https://denolib.github.io/v8-docs/](https://denolib.github.io/v8-docs/)API 手册\)

## 添加扩展

有两个函数/构造函数`DenoIsolate`和`InitializeContext`在`deno_new`中被使用，可能目前还不清楚。或多或少的，`DenoIsolate`是用来收集Isolate的信息的。而`InitializeContext`是更加有意思的一个。\(好像没有snapshot被提供他才会在这里被调用。但是，你也会发现`libdeno/api.cc`中的`deno_new_snapshotter`函数会被调用来生成一个新的snapshot，所以`InitializeContext`总是会被调用的\)

{% code-tabs %}
{% code-tabs-item title="libdeno/binding.cc" %}
```c
void InitializeContext(v8::Isolate* isolate, v8::Local<v8::Context> context) {
  v8::HandleScope handle_scope(isolate);
  v8::Context::Scope context_scope(context);

  auto global = context->Global();

  auto deno_val = v8::Object::New(isolate);
  CHECK(global->Set(context, deno::v8_str("libdeno"), deno_val).FromJust());

  auto print_tmpl = v8::FunctionTemplate::New(isolate, Print);
  auto print_val = print_tmpl->GetFunction(context).ToLocalChecked();
  CHECK(deno_val->Set(context, deno::v8_str("print"), print_val).FromJust());

  auto recv_tmpl = v8::FunctionTemplate::New(isolate, Recv);
  auto recv_val = recv_tmpl->GetFunction(context).ToLocalChecked();
  CHECK(deno_val->Set(context, deno::v8_str("recv"), recv_val).FromJust());

  auto send_tmpl = v8::FunctionTemplate::New(isolate, Send);
  auto send_val = send_tmpl->GetFunction(context).ToLocalChecked();
  CHECK(deno_val->Set(context, deno::v8_str("send"), send_val).FromJust());

  CHECK(deno_val->SetAccessor(context, deno::v8_str("shared"), Shared)
            .FromJust());
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

如果你对TypeScript端的`libdeno`API已经不陌生了，上面的被`deno::v8_str`包裹的名字你可能也不会陌生。事实上，这里是一些V8的C++扩展被绑定到JavaScript的地方，我们获取当前`Context`\(一个运行分隔的代码的V8的执行环境\)的全局变量，然后从C++添加一些额外的属性到全局变量。

基于上面的代码，当你从TypeScript调用`libdeno.send(...)`\(你会在`js/dispatch.ts`文件的`sendInternal`函数找到该调用\)，你实际上是调用了`Send`这个C++函数。对于`libdeno.print`调用也是同样的。

另一个例子：`console.log(...)`定义如下：

{% code-tabs %}
{% code-tabs-item title="js/console.ts" %}
```typescript
  log = (...args: any[]): void => {
    this.printFunc(stringifyArgs(args));
  };
```
{% endcode-tabs-item %}
{% endcode-tabs %}

`printFunc`是`Console`类的一个私有的属性，他的初始化值是这样的：

{% code-tabs %}
{% code-tabs-item title="js/globals.ts" %}
```typescript
window.console = new consoleTypes.Console(libdeno.print);
```
{% endcode-tabs-item %}
{% endcode-tabs %}

ok，我们在这里找到了`libdeno.print`。从上面的扩展代码中，我们知道从TypeScript调用`libdeno.print`实际上将会调用`libdeno/binding.cc`中的`Print`。

```cpp
void Print(const v8::FunctionCallbackInfo<v8::Value>& args) {
  // ... 忽略的代码
  v8::String::Utf8Value str(isolate, args[0]);
  bool is_err =
      args.Length() >= 2 ? args[1]->BooleanValue(context).ToChecked() : false;
  FILE* file = is_err ? stderr : stdout;
  fwrite(*str, sizeof(**str), str.length(), file);
  fprintf(file, "\n");
  fflush(file);
}
```

一点也不惊讶，跟C语言的print是一样的。因此，调用`console.log`最终也是间接调用了`fwrite/fprintf`！

查阅`libdeno/binding.cc`中的`Print`，`Send`和`Recv`来了解更多发生在底层的细节。 

## 在V8中执行代码

从`src/main.rs`文件中，紧接着isolate的创建，我们发现

{% code-tabs %}
{% code-tabs-item title="src/main.rs" %}
```rust
isolate
      .execute("denoMain();")
      .unwrap_or_else(print_err_and_exit);
```
{% endcode-tabs-item %}
{% endcode-tabs %}

从前面的部分，我们知道`denoMain`是一个TypeScript端的函数。`isolate.execute`被用来执行代码。让我们提取出他在`src/isolate.rs`文件中的定义：

{% code-tabs %}
{% code-tabs-item title="src/isolate.rs" %}
```rust
  /// 跟execute2()一样， 但是文件名默认实际是 "<anonymous>"。
  pub fn execute(&self, js_source: &str) -> Result<(), JSError> {
    self.execute2("<anonymous>", js_source)
  }

  /// 执行提供的JavaScript源码。
  /// js_filename参数仅仅当调试时提供。
  pub fn execute2(
    &self,
    js_filename: &str,
    js_source: &str,
  ) -> Result<(), JSError> {
    let filename = CString::new(js_filename).unwrap();
    let source = CString::new(js_source).unwrap();
    let r = unsafe {
      libdeno::deno_execute(
        self.libdeno_isolate,
        self.as_raw_ptr(),
        filename.as_ptr(),
        source.as_ptr(),
      )
    };
    if r == 0 {
      let js_error = self.last_exception().unwrap();
      return Err(js_error);
    }
    Ok(())
  }
```
{% endcode-tabs-item %}
{% endcode-tabs %}

我们最感兴趣的部分是`libdeno::deno_execute`。我们可以再次在`libdeno/api.cc`找到他的实际定义：

{% code-tabs %}
{% code-tabs-item title="libdeno/api.cc" %}
```c
int deno_execute(Deno* d_, void* user_data, const char* js_filename,
                 const char* js_source) {
  auto* d = unwrap(d_);
  // ... 忽略的代码
  auto context = d->context_.Get(d->isolate_);
  CHECK(!context.IsEmpty());
  return deno::Execute(context, js_filename, js_source) ? 1 : 0;
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

最后，我们找到`deno::Execute`：

```cpp
bool Execute(v8::Local<v8::Context> context, const char* js_filename,
             const char* js_source) {
  // ... 忽略的代码
  auto source = v8_str(js_source);
  return ExecuteV8StringSource(context, js_filename, source);
}

bool ExecuteV8StringSource(v8::Local<v8::Context> context,
                           const char* js_filename,
                           v8::Local<v8::String> source) {
  // ... 忽略的代码

  v8::TryCatch try_catch(isolate);

  auto name = v8_str(js_filename);

  v8::ScriptOrigin origin(name);

  auto script = v8::Script::Compile(context, source, &origin);

  if (script.IsEmpty()) {
    DCHECK(try_catch.HasCaught());
    HandleException(context, try_catch.Exception());
    return false;
  }

  auto result = script.ToLocalChecked()->Run(context);

  if (result.IsEmpty()) {
    DCHECK(try_catch.HasCaught());
    HandleException(context, try_catch.Exception());
    return false;
  }

  return true;
}
```

正如我们看到的，`Execute`最终把代码提交给了`v8::Script::Compile`，`v8::Script::Compile`编译了JavaScript代码然后调用他的`Run`方法。编译或者运行时，如果任何异常发生，都会被传递给`HandleException`：

{% code-tabs %}
{% code-tabs-item title="libdeno/binding.cc" %}
```cpp
void HandleException(v8::Local<v8::Context> context,
                     v8::Local<v8::Value> exception) {
  v8::Isolate* isolate = context->GetIsolate();
  DenoIsolate* d = FromIsolate(isolate);
  std::string json_str = EncodeExceptionAsJSON(context, exception);
  CHECK(d != nullptr);
  d->last_exception_ = json_str;
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

他设置了`d->last_exception_`为一个JSON格式的错误信息，该消息会被`deno_last_exception`读取：


{% code-tabs %}
{% code-tabs-item title="libdeno/api.cc" %}
```c
const char* deno_last_exception(Deno* d_) {
  auto* d = unwrap(d_);
  if (d->last_exception_.length() > 0) {
    return d->last_exception_.c_str();
  } else {
    return nullptr;
  }
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

Rust中会在`Isolate::last_exception`使用该错误信息：

{% code-tabs %}
{% code-tabs-item title="src/isolate.rs" %}
```rust
pub fn last_exception(&self) -> Option<JSError> {
    let ptr = unsafe { libdeno::deno_last_exception(self.libdeno_isolate) };
    // ... 忽略的代码
  }
```
{% endcode-tabs-item %}
{% endcode-tabs %}

并且在`Isolate::execute`内会检查是否错误处理是必需的。

哟！在TypeScript，C/C++和Rust之间我们经历了一个漫长的旅行。现在你应该非常清楚Deno是如何跟V8交互的了。其实没什么特别的！
