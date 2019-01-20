# 源码贡献指南

## 开始

在开始阅读该文档前，**请先查看官方的文档** [CONTRIBUTING.md](https://github.com/denoland/deno/blob/master/.github/CONTRIBUTING.md) **Deno团队出品**。本文档主要是做为官方文档的一个补充。

## 测试release版本

对于release版本\(非debug版本\)的测试，请先确保你构建的是release版本，然后

```bash
./tools/test.py target/release
```

## 新增一个第三方包

根据包类型的不同，你需要确保正确的在`package.json` 或 `Cargo.toml`文件中新增包。在`third_party/`目录新增一个包后，请提交一个PR到[deno\_third\_party](https://github.com/denoland/deno_third_party)。一旦第三方包的PR被接收后，你才可以在Deno的仓库中提交PR，Deno中的PR对应你在第三方包仓库的提交并且会从CI工具中获取该包。


如果你提交的是Rust crate，你需要在`build_extra/rust/BUILD.gn`文件中新增一个`rust_crate`项。你也需要在`BUILD.gn`文件的[main\_extern](https://github.com/denoland/deno/blob/73fb98ce70b327d1236b9540f3839a89c5d22ef0/BUILD.gn#L20-L48)中插入该项。




