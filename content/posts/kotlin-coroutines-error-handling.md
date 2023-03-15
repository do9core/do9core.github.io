+++
title = "协程和协程的等级并不相同→协程的异常处理"
date = "2020-09-03T09:45:00+08:00"
author = "do9core"
tags = ["Kotlin"]
description = "就好比人与人的悲欢并不相通"
readingTime = true
+++

这篇文章打算说一说协程的异常处理。

## 普通函数的异常处理

相信大家对于这个问题再熟悉不过了。

使用`try`和`catch`来进行异常的捕获，在JVM上，未经捕获的异常会被传递给`Thread.uncaughtExceptionHandler()`（下面简称UEH）来处理，结果大家应该也都很明白了，我们的程序会直接崩溃啦。

```kotlin
fun main() {
  try {
    codeThatMayThrowException()
  } catch (t: Throwable) {
    // ...
  }
}
```

## 挂起函数的异常处理

挂起函数内的错误处理其实和普通函数别无二至，但是挂起函数的错误，会被其所在的协程处理，这就涉及到下一部分的内容了。

```kotlin
suspend fun someFunction() {
  try {
    codeThatMayThrowException()
  } catch (t: Throwable) {
    // ...
  }
}
```

## 协程的异常处理

### 协同作用域下的协程（Job）

通常情况下，Kotlin的协程是运行在协同作用域下的，借用《深入理解Kotlin协程》中的一句话，就是：

> “子异常则父连坐”

即当子协程抛出**未经捕获**的异常时，父协程会“被迫”取消所有子协程和自身：

```kotlin
GlobalScope.launch {  // this: CoroutineScope
  val child1 = launch { // this: CoroutineScope
    error("Boom!")
  }
  child1.join()
  println("Done!") // Won't reach and crash.
}
```

通过`launch`构造器创建的协程，在遇到未经捕获的异常时会直接抛出给父协程处理，所以处理这些问题的方式就是直接捕获协程内的异常：

```kotlin
val child1 = launch { // this: CoroutineScope
  try {
    error("Boom!")
  } catch(e: Exception) {
    // handle the exception
  }
}
```

如果交由父级来处理，则需要使用`CoroutineExceptionHandler`，请注意，`CoroutineExceptionHandler`必须配置给`CoroutineScope`或最顶级协程（即直接在`CoroutineScope`下启动的协程）：

```kotlin
val exceptionHandler = CoroutineExceptionHandler { 
  _: CoroutineContext, t: Throwable ->
  // handle exception
  println("Caught $t")
}

GlobalScope.launch(exceptionHandler) {  // this: CoroutineScope
  val child1 = launch { // this: CoroutineScope
    error("Boom!")
  }
  child1.join()
  println("Done!") // Won't reach, but coroutine finished normally.
}
```

使用同样的`CoroutineExceptionHandler`，错误示范：

```kotlin
GlobalScope.launch {  // this: CoroutineScope
  val child1 = launch(exceptionHandler) { // this: CoroutineScope
    error("Boom!")
  }
  child1.join()
  println("Done!") // Won't reach and crash.
}
```

上面的方法是不会正常工作的，在协同作用域下的协程，只有顶层协程有权利处理未经捕获的异常，子协程发生的所有未捕获异常，都会导致整个协同作用域的瓦解，即便用`async`也是同样的结果：

```kotlin
GlobalScope.launch { // this: CoroutineScope
  val task = async<Int> { // this: CoroutineScope
    error("Boom!")
  }
  try {
    // Even we don't call await(), it still crash.
    task.await()
  } catch(e: Exception) {
    println("launch Caught: $e")
  }
  println("Done!")
}
```

和上面是同样的道理，`async`发生了未经捕获的异常，但是由于处于协同作用域下，它无权处理此异常，必须向父级传递，传递给launch启动的协程时，因为launch已经是顶层协程，所以它将会处理这个异常。

顶层协程在处理异常时，会优先考虑使用`CoroutineExceptionHandler`，但如果协程上下文中没有`CoroutineExceptionHandler`，那么这个顶层协程只能将异常交由当前线程的`Thread.uncaughtExceptionHandler()`来处理（JVM），此时程序就会崩溃啦。

为何要这样设计呢？

这样设计的目的是保证协程在遭遇错误时正确释放协程占用的资源。

### 主从作用域下的协程（SupervisorJob）

但有些场景下，这不符合我们的需求，一个并发任务产生问题，我们不希望取消其他的并发任务，而只需要处理一下这个错误就可以了，这个时候，我们就需要断开这个错误的传递链。

> ⚔️ 看我，拿胜利宝剑，断开魂结，断开锁链，断开一切的牵连~

这时候就要介绍一下`SupervisorJob`，它可以帮我们完成这个需求。

我们如何通过`SupervisorJob`来断开异常传递呢？最简单的方式就是创建一个`SupervisorJob()`为`CoroutineContext`的`CoroutineScope`，并在其内部启动协程，这样每个**直接**子协程就拥有了自己处理异常的权力：

```kotlin
val myScope = CoroutineScope(SupervisorJob())

// async throws excpetion.
// But myScope just accknowledged the exception and won't handle it.
val task = myScope.async<Int> { // this: CoroutineScope
  error("Boom!")
}

val job = myScope.launch { // this: CoroutineScope
  try {
    // Because our async finished with an exception.
    // The exception will be rethrown when we call await()
    task.await()
  } catch (e: Exception) {
    println("launch Caught $e")
  }
}

fun main() {
  // finished normally
  runBlocking { // this: CoroutineScope
    println("Wait...")
    job.join()
    println("Finish.")
  }
}
```

但是这个方法有些麻烦，有时候我们只是做一些简单的并发，所以协程框架为我们提供了`supervisorScope()`这个函数。

这个函数是一个挂起函数，会帮我们创建一个带有`SupervisorJob`的`CoroutineScope`，此函数会挂起(`suspend`)，直到这个`CoroutineScope`下启动的所有协程结束后恢复(`resume`)：

```kotlin
suspend fun <T, R> mapParallel(items: List<T>, transform: suspend (T) -> R): List<R> {
  if (items.isEmpty()) {
    // Fast path
    return emptyList()
  }
  return supervisorScope { // this: CoroutineScope
    val tasks = items.map { item ->
      async { // this: CoroutineScope
        transform(item)
      }
    }
    try {
      tasks.awaitAll()
    } catch (e: Exception) {
      emptyList()
    }
  }
}
```

我们用上面这个函数来进行一个并发的`map`操作：

```kotlin
fun main() = runBlocking {
  val source = listOf(1, 2, 3, 4)
  val result = mapParallel(source) { // it: Int
    delay(100)
    "mapped($it)"
  }
  println(result) // [mapped(1), mapped(2), mapped(3), mapped(4)]
}
```

没有异常的情况下，正常输出了`map`后的结果，如果我们加入一点异常情况：

```kotlin
val result = mapParallel(source) { // it: Int
  delay(100)
  if (it == 3) {
    error("I hate 3.")
  }
  "mapped($it)"
}
```

因为我们已经在`mapParallel`内进行了错误处理，所以此处我们可以得到正确结果:`[]`（一个空列表）。

是不是非常方便？但是在使用`SupervisorJob`时，有一个需要注意的地方，只有`SupervisorJob`的直接子协程才能获得处理异常的权力，下级的子协程仍会将异常向上传递。

我们知道每个协程在被创建时，都创建了`Job`作为生命周期标识（`AbstractCoroutine`实现了`Job`接口），一旦下层的异常传递到`SupervisorJob`下的子协程，其内部的`Job`仍会以协同作用域方式处理异常，如果此时没有`CoroutineExceptionHandler`来处理异常，异常将会被传递给`Thread`的`Thread.uncaughtExceptionHandler()`，程序仍然会崩溃：

```kotlin
fun main() {
  runBlocking {
    supervisorScope { // this: CoroutineScope [SupervisorJob]
      launch { // this: CoroutineScope [Job]
        // will crash
        val task = async<Int> { // this: CoroutineScope
          error("Boom!")
        }
      }
    }
  }
}
```

## 简单的总结

1. 对于事务性的操作，最好使用协同作用域，确保一个并发任务的所有部分都正确完成
2. 对于子任务间相对隔离的操作，可以考虑使用主从作用域，给予子协程处理异常的权力
3. 处理异常的场景下，最好使用`CoroutineExceptionHandler`

---

## 参考资料

1. 《深入理解Kotlin协程》 —— 霍丙乾
2. Exceptional Exceptions for Coroutines made easy…?

    [Exceptional Exceptions for Coroutines made easy...?](https://medium.com/the-kotlin-chronicle/coroutine-exceptions-3378f51a7d33)

3. Exceptions in coroutines

    [Exceptions in coroutines](https://medium.com/androiddevelopers/exceptions-in-coroutines-ce8da1ec060c)
