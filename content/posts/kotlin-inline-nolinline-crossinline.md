+++
title = "Kotlin内联函数关键字inline、noinline和crossinline"
date = "2022-07-20T09:45:00+08:00"
author = "do9core"
tags = ["Kotlin"]
description = "对比几个内联关键字的选项区别"
readingTime = true
+++

你在网上查这个东西，得到的结果肯定是这样：

- `inline`: 函数的代码内联到调用处
- `noinline`: `inline`函数的形参中，不希望内联的lambda，此时这个lambda里不能`return`
- `crossinline`: `inline`函数的形参中的lambda不能有`return`

你这乍一看，`noinline`和`crossinline`好像根本没毛线区别  
但事实不是这样的

### inline做了些啥

`inline`可以将代码块内联，同时导致一个结果，你可以在这个内联的代码块内部，直接返回调用此代码块的函数，例如：

```kotlin
inline fun testInline(block: () -> Unit) {
    block()
    println("Hey! Don't pass me!")
}

fun main() {
    testInline {
        println("I'm the evil block")
        // This will pass the println() call in testInline()
        return
    }
}

// 上面的代码内联后：
fun main() {
    println("I'm the evil block")
    return
    // Cannot reach this line
    println("Hey! Don't pass me!")
}
```

这导致了一个问题，在`block()`执行后，可能无法完成`inline`函数中的剩余过程，这在很多Builder模式的lambda中是非常致命的，此时，就需要禁用这种非局部返回，就需要使用到`crossinline`

### crossinline

使用`crossinline`修饰lambda，就可以阻止lambda内部的`return`，`return`只可以返回到lambda的作用域内

```kotlin
inline fun testInline(crossinline block: () -> Unit) {
    block()
    println("You can't pass me again!")
}

fun main() {
    testInline {
        println("HAHAHAHAHAHA!")
        // 'return' is not allowed here
        return
        // return@testInline is allowed
    }
}
```

### noinline

再说说`noinline`，直译这个关键字也能看得挺明白了，就是不让你`inline`，用原来的方式传递lambda参数。

那什么场景下需要`noinline`呢？一般就是你需要一个`lambda`的引用的时候，例如：

```kotlin
class Timer(
    val interval: Long,
    val callback: () -> Unit
) { 
    /* other logic that may call tick() */
    
    /* invoke callback */
    fun tick() = callback()
}

inline fun makeTimer(
    interval: () -> Long,
    noinline callback: () -> Unit
) = Timer(interval(), callback)

fun main() {
    val timer = makeTimer(
        interval = { 1000L },
        callback = { println("1 seconds passed.") }
    )
}
```

在`makerTimer`中的`callback`参数前，你必须添加`noinline`关键字，因为`Timer`需要持有`callback`这个lambda的引用才能在合适的时机调用它，自然也就不能把它做成内联的了。

而`interval`这个参数只有调用`makeTimer`时执行一次，自然就可以内联。

但是要注意，虽然`noinline`也可以实现`crossinline`的效果，但是与`crossinline`有所不同

如果使用`noinline`，对lambda表达式的调用会转换成普通的lambda调用，即生成`Function`实例，然后传递实例，并调用实例的`invoke()`方法来运行lambda，这样自然也就禁用了非局部返回，因为此时已经在另一个函数的域内了，`return`根本不知道返回到哪。

### 总结

绝大多数情况下，使用`noinline`引起的不能局部返回是**结果**，而使用`crossinline`禁用局部返回是**目的**。

### 参考资料

[inline, noinline, crossinline — What do they mean?](https://android.jlelse.eu/inline-noinline-crossinline-what-do-they-mean-b13f48e113c2)
