+++
title = "Android中的Fragment"
date = "2019-07-06T09:30:00+08:00"
author = "do9core"
tags = ["Android"]
description = "Fragment的学习小结"
readingTime = true
+++

## 定义

碎片化界面容器，用于嵌入显示 APP 指定内容。Fragment表示 Activity中的行为或用户界面部分。可以将多个Fragment组合在一个Activity中或在多个Activity中重复使用某个Fragment。

Fragment具有自己的生命周期，能接收自己的输入事件，并且可以在Activity运行时添加或移除Fragment。

## 特性

* Fragment依赖Activity，不能独立存在
* 一个Activity可以包含多个Fragment
* 一个Fragment可以被多个Activity重用
* Fragment可以帮助Activity分离成多个组件
* 使用Fragment来便于适应不同尺寸的UI
* Fragment是一个独立模块，与Activity绑定，但可以在运行中动态移除、加入、交换
* 相比Activity更轻量级

## 状态

* Resumed

Fragment在运行中的Activity中可见

* Paused

另一个Activity位于前台并具有焦点，但此片段所在的Activity仍然可见（前台Activity部分透明或没有覆盖整个屏幕）

* Stopped
    1. 宿主Activity已停止
    2. Fragment从Activity中移除但在返回栈

停止的Fragment仍处于活动状态，系统会保留信息，但是对用户不可见。若此时Activity被终止，它也会被终止。

## 生命周期

整个生命周期流程如图：

![Fragment生命周期](fragment_lifecycle.png)

Fragment的生命周期与其所在的Activity存在一定关系，当其所在Activity运行时，其生命周期如上图所示。

但是，当Fragment所在的Activity暂停（Pause）、停止（Stop）或销毁（Destroy）时，Fragment也会随之进行相应的生命周期过程。

| 方法                  | 调用时机                                                                             |
| --------------------- | ------------------------------------------------------------------------------------ |
| `onAttach()`          | Fragment与Activity关联时（*生命周期的第一个方法*）                                   |
| `onCreate()`          | Fragment被创建时                                                                     |
| `onCreateView()`      | Fragment构建视图时                                                                   |
| `onActivityCreated()` | 关联的Activity`onCreate()`结束后（*此时才能使用Activity的资源*，已废弃，不建议使用） |
| `onStart()`           | Fragment进入可见状态（*与Activity类似，暂时不能交互*）                               |
| `onResume()`          | Fragment可见并且正在运行时调用，与Activity绑定                                       |
| `onPause()`           | 与Activity绑定，时机相同                                                             |
| `onStop()`            | 与Activity绑定，时机相同                                                             |
| `onDestoryView()`     | 移除Fragment的视图层次结构时                                                         |
| `onDestory()`         | Fragment不再使用，即将销毁时（*仍能从Activity中找到，因为还没Detach*）               |
| `onDetach()`          | Fragment与Activity解除绑定时（*生命周期的最后一个方法*）                             |

## 返回栈

可以将Fragment加入返回栈保存，需要时弹出并重新加载视图层次结构。

* `popBackStack()`
* `addToBackStack()`
* `addOnBackStackChangeListener`

## 使用

加载Fragment有两种方式，静态加载和动态加载

### 静态加载Fragment

使用Fragment的最简单方式，直接将Fragment作为控件写入布局文件中

1. 继承Fragment基类，重写`onCreateView()`决定Fragment的布局
2. 在Activity的布局中使用`<fragment />`声明Fragment，控件的事件处理等代码交由Fragment去处理（**注意：一旦使用这种方式创建Fragment，就不能通过事务方式将其移除或替换**）

### 动态加载Fragment

动态加载Fragment是更加常用的方式，需要一个容器View来装载Fragment。

在布局中建立容器后，可以在Activity中使用Fragment事务动态添加、删除、替换Fragment。

* `supportFragmentManager`用于获取FragmentManager
* `add()`、`remove()`和`replace()`是动态管理Fragment的常用方法，第一个参数为容器ID，第二个参数为Fragment实例，第三个参数为Fragment的Tag，如果使用Tag，后续就可以通过`supportFragmentManager.findFragmentByTag()`查找Fragment实例
* 一次事务中可以进行多个操作，只要最后`commit()`即可
* `commit()`是异步的，调用后可能不会立即产生反应
* `addToBackStack()`是可选的，若加入了回退栈，用户点击返回时可以回退事务；否则会直接销毁Activity

Fragment也可以在Fragment中被加载，此时如果希望加载的Fragment成为当前Fragment的子Fragment，需要使用`Fragment::childFragmentManager`来加载。

## 通信

### 接口方式

#### Fragment -> 其他组件

1. 定义通信的接口：

    ```kotlin
    interface OnFragmentInteractionListener {
        fun onSomethingHappen(d: DataType)
    }
    ```

2. 让需要通信的Activity或Fragment实现此接口

3. 在Fragment的`onAttach()`中获取接口对象

    ```kotlin
    override fun onAttach(context: Context?) {
        super.onAttach(context)
        mListener = parentFragmentManager as OnFragmentInteractionListener
    }
    ```

4. 之后就可以在Fragment需要传递数据时，调用`mListener`的`onSomethingHappen()`方法将数据传给Activity

### 直接调用方法

#### Activity -> Fragment

在Fragment中定义方法：

```kotlin
fun doSomething(arg: ArgumentType) {
    ...
}
```

然后直接在Activity中找到Fragment并调用方法

```kotlin
val arg = ArgumentType()
val fragment = supportFragmentManager.findFragmentByTag("TAG") as MyFragment
fragment.doSomething(arg)
```

#### Fragment -> Activity

确认Activity类型的情况下，直接在Fragment中强转Activity并调用`public`方法

```kotlin
val arg = ArgumentType()
val mainActivity = activity as MainActivity
mainActivity?.usefulMethod(arg)
```

### 用Activity作为中介

#### Fragment -> Fragment

如果两个Fragment都隶属于同一个Activity，那么一个Fragment可以由Activity作为中介调用另一个Fragment的`public`方法

```kotlin
val arg = ArgumentType()
val fragmentManager = activity?.supportFragmentManager
val anotherFragment = fragmentManager?.findFragmentByTag("TAG")
anotherFragment?.usefulMethod(arg)
```

### 使用AndroidX的Fragment Result API

在最新的AndroidX库中，Fragment提供了全新的`setResultListener`和`setResult`方法，
它们是以`FragmentManager`为中介进行通信的工具，最推荐使用这种方式来进行传递

在这种方式下，只要选择正确的`FragmentManager`即可完成通信

### 广播方式

非常不建议，作用范围太大，容易产生副作用
