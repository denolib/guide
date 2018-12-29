# Installing Deno

## Get Deno Release Binary

Deno is still at very early development stage. The current releases are 0.2.x, which are defined to be "mildly usable" and is mostly for developer preview.

To install Deno release binary, run the following commands on \*nix

```bash
curl -L https://deno.land/x/install/install.py | python
export PATH=$HOME/.deno/bin:$PATH
```

On windows, you might want to install through Powershell:

```text
iex (iwr https://deno.land/x/install/install.ps1)
```

Deno is installed through scripts from [deno\_install](https://github.com/denoland/deno_install). If you encountered any installation problems, submit an issue there.

## Compile Deno From Source

If you are interested in contributing to Deno, you might want to compile Deno from source yourself.

Run the following commands to get make a debug build

```bash
# Fetch deps.
git clone --recurse-submodules https://github.com/denoland/deno.git
cd deno
# Setup and download important third party tools
./tools/setup.py
# Build.
./tools/build.py
```

{% hint style="info" %}
Notice that in some countries, domains, such as google.com, are blocked from access. You might want to use a VPN or alternative tools when running `./tools/setup.py`.
{% endhint %}

Deno is built with GN and Ninja, tools that are used also by the Chromium team. The built files will be located at `target/debug/`

Alternatively, to create a release build for evaluation, set DENO\_BUILD\_MODE to release on build:

```bash
DENO_BUILD_MODE=release ./tools/build.py
```

The built files will be located at `target/release/`

Deno also comes with Cargo build support:

```bash
cargo build
```



