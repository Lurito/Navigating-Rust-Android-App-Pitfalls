# 03 - 构建 Tauri 2 Demo

## 安装 Tauri 命令行工具

如果你还没有安装过 Tauri 的以下两个命令行工具，则需要先安装他们：

``` Bash
cargo install create-tauri-app
cargo install tauri-cli
```

## 创建并初始化项目

我们就先创建一个最简单的 demo 项目来做示例：

``` Bash
# 须替换下面两个变量为实际的安装位置
export ANDROID_HOME="~/android-sdk-linux"
export NDK_HOME="~/android-sdk-linux/ndk/27.2.12479018"

cargo create-tauri-app tauri-android-demo -y
cd tauri-android-demo
cargo tauri android init --skip-targets-install
```

## 修改 Gradle 镜像源

如果用于构建的主机位于中国大陆，建议设置镜像源，否则可能因为镜像拉取失败而导致构建失败。

位于中国大陆以外的构建主机可以跳过这一节。

Linux 系统下，编辑 `~/.gradle/init.gradle` 来设置镜像源：

``` Gradle
allprojects {
    repositories {
        // 国内镜像源
        maven { url 'https://maven.aliyun.com/repository/public/' }
        maven { url 'https://maven.aliyun.com/repository/google/' }
        maven { url 'https://maven.aliyun.com/repository/gradle-plugin/' }
        // 备用官方源
        google()
        mavenCentral()  
    }
}
```

## 生成签名证书

如果已经有签名证书了，例如我们在[第二章](./02-构建Slint-Demo.md)生成过一个，则可以跳过这一步。

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

## 配置 Gradle 使用签名密钥

> 本节的内容参考 [Tauri 官方文档的对应内容](https://v2.tauri.app/distribute/sign/android/#configure-gradle-to-use-the-signing-key) 撰写，你可以参见此链接查看最新的指引。

首先，我们需要在项目根目录下的 `src-tauri/gen/android/` 目录创建一个 `keystore.properties` 文件，包含以下内容（需要将这三个参数根据实际的证书配置修改）：

```
password=mypassword
keyAlias=my-alias
storeFile=my-release-key.pfx
```

> 如果 `storeFile` 的路径按上面这样填写相对路径而不是绝对路径，程序会根据相对于根目录下 `src-tauri/gen/android/` 的路径来查找证书。

接着我们需要编辑项目根目录下的 `src-tauri/gen/android/app/build.gradle.kts` 文件，来配置让 gradle 使用证书来签名你的 APK 文件。

首先我们需要在文件的头部加入引入语句：

``` Kotlin
import java.io.FileInputStream
```

然后找到 `android` 之下的 `buildTypes` 段落，在这之前加入以下内容：

``` Kotlin
signingConfigs {
    create("release") {
        val keystorePropertiesFile = rootProject.file("keystore.properties")
        val keystoreProperties = Properties()
        if (keystorePropertiesFile.exists()) {
            keystoreProperties.load(FileInputStream(keystorePropertiesFile))
        }

        keyAlias = keystoreProperties["keyAlias"] as String
        keyPassword = keystoreProperties["password"] as String
        storeFile = file(keystoreProperties["storeFile"] as String)
        storePassword = keystoreProperties["password"] as String
    }
}
```

最后，我们要在 `buildTypes` 下的 `getByName("release")` 段落中添加以下语句：

``` Kotlin
buildTypes {
    ...
    getByName("release") {
        ...
        signingConfig = signingConfigs.getByName("release")
    }
}
```

## 执行构建并打包为 APK

在终端中运行以下命令，配置环境变量（需要改为实际的安装位置）并构建打包：

``` Bash
# 须替换下面两个变量为实际的安装位置
export ANDROID_HOME="~/android-sdk-linux"
export ANDROID_NDK_ROOT="~/android-sdk-linux/ndk/27.2.12479018"

cargo tauri android build --apk --target aarch64
```

执行完毕后，我们的 APK 就会出现在根目录下的 `src-tauri\gen\android\app\build\outputs\apk\universal\release` 了。
