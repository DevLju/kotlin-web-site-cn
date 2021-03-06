---
type: doc
layout: reference
category: "Syntax"
title: "内联函数"
---

# 内联函数

使用[高阶函数](lambdas.html)会带来一些运行时的效率损失：每一个函数都是一个对象，并且会捕获一个闭包。
即那些在函数体内会访问到的变量。
内存分配（对于函数对象和类）和虚拟调用会引入运行时间开销。

但是在许多情况下通过内联化 lambda 表达式可以消除这类的开销。
下述函数是这种情况的很好的例子。即 `lock()` 函数可以很容易地在调用处内联。
考虑下面的情况：

``` kotlin
lock(l) { foo() }
```

编译器没有为参数创建一个函数对象并生成一个调用。取而代之，编译器可以生成以下代码：

``` kotlin
l.lock()
try {
    foo()
}
finally {
    l.unlock()
}
```

这个不是我们从一开始就想要的吗？

为了让编译器这么做，我们需要使用 `inline` 修饰符标记 `lock()` 函数：

``` kotlin
inline fun lock<T>(lock: Lock, body: () -> T): T {
    // ...
}
```

`inline` 修饰符影响函数本身和传给它的 lambda 表达式：所有这些都将内联
到调用处。

内联可能导致生成的代码增加，但是如果我们使用得当（不内联大函数），它将在
性能上有所提升，尤其是在循环中的“超多态（megamorphic）”调用处。

## 禁用内联

如果你只想被（作为参数）传给一个内联函数的 lamda 表达式中只有一些被内联，你可以用 `noinline` 修饰符标记
一些函数参数：

``` kotlin
inline fun foo(inlined: () -> Unit, noinline notInlined: () -> Unit) {
    // ...
}
```

可以内联的 lambda 表达式只能在内联函数内部调用或者作为可内联的参数传递，
但是 `noinline` 的可以以任何我们喜欢的方式操作：存储在字段中、传送它等等。

需要注意的是，如果一个内联函数没有可内联的函数参数并且没有
[具体化的类型参数](#具体化的类型参数)，编译器会产生一个警告，因为内联这样的函数
很可能并无益处（如果你确认需要内联，则可以关掉该警告）。

## 非局部返回

在 Kotlin 中，我们可以只使用一个正常的、非限定的 `return` 来退出一个命名或匿名函数。
这意味着要退出一个 lambda 表达式，我们必须使用一个[标签](returns.html#标签处返回)，并且
在 lambda 表达式内部禁止使用裸 `return`，因为 lambda 表达式不能使包含它的函数返回：

``` kotlin
fun foo() {
    ordinaryFunction {
        return // 错误：不能使 `foo` 在此处返回
    }
}
```

但是如果 lambda 表达式传给的函数是内联的，该 return 也可以内联，所以它是允许的：

``` kotlin
fun foo() {
    inlineFunction {
        return // OK：该 lambda 表达式是内联的
    }
}
```

这种返回（位于 lambda 表达式中，但退出包含它的函数）称为*非局部*返回。 我们习惯了
在循环中用这种结构，其内联函数通常包含：

``` kotlin
fun hasZeros(ints: List<Int>): Boolean {
    ints.forEach {
        if (it == 0) return true // 从 hasZeros 返回
    }
    return false
}
```

请注意，一些内联函数可能调用传给它们的不是直接来自函数体、而是来自另一个执行
上下文的 lambda 表达式参数，例如来自局部对象或嵌套函数。在这种情况下，该 lambda 表达式中
也不允许非局部控制流。为了标识这种情况，该 lambda 表达式参数需要
用 `crossinline` 修饰符标记:

``` kotlin
inline fun f(crossinline body: () -> Unit) {
    val f = object: Runnable {
        override fun run() = body()
    }
    // ...
}
```


> `break` 和 `continue` 在内联的 lambda 表达式中还不可用，但我们也计划支持它们

## 具体化的类型参数

有时候我们需要访问一个作为参数传给我们的一个类型：

``` kotlin
fun <T> TreeNode.findParentOfType(clazz: Class<T>): T? {
    var p = parent
    while (p != null && !clazz.isInstance(p)) {
        p = p?.parent
    }
    @Suppress("UNCHECKED_CAST")
    return p as T
}
```

在这里我们向上遍历一棵树并且检查每个节点是不是特定的类型。
这都没有问题，但是调用处不是很优雅：

``` kotlin
myTree.findParentOfType(MyTreeNodeType::class.java)
```

我们真正想要的只是传一个类型给该函数，即像这样调用它：

``` kotlin
myTree.findParentOfType<MyTreeNodeType>()
```

为能够这么做，内联函数支持*具体化的类型参数*，于是我们可以这样写：

``` kotlin
inline fun <reified T> TreeNode.findParentOfType(): T? {
    var p = parent
    while (p != null && p !is T) {
        p = p?.parent
    }
    return p as T
}
```

我们使用 `reified` 修饰符来限定类型参数，现在可以在函数内部访问它了，
几乎就像是一个普通的类一样。由于函数是内联的，不需要反射，正常的操作符如 `!is`
和 `as` 现在都能用了。此外，我们还可以按照上面提到的方式调用它：`myTree.findParentOfType<MyTreeNodeType>()`。

虽然在许多情况下可能不需要反射，但我们仍然可以对一个具体化的类型参数使用它：

``` kotlin
inline fun <reified T> membersOf() = T::class.members

fun main(s: Array<String>) {
    println(membersOf<StringBuilder>().joinToString("\n"))
}
```

普通的函数（未标记为内联函数的）不能有具体化参数。
不具有运行时表示的类型（例如非具体化的类型参数或者类似于`Nothing`的虚构类型）
不能用作具体化的类型参数的实参。

相关底层描述，请参见[规范文档](https://github.com/JetBrains/kotlin/blob/master/spec-docs/reified-type-parameters.md)。

## Inline properties (since 1.1)

The `inline` modifier can be used on accessors of properties that don't have a backing field.
You can annotate individual property accessors:

``` kotlin
val foo: Foo
    inline get() = Foo()

var bar: Bar
    get() = ...
    inline set(v) { ... }
```

You can also annotate an entire property, which marks both of its accessors as inline:

``` kotlin
inline var bar: Bar
    get() = ...
    set(v) { ... }
```

At the call site, inline accessors are inlined as regular inline functions.
