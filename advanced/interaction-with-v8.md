---
description: >-
  In this section, we will explore how libdeno is used to interact with V8, the
  underlying JavaScript engine of Deno.
---

# Interaction with V8

## Creating V8 Platform

From `src/main.rs`, we would find these 2 lines of code:

{% code-tabs %}
{% code-tabs-item title="src/main.rs" %}
```rust
let snapshot = snapshot::deno_snapshot();
let isolate = isolate::Isolate::new(snapshot, state, ops::dispatch);
```
{% endcode-tabs-item %}
{% endcode-tabs %}

This is where the V8 isolate is created. Going into the definition of `Isolate::new` in `src/isolate.rs`:

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
      recv_cb: pre_dispatch, // callback to invoke when Rust receives a message
    };
    let libdeno_isolate = unsafe { libdeno::deno_new(config) };
    // This channel handles sending async messages back to the runtime.
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

Here, we would find 2 calls on `libdeno`: `deno_init` and `deno_new`. These two functions, unlike many other functions used, are defined from `libdeno/api.cc`, and are made available thanks to FFI and `libc`. Their Rust interface is provided through `src/libdeno.rs`.

Go to `libdeno/api.cc`:

{% code-tabs %}
{% code-tabs-item title="libdeno/api.cc" %}
```c
void deno_init() {
  auto* p = v8::platform::CreateDefaultPlatform();
  v8::V8::InitializePlatform(p);
  v8::V8::Initialize();
}

Deno* deno_new(deno_config config) {
  // ... code omitted
  deno::DenoIsolate* d = new deno::DenoIsolate(config);
  // ... code omitted
  v8::Isolate* isolate = v8::Isolate::New(params);
  // ... code omitted

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
      // If no snapshot is provided, we initialize the context with empty
      // main source code and source maps.
      deno::InitializeContext(isolate, context);
    }
    d->context_.Reset(isolate, context);
  }

  return reinterpret_cast<Deno*>(d);
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

From these two functions, we would find that `deno_init` is used to initialize the V8 platform, while `deno_new` is used to create a new isolated VM instance on this platform. \(A few V8 embedding APIs are used here. To learn more about V8 embedding, check out [https://v8.dev/docs/embed](https://v8.dev/docs/embed) for concepts, and [https://denolib.github.io/v8-docs/](https://denolib.github.io/v8-docs/) for API Reference.\)

## Adding Bindings

There are 2 important functions/constructors used in `deno_new` that might not be immediately clear: `DenoIsolate` and `InitializeContext`. It turns out `DenoIsolate` serves more or less as a collection of Isolate information. Instead, `InitializeContext` is the more interesting one. \(It seems to be invoked here only when there is no snapshot provided. However, you'll also find the function being used in `deno_new_snapshotter` in `libdeno/api.cc` to create a new snapshot, so it is always an inevitable step\):

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

If you are familiar with the `libdeno` API on the TypeScript end, the above names wrapped in `deno::v8_str` might sound familiar to you. In fact, this is where the very few C++ bindings onto JavaScript are attached: we get the global object of current `Context` \(a V8 execution environment that allows separate code to run\) and add some extra properties onto it, from C++.

Based on the code above, whenever you call `libdeno.send(...)` from TypeScript \(you'll find such usage in `sendInternal` of `js/dispatch.ts`\), you are actually calling into a C++ function called `Send`. Similar things happens to `libdeno.print`.

Another example: `console.log(...)` is defined as

{% code-tabs %}
{% code-tabs-item title="js/console.ts" %}
```typescript
  log = (...args: any[]): void => {
    this.printFunc(stringifyArgs(args));
  };
```
{% endcode-tabs-item %}
{% endcode-tabs %}

This `printFunc` is a private field in class `Console`, and its initial value is provided through

{% code-tabs %}
{% code-tabs-item title="js/globals.ts" %}
```typescript
window.console = new consoleTypes.Console(libdeno.print);
```
{% endcode-tabs-item %}
{% endcode-tabs %}

Okay, we find `libdeno.print` here. From the above bindings code, we know that calling to `libdeno.print` from TypeScript is equivalent to calling `Print` in `libdeno/binding.cc`, inside of which we would discover

```cpp
void Print(const v8::FunctionCallbackInfo<v8::Value>& args) {
  // ... code omitted
  v8::String::Utf8Value str(isolate, args[0]);
  bool is_err =
      args.Length() >= 2 ? args[1]->BooleanValue(context).ToChecked() : false;
  const char* cstr = ToCString(str);
  auto& stream = is_err ? std::cerr : std::cout;
  stream << cstr << std::endl;
}
```

Nothing more surprising than some familiar standard C++ print formula. Therefore, calling `console.log` is in fact just indirectly calling `std::cout << cstr << std::endl`!

Check out the source code of `Print`, `Send` and `Recv` in `libdeno/binding.cc` to understand what is happening behind the scene.

## Executing Code on V8

From `src/main.rs`, immediately following isolate creation, we find

{% code-tabs %}
{% code-tabs-item title="src/main.rs" %}
```rust
isolate
      .execute("denoMain();")
      .unwrap_or_else(print_err_and_exit);
```
{% endcode-tabs-item %}
{% endcode-tabs %}

From the previous section, we know `denoMain` is a TypeScript side function. `isolate.execute` is used to run the code. Let's extract its definition from `src/isolate.rs`:

{% code-tabs %}
{% code-tabs-item title="src/isolate.rs" %}
```rust
  /// Same as execute2() but the filename defaults to "<anonymous>".
  pub fn execute(&self, js_source: &str) -> Result<(), JSError> {
    self.execute2("<anonymous>", js_source)
  }

  /// Executes the provided JavaScript source code. The js_filename argument is
  /// provided only for debugging purposes.
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

The only interesting function we care in this section is `libdeno::deno_execute`. We can find its actual definition in `libdeno/api.cc` again:

{% code-tabs %}
{% code-tabs-item title="libdeno/api.cc" %}
```c
int deno_execute(Deno* d_, void* user_data, const char* js_filename,
                 const char* js_source) {
  auto* d = unwrap(d_);
  // ... code omitted
  auto context = d->context_.Get(d->isolate_);
  CHECK(!context.IsEmpty());
  return deno::Execute(context, js_filename, js_source) ? 1 : 0;
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

Eventually, let's find `deno::Execute`:

```cpp
bool Execute(v8::Local<v8::Context> context, const char* js_filename,
             const char* js_source) {
  // ... code omitted
  auto source = v8_str(js_source);
  return ExecuteV8StringSource(context, js_filename, source);
}

bool ExecuteV8StringSource(v8::Local<v8::Context> context,
                           const char* js_filename,
                           v8::Local<v8::String> source) {
  // ... code omitted

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

As we see here, `Execute` is eventually submitting the code to `v8::Script::Compile`, which compiles the JavaScript code and call `Run` on it. Any exceptions, compile time or runtime, are further processed through `HandleException`:

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

It sets `d->last_exception_` to be an error message formatted in JSON, which was read in `deno_last_exception`:

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

which is used in `Isolate::last_exception` from Rust:

{% code-tabs %}
{% code-tabs-item title="src/isolate.rs" %}
```rust
pub fn last_exception(&self) -> Option<JSError> {
    let ptr = unsafe { libdeno::deno_last_exception(self.libdeno_isolate) };
    // ... code omitted
  }
```
{% endcode-tabs-item %}
{% endcode-tabs %}

and is checked inside of `Isolate::execute` to decide if error handling is necessary.

Whew! That's a long trip across TypeScript, C/C++ and Rust. However, it should be very clear to you how Deno is interacting with V8 now. Nothing magical.

