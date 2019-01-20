# 源码一览

为了不让你感到迷茫，在这一部分中，我们将会探索当前Deno仓库的源码结构。

代码仓库的链接：[https://github.com/denoland/deno](https://github.com/denoland/deno)

我们将会列举出你可能会使用到的一些重要的目录或文件，然后简短的介绍他们：

```text
#  目录
build_extra/: 一些定制化的Deno构建规则
js/: TypeScript前台
libdeno/: C/C++中台
src/: Rust后台
tests/: 集成测试
third_party/: 第三方代码
tools/: 用来测试和构建Deno的工具

# 文件
BUILD.gn: 主要的Deno构建规则
Cargo.toml: Rust后台的一些依赖信息
package.json: Node依赖(主要是TypeScript的编译器)
build.rs: Cargo构建规则
```

## build\_extra/

一些定制化的构建Deno的规则，当前是由FlatBuffers和Rust crates的构建规则组成。

你可能需要特别关注的一个文件是`build_extra/rust/BUILD.gn`。当在`third_party/`中添加一个新的Rust crate，我们需要在这个文件添加一个项。例如，我们添加了一个构建`time`crate的规则：

{% code-tabs %}
{% code-tabs-item title="build\_extra/rust/BUILD.gn" %}
```
rust_crate("time") {
  # carate存放在哪里
  source_root = "$registry_github/time-0.1.40/src/lib.rs"
  # 他的依赖是什么？
  extern = [
    ":libc",
    ":winapi",
  ]
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

\(`rust_crate` GN 模板 是定义在 `build_extra/rust/rust.gni`\)

## js/

TypeScript前台代码存放的目录。大多数的APIs被定义在他们的文件，同时有一个关联的测试文件， `readFile` 被定义在 `read_file.ts`, 同时，他的单元测试代码是在 `read_file_test.ts`.

除了定义API的文件，还有其他一些特殊的文件：

1. `main.ts`: `denoMain` 被定义在这个文件, Rust调用这个方法作为启动TypeScript代码的入口。
2. `globals.ts`: 所有不需要imports的全局变量收归在这个文件。
3. `deno.ts`: All `deno` 命名空间的APIs从这里被导出。
4. `libdeno.ts`: 暴露给TypeScript的定制化的V8 API的类型定义。
5. `unit_tests.ts`: 所有的单元测试从这里被导入并且被执行。
6. `compiler.ts`: 为Deno定制化的TypeScript编译逻辑。

## libdeno/

C/C++ libdeno中台代码存放在这里

1. `api.cc`: 暴露给Rust的libdeno APIs
2. `binding.cc`: 处理跟V8之间的交互并且绑定V8 C++扩展。这些APIs被暴露给了Rust和TypeScript。
3. `snapshot_creator.cc`: 构建的时候创建V8 snapshot的逻辑。

## src/

Rust后端代码存放在这里

1. `libdeno.rs`: 暴露`libdeno/api.cc`的libdeno APIs给Rust。
2. `isolate.rs`: 获取V8 isolate \(一个V8引擎的isolated实例\)和使用了libdeno APIs的事件循环创建逻辑。
3. `deno_dir.rs`: Deno模块解析和缓存的逻辑。
4. `ops.rs`: 处理Deno Op \(operation\)。真正的文件系统和网络访问是在这里完成的。
5. `main.rs`: 包含了Rust的main函数。当你允许Deno 二进制文件，这里就是**真正的**入口。
6. `msg.fbs`: FlatBuffers消息结构的定义文件， 被用来进行消息传递。 当你需要添加或者修改消息结构的时候，你需要修改该文件。

## tests/

Deno的集成测试

`*.js`或 `*.ts` 文件定义了每个测试样例的代码。`*.test`定义了这个测试样例该如何执行。通常的，`*.out`文件定义了一个测试样例的正常输出\(具体的文件名是在`*.test`文件中指定的\)。

请查阅[tests/README.md](https://github.com/denoland/deno/blob/master/tests/README.md)来获得更多的细节。

## third\_party/

Deno的第三方代码库。包含了node模块，v8，Rust crate代码等等。目录的位置是[https://github.com/denoland/deno\_third\_party](https://github.com/denoland/deno_third_party)。

## tools/

用来执行一系列任务的大部分的Python脚本。

1. `build.py`: 构建Deno
2. `setup.py`: 设置构建环境和下载必须的代码
3. `format.py`: 代码格式化
4. `lint.py`: 代码提示
5. `sync_third_party.py`: 在用户修改`package.json`, `Cargo.toml`等文件之后，同步 third\_party 代码

## BUILD.gn

包含了Deno最重要的构建逻辑。在接下来的为Deno添加新的调用接口的例子中，我们将会讲解到该文件。

## build.rs

定义了`cargo build`的逻辑。内部调用的是`BUILD.gn`。

