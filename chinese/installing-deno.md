# 安装 Deno

## 获取Deno的release二进制文件

Deno现在还处在非常初级的发展阶段。当前的release版本是0.2.x，奉行“适度可用”原则，并且主要是给开发者预览的。

如果想要安装Deno的release二进制文件，需要在\*nix系统运行下面的命令

```bash
curl -L https://deno.land/x/install/install.py | python
export PATH=$HOME/.deno/bin:$PATH
```

在windows系统上，你可能需要用Powershell来安装：

```text
iex (iwr https://deno.land/x/install/install.ps1)
```

Deno是通过脚本[deno\_install](https://github.com/denoland/deno_install)安装的。如果你在安装过程
中遇到问题，请在那里提issue。

## 用源码编译Deno

如果你对贡献源码给Deno非常感兴趣，那么你可能想要从源码编译Deno。

运行下面的命令来编译生成一个debug版本

```bash
# 获取源码.
git clone --recurse-submodules https://github.com/denoland/deno.git
cd deno
# 设置和安装第三方依赖
./tools/setup.py
# 构建.
./tools/build.py
```

{% hint style="info" %}
需要注意的是，有些域名，比如google.com，是被墙的。当运行`./tools/setup.py`时，你可能需要用VPN代理或者其他替代工具。
{% endhint %}

Deno是用GN和Ninja工具构建的，这两个工具也是Chromium团队的构建工具。生成的构建文件放在了`target/debug/`目录。

另外，如果想要构建一个release版本，在构建的时候，你需要设置你的环境DENO\_BUILD\_MODE为release：

```bash
DENO_BUILD_MODE=release ./tools/build.py
```

生成的构建文件将会存放在`target/release/`目录

Deno 也可以用Cargo进行构建:

```bash
cargo build
```

