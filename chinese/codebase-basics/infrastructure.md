# Deno基础架构

Deno主要由三个独立的部分构成：

## 1. TypeScript前台

那些没有直接的系统调用的公开的接口，APIs和大多数的函数功能，是由TypeScript前台实现的。当前使用了[https://deno.land/typedoc](https://deno.land/typedoc)工具来自动的生成TypeScript API文档\(后面可能回改用其他的工具\)。

TypeScript/JavaScript被当做“没有特权方”，默认的，他们没有文件系统很网络的访问权限\(因为他们是运行在v8沙箱环境的\)。如果想要进行文件和网络访问，他们需要跟“特权方”，即rust后台进行通信。


因此许多的Deno APIs\(特别是系统调用\)是TypeScript前台通过创建buffers数据，然后发送给`libdeno`中台插件，然后等待\(同步或者异步\)结果回传回来。

## 2. C++ libdeno中台

`libdeno`是处于TypeScript前台和Rust后台直接比较小的一层，他用来跟v8进行通信并且暴露出一些必须的插件。他是由C/C++实现的。


`libdeno`暴露了在TypeScript和Rust之间发送和接受消息的传递接口。他也用来初始化v8 platforms/isolates 和创建或加载V8 snapshot\(用了保存序列化得堆信息的数据块。详细请参考[https://v8.dev/blog/custom-startup-snapshots](https://v8.dev/blog/custom-startup-snapshots)\)。值得注意的是，Deno的snapshot保护了一个完整的TypeScript编译器。这很大的缩短了编译器启动时间。

## 3. Rust后台

当前，有文件系统，网络和环境访问权限的后端或者“特权方”，是由Rust实现的。

对于不太熟悉Rust这门语言的人来说，Rust是由Mozilla开发出来的一门系统编程语言，具备内存和并发安全的特性。他也在[Servo](https://servo.org/)项目中被使用。

在2018年6月，Deno原形后端的技术选型是Go语言，后面从Go迁移到了Rust。主要原因是双GC。具体详情请查看[this thread](https://github.com/denoland/deno/issues/205)。

