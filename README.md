# 基于Kuikly的鸿蒙平台的Serialization能力

* 只支持 HarmonyOS Native平台。
* 提供类注解 `@Serializable` 并且支持标准集合类型。
* 只支持Json格式。

## 背景

[Kuikly跨端KMP无法直接使用官方 KMP 库，需要适配。](https://github.com/Tencent-TDS/KuiklyUI/issues/187)

## 准备

阅读[KMP模块鸿蒙 Kotlin/Native 适配](https://kuikly.tds.qq.com/%E5%BC%80%E5%8F%91%E6%96%87%E6%A1%A3/kuiklybase-ohos-kn.html)

### Gradle

必须做两件事：

1) 添加 **[插件](#1-setting-up-the-serialization-plugin)**.
2) 添加 **[依赖库](#2-dependency-on-the-json-library)**.

#### 1) 设置插件

设置的「kotlinx-serialization」插件，其版本必须对应 Kuikly Kotlin
定制化版本。例如：[2.0.21-KBA-004](https://kuikly.tds.qq.com/%E5%BC%80%E5%8F%91%E6%96%87%E6%A1%A3/kuiklybase-ohos-kn.html#%E4%BD%BF%E7%94%A8%E5%AE%9A%E5%88%B6%E5%8C%96kotlin%E7%89%88%E6%9C%AC)

Kotlin DSL:

```kotlin
plugins {
    kotlin("multiplatform") version "2.0.21-KBA-004"
    kotlin("plugin.serialization") version "2.0.21-KBA-004"
}
```

#### 2) 添加依赖库

在设置插件之后，添加「kotlinx-serialization」依赖。

Kotlin DSL:

```kotlin
repositories {
    maven {
        // 用于拉取 Kuikly 定制的 kotlinx-serialization 私仓依赖
        url = uri("https://maven-central.<#私仓地址#>")
    }
    maven {
        // 用于拉取 Kotlin Kuikly 定制版本依赖
        url = uri("https://mirrors.tencent.com/nexus/repository/maven-public/")
    }
}

dependencies {

    implementation("<#自定义 group#>:kotlinx-serialization-core-ohosarm64::<#基于1.5.1fork 的递增版本号#>")
    implementation("<#自定义 group#>:kotlinx-serialization-json-ohosarm64::<#基于1.5.1fork 的递增版本号#>") {
        exclude(group = "com.zhaopin.kotlinx", module = "kotlinx-serialization-core")
    }
}
```

## 使用示例

```kotlin
import kotlinx.serialization.*
import kotlinx.serialization.json.*

@Serializable
data class Project(val name: String, val language: String)

fun main() {
    // Serializing objects
    val data = Project("kotlinx.serialization", "Kotlin")
    val string = Json.encodeToString(data)
    println(string) // {"name":"kotlinx.serialization","language":"Kotlin"} 
    // Deserializing back into objects
    val obj = Json.decodeFromString<Project>(string)
    println(obj) // Project(name=kotlinx.serialization, language=Kotlin)
}
```

## 适配鸿蒙平台的具体步骤

* 根据Android项目Kotlin版本选择「kotlinx-serialization」对应fork的tag分支
    * 例如：我们项目使用Gradle 7.4，Kotlin 1.8.22，JDK 17，则选择[「kotlinx-serialization」的1.5.1分支的tag](https://github.com/Kotlin/kotlinx.serialization/releases/tag/v1.5.1)
* 修改「group」和「version」，防止与官方冲突
* 按照[Kuikly文档](https://kuikly.tds.qq.com/%E5%BC%80%E5%8F%91%E6%96%87%E6%A1%A3/kuiklybase-ohos-kn.html)操作
    * 添加 Kuikly Maven 源
    * 设置依赖插件的版本对应 Kuikly Kotlin 版本，例如：kotlin("multiplatform").version("2.0.21-KBA-004")
    * 通过Kuikly Example 编译HarmonyOS产物日志得到信息：
        * HarmonyOS编译目标：ohos_arm64(HarmonyOS的 64位 Arm 架构)
        * 编译工具：「konanc」编译器（Kotlin/Native），使用 clang++和 ld.lld进行本地代码链接
        * Kotlin版本：kotlin-native-prebuilt-macos-x86_64-2.0.21-KBA-004
        * 产物：源于 Kotlin/Native自动生成 C++代码（api.cpp)
    * 分析：通过通过构建日志信息可以认为「ohos_arm64」是 Kotlin/Native支持的一个目标平台，可以将其平台认为是一个原生（Native）目标，与 iOS、macOS、Linux 同级。
    * 适配步骤：
        * 在 KMP 模块「build.gradle.kt」添加鸿蒙平台「ohosArm64」编译目标;
        * 在 KMP 模块中添加「ohosArm64main」鸿蒙平台目录，并且实现「actual fun」，可以复制源代码中的「NativeMain」的实现。
        * 发布到私仓

## 发布脚本
publish_release.sh:
```shell
#!/bin/bash

set -e 

./gradlew :kotlinx-serialization-core:publishAllPublicationsToMavenRepository
./gradlew :kotlinx-serialization-bom:publishAllPublicationsToMavenRepository
./gradlew :kotlinx-serialization-json:publishAllPublicationsToMavenRepository
./gradlew :kotlinx-serialization-cbor:publishAllPublicationsToMavenRepository
./gradlew :kotlinx-serialization-protobuf:publishAllPublicationsToMavenRepository
./gradlew :kotlinx-serialization-properties:publishAllPublicationsToMavenRepository
./gradlew :kotlinx-serialization-hocon:publishAllPublicationsToMavenRepository

CURRENT_VERSION=$(grep "^version=" gradle.properties | cut -d'=' -f2)
echo "- 版本: $CURRENT_VERSION"
```