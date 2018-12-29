# Contribution Guide

## Getting Started

Before reading this document, **please check out the official** [CONTRIBUTING.md](https://github.com/denoland/deno/blob/master/.github/CONTRIBUTING.md)  **by Deno team**. This document aims to be a complement to the above guidelines.

## Test Release Build

To run tests on release \(instead of debug\) build, ensure you have built the release version, and 

```bash
./tools/test.py target/release
```

## Adding a New Third Party Package

Based on the type of the package, you might want to ensure the package is correctly recorded in either `package.json` or `Cargo.toml` . After the third party package is added in `third_party/`, submit a PR to [deno\_third\_party](https://github.com/denoland/deno_third_party). Once the third party PR is present, you will be able to submit PRs in deno repo that references to the corresponding commit and access the packages in CI.

For adding a new Rust crate, you might want to look into `build_extra/rust/BUILD.gn` and adding a new `rust_crate` entry there. You might also need to insert the entry in [main\_extern](https://github.com/denoland/deno/blob/73fb98ce70b327d1236b9540f3839a89c5d22ef0/BUILD.gn#L20-L48) inside of `BUILD.gn`.





