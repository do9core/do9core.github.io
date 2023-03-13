+++
title = "使用StateFlow和SharedFlow替代LiveData"
date = "2021-09-03T09:25:00+08:00"
author = "do9core"
tags = ["Kotlin", "Android"]
description = "理解StateFlow和SharedFlow，如何用它们替代LiveData"
readingTime = true
+++

## 放在前面

随着Kotlin协程1.4版本的正式发布，我们终于迎来了稳定版本的`StateFlow`和`SharedFlow`。

得益于协程的结构化并发与Android组件生命周期的良好协调性，也许已经是时候使用他们来代替`LiveData`了。

## SharedFlow

在Kotlin协程中，`Flow`被设计为"Cold Flow"，只有当`FlowCollector`开始收集时，`Flow`才会被**激活**。

这种设计有一个问题，那就是`Flow`的收集者占据着`Flow`的主导权，不论是启动还是关闭，控制权都在收集者手中，这就导致`Flow`在共享构建起来成本较高的资源时不太方便。

`Flow`这个概念，不仅仅局限于“一组数据”，也可以是一个“变化的实体”。

所以`SharedFlow`将`Flow`“加热”，让`Flow`的主导权转移到`Flow`自身而非收集者，从而实现`Flow`状态在多个收集者之间共享，解决了共享高成本资源的问题。

### SharedFlow的接口定义

```kotlin
interface SharedFlow<out T> : Flow<T> {
    /**
     * 重放数据
     * [replayCache]中的数据，会被发射给新的订阅者
     */
    val replayCache: List<T>
}

interface MutableSharedFlow<T> : SharedFlow<T>, FlowCollector<T> {
    // 尝试发射[value]到当前SharedFlow而不挂起
    fun tryEmit(value: T): Boolean
    // 订阅者数量的StateFlow
    val subscriptionCount: StateFlow<Int>
    // 清空[replayCache]
    fun resetReplayCache()
}
```

### SharedFlow的应用场景

* 描述事件

  由于`LiveData`描述状态的特性（即“倒灌”），它并不能很好地进行事件类型数据的表达。

  虽然在开发者们的不断探索中，诞生了`SingleLiveEvent`，`Event Wrapper`这些的方式，但是往往存在许多问题，并不是很完善的解决方案。

  `SharedFlow`的诞生解决了这些问题，也可以说它就是为了解决这个问题。

  对于`SharedFlow`的收集者，只会得到其**订阅后**发射到`SharedFlow`中的数据，不会被`SharedFlow`的当前状态干扰，这使它非常适合用于处理事件。

  这就和`Rx`中的`Subject`十分类似。

  同时，收集者的生命周期是依赖于协程的，而协程的生命周期由作用域`CoroutineScope`来控制，通过维护适当的`CoroutineScope`，我们可以最大程度避免内存泄漏的问题。

  设计文档中就提供了用`SharedFlow`实现`EventBus`的示例。

* 共享数据

  对于构建成本较高的数据，可以使用`SharedFlow`来进行共享。

### SharedFlow的特殊情境

* onStart操作符

  由于`SharedFlow`的Hot Flow特性，`onStart`操作符在其上的作用就显得十分微妙：`SharedFlow`启动时，可能没有任何收集器，所以`onStart`中发射的数据可能永远不会被接收到。

  为了解决这个问题，协程为`SharedFlow`引入了`onSubscription`操作符，允许在收集器开始收集时，为`SharedFlow`插入数据，`onSubscription`保证后续的收集器可以获得其发射的数据:

  ```kotlin
  // The 100 ensure being collected by collector
  val sharedFlow = mySharedFlow.onSubscription { emit(100) } 
  ```

* `SharedFlow`永不结束

  也是由于`SharedFlow`的Hot Flow特性。

  这意味着`SharedFlow`的收集者除非因协程取消，否则收集的过程永远不会结束，所有依赖收集结束的**终止操作符**(如`toList`)也将永远挂起。

  这是需要开发者在开发过程中谨慎区分的。

### SharedFlow的设计文档

对完整内容感兴趣，可以阅读`SharedFlow`的[设计文档](https://github.com/Kotlin/kotlinx.coroutines/issues/2034)

## StateFlow

`SharedFlow`适用于描述事件，`StateFlow`适用于描述状态，因为`StateFlow`可以保存最后的状态。

从作用上来看，`StateFlow`就等同于一个`replayCache`为`1`，背压策略为`conflated`的`SharedFlow`，它总是会保留最后被设置的值。

但是根据文档的说明，`StateFlow`具有一个独立的更加高效的实现。

因为`StateFlow`扩展了`SharedFlow`，所以与`SharedFlow`一样，`StateFlow`属于Hot Flow。

同时，这也使得`SharedFlow`成为了Kotlin协程`Flow`中唯一的Hot Flow，减轻了开发者区分各种流是Cold还是Hot的负担。

### StateFlow的接口定义

```kotlin
interface StateFlow<out T> : SharedFlow<T> {
    // StateFlow的当前值（只读）
    val value: T
}

interface MutableStateFlow<T> : StateFlow<T>, MutableSharedFlow<T> {
    // StateFlow的当前值（可写）
    override var value: T
    // CAS，大家都懂
    fun compareAndSet(expect: T, update: T): Boolean
}
```

### StateFlow的应用场景

* 描述状态

  正如`StateFlow`的名字一样，它就是为**状态**而生的`Flow`，而这个作用等同于`LiveData`目前的作用。

  对于`StateFlow`的收集者，在开始收集时，会得到`StateFlow`的当前状态，即`value`，从而可以不间断地获得状态值。

  同时，由于`MutableStateFlow`构造方法要求的默认值，我们也避免了使用`MutableLiveData`时，收到`null`值的问题。

### StateFlow的特殊情境

* `StateFlow`永不结束

  与`SharedFlow`一样，因为Hot Flow特性，`StateFlow`也不会结束。
  
* `Conflated`

  我们可以直接使用`MutableStateFlow`的`value`属性为其赋值，而且不需要挂起，这是因为`StateFlow`是“`Conflated`”的，这一策略就是保留最新值，丢弃旧值。体现在UI上，就是总会表示最新的结果。
  
  在这种生产者、消费者协同工作环境下，消费者消费速度过快，那么消费者将被挂起，但是生产者过快，可以根据实际场景选择不同的背压策略，在协程中，背压策略一般有以下几种：
  
  1. `SUSPEND`: 过快的生产者被挂起，直到消费者处理完后恢复
  2. `DROP_LATEST`: 过快的生产者生产的**最新**数据被丢弃
  3. `DROP_OLDEST`: 过快的生产者生产的**最旧**数据被丢弃
  4. `BUFFERED`: 过快的生产者生产的数据被保存到缓冲区中

### StateFlow的设计文档

对完整内容感兴趣，可以阅读`StateFlow`的[设计文档](https://github.com/Kotlin/kotlinx.coroutines/issues/1973)

## 替换LiveData

### 从事件LiveData到SharedFlow

这部分的替换显得十分直接，之前的`SingleLiveEvent<T>`也好，`MutableLiveData<Event<T>>`也罢，都可以直接简单地替换为`SharedFlow<T>`:

```kotlin
// Before:
class SomeViewModel : ViewModel() {
    
    val someEvent: SingleLiveEvent<Int> = SingleLiveEvent()
    
    fun event() {
        someEvent.value = 100
    }
    
    /* Or:
    private val _someEvent: MutableLiveData<Event<Int>> = MutableLiveData()
    val someEvent: LiveData<Event<Int>> get() = _someEvent
    
    fun event() {
       _someEvent.value = Event(100)
    }
    */
}

// After:
class SomeViewModel : ViewModel() {
    
    // 还可以根据需要配置背压策略，但是EventWrapper和SingleLiveData都是做不到的
    private val _someEvent: MutableSharedFlow<Int> = MutableSharedFlow()
    val someEvent: SharedFlow<Int> get() = _someEvent
    
    fun event() {
        _someEvent.tryEmit(100)
    }
    
    suspend fun eventSuspend() {
        _someEvent.emit(100)
    }
}
```

观察者一侧，使用`Lifecycle`库`LifecycleOwner`提供的`lifecycleScope`启动协程来进行收集：

```kotlin
class SomeActivity : AppCompatActivity() {
 
    val viewModel: SomeViewModel by viewModels()
    
    override fun onCreate(savedInstanceState: Bundle?) {
        lifecycleScope.launch {
            /**
             * 可以根据需求灵活选择生命周期
             */
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.someEvent.collect { params ->
                  // 处理事件
                }
            }
        }
    }
}
```

所有在`lifecycleScope`下启动的协程，在`Lifecycle`抵达`DESTROYED`状态时，都会自动取消（**前提是协程可以取消**）。

### 从状态LiveData到StateFlow

描述状态才是一个`LiveData`的正确使用方法。

从状态型`LiveData`迁移到`StateFlow`的过程与事件型的`LiveData`迁移到`SharedFlow`类似，但是与之不同的是，`StateFlow`是要求初始值的：

```kotlin
// Before:
class SomeViewModel : ViewModel() {
    
    private val _someState: MutableLiveData<SomeModel> = MutableLiveData()
    val someState: LiveData<SomeModel> get() = _someState
    
    fun updateState(newState: SomeModel) {
        _someState.value = newState
    }
}

// After:
class SomeViewModel : ViewModel() {
    
    private val _someState: MutableStateFlow<SomeModel> = MutableStateFlow(SomeModel.initial)
    val someState: StateFlow<SomeModel> get() = _someState
    
    fun updateState(newState: SomeModel) {
        _someState.value = newState
    }
}
```

这种强制初始值的方式，在我看来更符合“**状态**”的定义，因为初始状态也是状态，应该有一个值来描述这个状态，而不是像`LiveData`一样，没有值。

观察侧，同样使用`lifecycleScope`，和平常收集`Flow`一样：

```kotlin
class SomeActivity : AppCompatActivity() {
 
    val viewModel: SomeViewModel by viewModels()
    
    override fun onCreate(savedInstanceState: Bundle?) {
        lifecycleScope.launch {
            /**
             * 可以根据需求灵活选择生命周期
             */
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.someState.collect { state ->
                    // 更新状态
                }
            }
        }
        // 也可以轻松地访问状态的当前值
        viewModel.someState.value
    }
}
```
