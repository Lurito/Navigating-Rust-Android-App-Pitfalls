# 02 - 构建 Slint Demo

接下来我们就用 [Slint](https://slint.dev/) 来构建一个安卓 demo 程序。

## 拉取仓库

首先，将 Slint 的仓库拉取到本地，并进入 `Weather Demo` 示例的目录：

``` Bash
git clone https://github.com/slint-ui/slint.git
cd slint/demos/weather-demo
```

## 安装 cargo-apk

我们需要安装 [`cargo-apk`](https://crates.io/crates/cargo-apk) 作为构建和打包工具：

``` Bash
cargo install cargo-apk
```

尽管 `cargo-apk` 在自己的主页上被标为弃用，且建议使用 [`xbuild`](https://crates.io/crates/xbuild) 作为替代。但 `xbuild` 在撰文时已经两年没有更新过，且可配置项远少于 `cargo-apk`，甚至在 crates.io 上连一个 `README.md` 主页都没有，实在不像一个可靠的工具项目。因此本文仍然建议用 `cargo-apk` 来构建和打包。

## 配置 Android SDK 版本

如果我们安装的 Android SDK 版本是 30 以外的版本，我们需要编辑当前目录下的 `Cargo.toml`，在文件末尾添加以下内容：

``` toml
[package.metadata.android.sdk]
target_sdk_version = 35 # 撰文时的最新版本，须替换为实际安装的 Android SDK 版本
min_sdk_version = 18 # Android 4.3，须替换为实际的最低可安装版本
```

由于作者并不了解安卓系统，因此这里只做简要介绍。更多信息可以查阅 [安卓官网的对应指南](https://developer.android.com/guide/topics/manifest/uses-sdk-element) 和 [cargo-apk 的主页](https://crates.io/crates/cargo-apk)。

## 配置签名证书

安卓 APK 需要签名才能安装，我们需要先生成一个 X.509 标准、PKS12 格式、ECDSA 算法的证书来签名。打开终端输入以下命令：

``` Bash
# 定义变量：名称、别名、密码和文件名
# 建议根据实际情况修改这些变量
CN="My App"
ALIAS="my-alias"
PASSWORD="mypassword"
PFX_NAME="my-release-key.pfx"

# 生成 ECDSA 私钥（P-256 曲线）
openssl ecparam -name prime256v1 -genkey -noout -out ecdsa.key
# 创建自签名证书（有效期 30 年）
openssl req -x509 -key ecdsa.key -out ecdsa.crt -days 10957 -subj "/CN=$CN"
# 导出为 PKCS12 格式
openssl pkcs12 -export -in ecdsa.crt -inkey ecdsa.key -name "$ALIAS" -out "$PFX_NAME" -password pass:"$PASSWORD"

# 删除输出的中间文件
rm ecdsa.crt ecdsa.key
```

> 以上脚本在 Windows Git Bash 环境中会因为触发一个 bug 而报错：`"/CN=$CN"` 会被识别为路径而被替换为 Git Bash 的程序目录。
>
> 在 WSL 和其他环境中可以正常运行。

⚠️ 注意：你需要妥善保管你的证书。安卓系统只允许同样证书签名的安装包作为更新包。也就是说，一旦你遗失了之前的证书，你的用户将无法更新应用程序，除非卸载重新安装。

生成证书后，需要编辑 `Cargo.toml`，在文件末尾添加证书配置：

``` toml
[package.metadata.android.signing.dev]
path = "my-release-key.pfx"      # 须替换为实际文件名
keystore_password = "mypassword" # 须替换为实际的密码

[package.metadata.android.signing.release]
path = "my-release-key.pfx"      # 须替换为实际文件名
keystore_password = "mypassword" # 须替换为实际的密码
```

## 执行构建并打包为 APK

在终端中运行以下命令，配置环境变量（需要改为实际的安装位置）并构建打包：

``` Bash
# 须替换下面两个变量为实际的安装位置
export ANDROID_HOME="~/android-sdk-linux"
export ANDROID_NDK_ROOT="~/android-sdk-linux/ndk/27.2.12479018"

cargo apk build --lib --release
```

> 中国大陆开发者最好能在有代理的情况下构建，因为构建过程中会自动从 Github 拉取 skia。缺少网络代理可能会导致构建速度缓慢，甚至构建失败。

构建可能需要一点时间……待执行完成后，我们的第一个 APK 就出现在 `../../target/debug/apk/weather-demo.apk` 了。
