+++
title = "Kotlin局部返回"
date = "2022-07-20T09:34:00+08:00"
author = "do9core"
tags = ["Kotlin"]
description = "总结Kotlin中的局部返回"
readingTime = true
+++

这个问题主要体现在`forEach()`和`repeat()`函数中

通常，我们会如下方式调用这两个函数：

```kotlin
(1..100).forEach { println(it) }
```

但是如果我想像通常循环一样，在特定点跳出`forEach()`呢？

对于条件明确的，我们可以在`forEach()`函数前明确范围，比如：

```kotlin
(1..100).drop(5).take(50).forEach { println(it) }
```

但是如果条件不明确，需要在循环中判断，这种方法就不行了

那`break`呢？

直接使用`break`肯定是不行的，因为这在一个lambda表达式内部

此时，你可能会说，既然是lambda内，我用`return`不就完了吗？

但是如果你这么写：

```kotlin
(1..100).forEach {
    println(it)
    if (it > 50) {
        return@forEach
    }
}
```

那你就掉坑里了，看似这是让函数返回到`forEach()`执行，实际上，这是一个局部返回

你只是返回到了`forEach()`函数的`block`而已，并没有离开`forEach()`函数！

这么写可能更直观：

```kotlin
(1..100).forEach loop@{
    println(it)
    if (it > 50) {
        return@loop
    }
}
```

我们此时根本没有实现`break`的效果，相反，实现的是`continue`的效果

如果想要`break`，就需要使用到非局部返回，查看kotlin中`forEach()`函数的源代码可以发现

`forEach()`函数的`block`并未被标记为`crossinline`或`noinline`，所以是可以使用非局部返回的：

```kotlin
fun nonLocal() {
    (1..100).forEach {
        println(it)
        if (it > 50) {
            return@nonLocal
        }
    }
}
```

这样我们就实现了`break`效果，成功从`forEach()`函数离开了

你可能会觉得这也太不方便了，难道我用到`forEach()`的地方想要`break`，就要套一层函数吗？

其实不然，kotlin中，我们有可以随处运行的`run()`函数

当你需要break一个forEach的时候，只要用run函数包裹就可以了：

```kotlin
run {
    (1..100).forEach {
        println(it)
        if (it > 50) {
            return@run
        }
    }
}

// other logic
```

其实不仅仅是`repeat()`和`forEach()`，所有传入lambda的函数，都是如此工作的

一般的函数中因为不存在循环，所以我们直接用局部返回，回到调用函数后，它完成其他工作自然就退出了

但是这类存在循环的函数中，如果对局部返回和非局部返回的理解不到位  
就会发生 “我明明return了，为什么还是执行了多次？” 这样的问题
