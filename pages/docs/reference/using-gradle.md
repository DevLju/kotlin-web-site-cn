---
type: doc
layout: reference
title: "使用 Gradle"
---

# 使用 Gradle

In order to build Kotlin with Gradle you should [set up the *kotlin-gradle* plugin](#插件和版本), [apply it](#targeting-the-jvm) to your project and [add *kotlin-stdlib* dependencies](#配置依赖). Those actions may also be performed automatically in IntelliJ IDEA by invoking the Tools \| Kotlin \| Configure Kotlin in Project action.

You can also enable [incremental compilation](#incremental-compilation) to make your builds faster. 

## 插件和版本

使用 *kotlin-gradle-plugin* 编译Kotlin的源代码和模块.

要用的 Kotlin 版本通常是通过 *kotlin.version*属性来定义:

``` groovy
buildscript {
   ext.kotlin_version = '<version to use>'

   repositories {
     mavenCentral()
   }

   dependencies {
     classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"
   }
}
```

This is not required when using Kotlin Gradle plugin 1.1.1 and above with the [Gradle plugins DSL](https://docs.gradle.org/current/userguide/plugins.html#sec:plugins_block).

## Targeting the JVM
## 应用于JVM

为了在JVM中应用, Kotlin插件需要配置如下：

``` groovy
apply plugin: "kotlin"
```

Or, starting with Kotlin 1.1.1, the plugin can be applied using the [Gradle plugins DSL](https://docs.gradle.org/current/userguide/plugins.html#sec:plugins_block):

```groovy
plugins {
  id "org.jetbrains.kotlin.jvm" version "<version to use>"
}
```
The `version` should be literal in this block, and it cannot be applied from another build script.

Kotlin源文件和Java源文件可以在同一个文件夹中存在, 也可以在不同文件夹中。默认采用的是不同的文件夹：

``` groovy
project
    - src
        - main (root)
            - kotlin
            - java
```

如果不想使用默认选项，你需要更新对应的 *sourceSets* 属性

``` groovy
sourceSets {
    main.kotlin.srcDirs += 'src/main/myKotlin'
    main.java.srcDirs += 'src/main/myJava'
}
```

## 应用于 JavaScript

当应用于 JavaScript 的时候, 需要设置一个不同的插件:

``` groovy
apply plugin: "kotlin2js"
```

该插件仅作用于Kotlin文件，因此推荐使用这个插件来区分Kotlin和Java文件 (这种情况仅仅是同一工程中包含Java源文件的时候). 如果
不使用默认选项，又为了应用于 JVM，我们需要指定源文件夹使用 *sourceSets*

``` groovy
sourceSets {
    main.kotlin.srcDirs += 'src/main/myKotlin'
}
```

如果你想创建一个可重用的库, 使用 `kotlinOptions.metaInfo` 来生成额外的二进制形式的JS文件.
这个文件应该和编译结果一起分发.

``` groovy
compileKotlin2Js {
	kotlinOptions.metaInfo = true
}
```


## 应用于 Android

Android的 Gradle模型和传统的Gradle有些不同, 因此如果我们想要通过Kotlin来创建一个Android应用, 应该使用
*kotlin-android* 插件来代替 *kotlin*:

``` groovy
buildscript {
    ...
}
apply plugin: 'com.android.application'
apply plugin: 'kotlin-android'
```

### Android Studio

如果你使用的是Android Studio, 下面的一些属性需要添加到文件中:

``` groovy
android {
  ...

  sourceSets {
    main.java.srcDirs += 'src/main/kotlin'
  }
}
```

上述属性可以使kotlin目录在Android Studio中作为源码根目录存在, 所以当项目模型加载到IDE可以被正确识别. Alternatively, you can put Kotlin classes in the Java source directory, typically located in `src/main/java`.



## 配置依赖

In addition to the kotlin-gradle-plugin dependency shown above, you need to add a dependency on the Kotlin standard library:

``` groovy
buildscript {
    ext.kotlin_version = '<version to use>'
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"
    }
}

apply plugin: "kotlin" // or apply plugin: "kotlin2js" if targeting JavaScript

repositories {
    mavenCentral()
}

dependencies {
    compile "org.jetbrains.kotlin:kotlin-stdlib:$kotlin_version"
}
```

If your project uses [Kotlin reflection](/api/latest/jvm/stdlib/kotlin.reflect.full/index.html) or testing facilities, you need to add the corresponding dependencies as well:

``` groovy
compile "org.jetbrains.kotlin:kotlin-reflect:$kotlin_version"
testCompile "org.jetbrains.kotlin:kotlin-test:$kotlin_version"
testCompile "org.jetbrains.kotlin:kotlin-test-junit:$kotlin_version"
```

## Annotation processing

The Kotlin plugin supports annotation processors like _Dagger_ or _DBFlow_. In order for them to work with Kotlin classes, apply the `kotlin-kapt` plugin:

``` groovy
apply plugin: 'kotlin-kapt'
```

Or, starting with Kotlin 1.1.1, you can apply it using the plugins DSL:

```groovy
plugins {
  id "org.jetbrains.kotlin.kapt" version "<version to use>"
}
```

Then add the respective dependencies using the `kapt` configuration in your `dependencies` block:
```groovy
dependencies {
  kapt 'groupId:artifactId:version'
}
```

If you previously used the [android-apt](https://bitbucket.org/hvisser/android-apt) plugin, remove it from your `build.gradle` file and replace usages of the `apt` configuration with `kapt`. If your project contains Java classes, `kapt` will also take care of them.

If you use annotation processors for your `androidTest` or `test` sources, the respective `kapt` configurations are named `kaptAndroidTest` and `kaptTest`. Note that `kaptAndroidTest` and `kaptTest` extends `kapt`, so you can just provide the `kapt` dependency and it will be available both for production sources and tests.

Some annotation processors (such as `AutoFactory`) rely on precise types in declaration signatures. By default, Kapt replaces every unknown type (including types for the generated classes) to `NonExistentClass`, but you can change this behavior. Add the additional flag to the `build.gradle` file to enable error type inferring in stubs:

``` groovy
kapt {
    correctErrorTypes = true
}
```

Note that this option is experimental and it is disabled by default.

## Incremental compilation

Kotlin supports optional incremental compilation in Gradle.
Incremental compilation tracks changes of source files between builds so only files affected by these changes would be compiled.

Starting with Kotlin 1.1.1, incremental compilation is enabled by default.

There are several ways to override the default setting:

  1. add `kotlin.incremental=true` or `kotlin.incremental=false` line either to a `gradle.properties` or a `local.properties` file;

  2. add `-Pkotlin.incremental=true` or `-Pkotlin.incremental=false` to gradle command line parameters. Note that in this case the parameter should be added to each subsequent build, and any build with disabled incremental compilation invalidates incremental caches.

When incremental compilation is enabled, you should see the following warning message in your build log:
```
Using kotlin incremental compilation
```

Note, that the first build won't be incremental.

## Coroutines support

[Coroutines](coroutines.html) support is an experimental feature in Kotlin 1.1, so the Kotlin compiler reports a warning when you use coroutines in your project.
To turn off the warning, add the following block to your `build.gradle` file:

``` groovy
kotlin {
    experimental {
        coroutines 'enable'
    }
}
```

## Compiler Options

To specify additional compilation options, use the `kotlinOptions` property of a Kotlin compilation task.

When targeting the JVM, the tasks are called `compileKotlin` for production code and `compileTestKotlin`
for test code. The tasks for custom source sets of are called accordingly to the `compile<Name>Kotlin` pattern.

When targeting JavaScript, the tasks are called `compileKotlin2Js` and `compileTestKotlin2Js` respectively, and `compile<Name>Kotlin2Js` for custom source sets.

Examples:

``` groovy
compileKotlin {
    kotlinOptions.suppressWarnings = true
}

compileKotlin {
    kotlinOptions {
        suppressWarnings = true
    }
}
```


A complete list of options for the Gradle tasks follows:

### Attributes common for JVM and JS

| Name | Description | Possible values |Default value |
|------|-------------|-----------------|--------------|
| `apiVersion` | Allow to use declarations only from the specified version of bundled libraries | "1.0", "1.1" | "1.1" |
| `languageVersion` | Provide source compatibility with specified language version | "1.0", "1.1" | "1.1" |
| `suppressWarnings` | Generate no warnings |  | false |
| `verbose` | Enable verbose logging output |  | false |
| `freeCompilerArgs` | A list of additional compiler arguments |  | [] |

### Attributes specific for JVM

| Name | Description | Possible values |Default value |
|------|-------------|-----------------|--------------|
| `javaParameters` | Generate metadata for Java 1.8 reflection on method parameters |  | false |
| `jdkHome` | Path to JDK home directory to include into classpath, if differs from default JAVA_HOME |  |  |
| `jvmTarget` | Target version of the generated JVM bytecode (1.6 or 1.8), default is 1.6 | "1.6", "1.8" | "1.6" |
| `noJdk` | Don't include Java runtime into classpath |  | false |
| `noReflect` | Don't include Kotlin reflection implementation into classpath |  | true |
| `noStdlib` | Don't include Kotlin runtime into classpath |  | true |

### Attributes specific for JS

| Name | Description | Possible values |Default value |
|------|-------------|-----------------|--------------|
| `main` | Whether a main function should be called | "call", "noCall" | "call" |
| `metaInfo` | Generate .meta.js and .kjsm files with metadata. Use to create a library |  | true |
| `moduleKind` | Kind of a module generated by compiler | "plain", "amd", "commonjs", "umd" | "plain" |
| `noStdlib` | Don't use bundled Kotlin stdlib |  | true |
| `outputFile` | Output file path |  |  |
| `sourceMap` | Generate source map |  | false |
| `target` | Generate JS files for specific ECMA version | "v5" | "v5" |


## OSGi

OSGi 支持查看 [Kotlin OSGi page](kotlin-osgi.html).

## 例子

[Kotlin Repository](https://github.com/jetbrains/kotlin) 包含的例子:

* [Kotlin](https://github.com/JetBrains/kotlin-examples/tree/master/gradle/hello-world)
* [Mixed Java and Kotlin](https://github.com/JetBrains/kotlin-examples/tree/master/gradle/mixed-java-kotlin-hello-world)
* [Android](https://github.com/JetBrains/kotlin-examples/tree/master/gradle/android-mixed-java-kotlin-project)
* [JavaScript](https://github.com/JetBrains/kotlin/tree/master/libraries/tools/kotlin-gradle-plugin-integration-tests/src/test/resources/testProject/kotlin2JsProject)
