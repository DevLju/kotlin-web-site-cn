---
type: doc
layout: reference
category: "Syntax"
title: "异常"
---

# 异常

## 异常类

Kotlin 中所有异常类都是 `Throwable` 类的子孙类。
每个异常都有消息、堆栈回溯信息和可选的原因。

使用 *throw*{: .keyword }-表达式来抛出异常。

``` kotlin
throw MyException("Hi There!")
```

使用 *try*{: .keyword }-表达式来捕获异常。

``` kotlin
try {
    // 一些代码
}
catch (e: SomeException) {
    // 处理程序
}
finally {
    // 可选的 finally 块
}
```

可以有零到多个 *catch*{: .keyword } 块。*finally*{: .keyword } 块可以省略。
但是 *catch*{: .keyword } 和 *finally*{: .keyword } 块至少应该存在一个。

### Try 是一个表达式

*try*{: .keyword } 是一个表达式，即它可以有一个返回值。

``` kotlin
val a: Int? = try { parseInt(input) } catch (e: NumberFormatException) { null }
```

*try*{: .keyword }-表达式的返回值是 *try*{: .keyword } 块中的
最后一个表达式或者是（所有）*catch*{: .keyword } 块中的最后一个表达式。
*finally*{: .keyword } 块中的内容不会影响表达式的结果。

## 受检的异常

Kotlin 没有受检的异常。这其中有很多原因，但我们会提供一个简单的例子。

以下是 JDK 中 `StringBuilder` 类实现的一个示例接口

``` java
Appendable append(CharSequence csq) throws IOException;
```

这个签名是什么意思？ 它是说，每次我追加一个字符串到一些东西（一个 `StringBuilder`、某种日志、一个控制台等）上时
我就必须捕获那些 `IOException`。 为什么？因为它可能正在执行 IO 操作（`Writer` 也实现了 `Appendable`）……
所以它导致这种代码随处可见的出现：

``` kotlin
try {
    log.append(message)
}
catch (IOException e) {
    // 必须要安全
}
```

这并不好，参见[《Effective Java》](http://www.oracle.com/technetwork/java/effectivejava-136174.html) 第 65 条：*不要忽略异常*。

Bruce Eckel 在[《Java 是否需要受检的异常？》（Does Java need Checked Exceptions?）](http://www.mindview.net/Etc/Discussions/CheckedExceptions) 中指出：

> 通过一些小程序测试得出的结论是异常规范会同时提高开发者的生产力和代码质量，但是大型软件项目的经验表明一个不同的结论——生产力降低、代码质量很少或没有提高。

其他相关引证：

* [《Java 的受检异常是一个错误》（Java's checked exceptions were a mistake）](http://radio-weblogs.com/0122027/stories/2003/04/01/JavasCheckedExceptionsWereAMistake.html)（Rod Waldhoff）
* [《受检异常的烦恼》（The Trouble with Checked Exceptions）](http://www.artima.com/intv/handcuffs.html)（Anders Hejlsberg）

## The Nothing type

`throw` is an expression in Kotlin, so you can use it, for example, as part of an Elvis expression:

``` kotlin
val s = person.name ?: throw IllegalArgumentException("Name required")
```

The type of the `throw` expression is the special type `Nothing`.
The type has no values and is used to mark code locations that can never be reached.
In your own code, you can use `Nothing` to mark a function that never returns:

``` kotlin
fun fail(message: String): Nothing {
    throw IllegalArgumentException(message)
}
```

When you call this function, the compiler will know that the execution doesn't continue beyond the call:

``` kotlin
val s = person.name ?: fail("Name required")
println(s)     // 's' is known to be initialized at this point
```

## Java 互操作性

与 Java 互操作性相关的信息，请参见 [Java 互操作性章节](java-interop.html)中的异常部分。
