+++
title = "Android Databinding"
date = "2020-02-11T21:08:00+08:00"
author = "do9core"
tags = ["Android"]
description = "DataBing学习总结"
readingTime = true
archive =  true
+++

> 老古董了，不推荐使用，推荐去用ViewBinding，或者直接一步到位用Compose吧

## 开启Data Binding

添加Data Binding到gradle构建文件中：

```gradle
android {
    // ...
    dataBinding {
        enabled = true
    }
    // ...

    // 在新版本的AGP中，需要使用以下方式
    buildFeatures {
        dataBinding true
    }
}
```

> *Android Studio的Data Binding插件需要Android Studio 1.3.0 或 更高版本。*

## 使用Data Binding

### Layout文件

使用Data Binding的Layout文件有一些不同，根标签是`<layout />`，然后包含一个**data**元素和一个**view**根元素，这个view元素即为不使用Data Binding时的根元素。

例如：

```xml
<layout>
    <data>
        <variable 
            name="user"
            type="com.example.model.User"/>
    </data>
    <LinerLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent">
        <TextView 
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="@{user.name}"/>
    </LinerLayout>
</layout>
```

`<data />`内部描述了一个名为`user`的`User`类型变量，这个变量的属性可以在View中访问。

View中需要获取变量属性时，需要使用`@{}`，比如TextView在获取user的name属性时，使用了`android:text="@{user.name}"`。

需要将视图上的修改返回到变量中，可以使用`@={}`建立双向绑定，比如：`android:text="@{user.name}"`，当输入的值修改时，新的值将会反映到变量所绑定的对象上。

除此之外，Layout中还包含其他用法：

* Import

    在`<data />`内部可以使用`<import />`添加引用，如：

    ```xml
    <data>
        <import type="android.view.View"/>
    </data>
    ```

    之后，可以在布局中使用引入的`View`类型：

    ```xml
    <TextView 
        ...
        ...
        android:visibility="@{bird.canFly ? View.VISIBLE : View.GONE}"/>
    ```

    如果多个`<import />`导入了相同类名不同包名的引用，可以通过`alias`属性指定引用名称消除冲突：

    ```xml
    <!-- View 对应 android.view.View -->
    <import type="android.view.View">
    <!-- Vista 对应 com.example.real.estate.View -->
    <import type="com.example.real.estate.View"
            alias="Vista"/>
    ```

    使用`<import />`导入的类型可以直接在`<variable />`中使用：

    ```xml
    <data>
        <import type="com.example.User"/>
        <variable name="user" type="User"/>
    </data>
    ```

* Variables

    在`<data />`中可以使用任意数量的`<variable />`来添加用于绑定的属性。

* Includes

    通过使用`<include />`将变量传递到被包含的布局中：

    ```xml
    <layout ...>
        <data>
            <variable name="user" type="com.example.User"/>
        </data>
        <LinearLayout ...>
            <include layout="@layout/name"
                bind:user="@{user}"/>
            <include layout="@layout/contact"
                bind:user="@{user}"/>
        </LinearLayout>
    </layout>
    ```

    > *`name.xml`和`contact.xml`中必须有变量`user`才能成功传入*

* 表达式

    1. 数值运算 `+`, `-`, `*`, `/`, `%`
    2. 字符串链接 `+`
    3. 逻辑运算 `&&`, `||`
    4. 二进制运算 `&`, `|`, `!`
    5. 移位 `>>`, `>>>`, `<<`
    6. 比较 `==`, `!=`, `>`, `<`, `>=`, `<=`
    7. 类型判断 `instanceof`
    8. 分组 `()`
    9. 空值 `null`
    10. 转换 `Cast`
    11. 方法调用 `.`
    12. 数据访问 `[]`
    13. 三元运算 `?:`
    14. 空值合并 `??`

    不支持的表达式：
    1. `this`
    2. `super`
    3. `new`
    4. 显式泛型调用

### Binding

默认情况下，使用Data Binding后，会生成一个Binding类，其名称由布局文件的文件名决定，比如：`activity_main.xml`文件生成的对应Binding类为`ActivityMainBinding`，位于模块包下的`databinding`包内。

这个Binding类包含从模型属性到视图的所有绑定信息。

* 自定义Binding类的名称和位置

Binding类的名称可以通过`<data />`的属性来调整，如：

```xml
<!-- com.example.app.databinding.ContactItem -->
<data class="ContactItem">
  ...
</data>
```

这会生成名称为`ContactItem`的Binding类，此类位于模块包下的`databinding`包内。

如果需要将Binding类放置在其他包中：

1. 添加前缀`.`放置在模块包内，如：

    ```xml
    <!-- com.example.app.ContactItem -->
    <data class=".ContactItem">
      ...
    </data>
    ```

2. 直接给出完整包名，如：

    ```xml
    <!-- com.example.ContactItem -->
    <data class="com.example.ContactItem">
      ...
    </data>
    ```

#### 使用Binding类

在Activity中，可以通过以下代码建立绑定：

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    val binding: ActivityMainBinding =
        DataBindingUtil.setContentView(
            this, R.layout.activity_main
        )
    val user = User("user name")
    binding.user = user
}
```

在Fragment中，可以通过以下代码建立绑定：

```kotlin
override fun onCreateView(
    inflater: LayoutInflater,
    container: ViewGroup?,
    savedInstanceState: Bundle?
): View? {
    val binding: FragmentUserBinding = 
        FragmentUserBinding.inflate(
            inflater, container, false
        )
    binding.user = User("user name")
    return binding.root
}
```

### 被绑定的类

被绑定的类需要使用`@Bindable`标注需要绑定的属性：

```kotlin
class Validator {

    val user: User = User()

    @get:Bindable
    var nameError = ""
        set(value) {
            field = value
            notifyPropertyChanged(BR.name)
        }
        
    @Bindable("nameError")
    fun isInvalidName(): Boolean = nameError.isNotEmpty()
    
    fun validateName(): Boolean {
        if(user.name.get()?.isEmpty() == true) {
            
        }
    }
}
```
