+++
title = "Shell中的重定向和管道"
date = "2022-05-25T15:10:00+08:00"
author = "do9core"
tags = ["Linux", "Shell"]
description = "总结 | 和 > 的使用"
readingTime = true
+++

## 基本概念

### Linux命令执行过程

```goat
+-------------+ stdin   +---------------+  stdout  +-------------+
| File/Device +-------->| Shell Command +--------->| File/Device |
+-------------+         +-------+-------+          +-------------+
                                |stderr
                                v
                        +---------------+
                        |  File/Device  |
                        +---------------+
```

Linux中，万物皆文件，执行命令的过程也是如此。  
将文件读入标准输入流，执行命令；如果执行成功，将结果输出到标准输出流；如果执行失败，则将结果输出到标准错误流。

### 文件描述符（File Descriptor）

Linux会为打开的文件分配文件描述符，通过描述符对文件进行操作。  
为了完成上述流程，系统本身会占用3个文件描述符0，1和2，分别对应：

* 0 standard input 即**标准输入**
* 1 standard output 即**标准输出**
* 2 standard error 即**标准错误**

默认情况下，标准输入为键盘，标准输出和标准错误为监视器，用户可以通过重定向方式来更改它们

## 重定向

### 重定向方向和基本用法

Linux系统允许用户通过`>`和`<`分别对输入和输出进行重定向：

* `<` 输入重定向。`cmd < input.file` 即，将`input.file`重定向为`cmd`的stdin
* `>` 输出重定向。`cmd > output.file` 即，将`cmd`的stdout重定向到`output.file`
* `>>` 输出重定向（追加）。`cmd >> output.file`即，将`cmd`到stdout重定向到`output.file`，但是采用追加方式

`>`和`>>`的区别在于，`>`会覆盖文件，`>>`只会将新内容追加到输出文件中

### &的作用

通常，在命令的最后添加`&`，可以使其在后台运行，不占用当前会话  
除此之外，`&`在重定向中有一些特殊作用：

* `&n`, 其中n为数字，表示描述符n代表的文件，例如`&1`代表stdin
* `>&-`，表述关闭输出
* `<&-`，表述关闭输入

### 特殊用法

#### 丢弃输出

Linux系统允许用户抛弃某些输出，用户可以通过将输出重定向到`/dev/null`来抛弃输出  
`/dev/null`也被称为`Bit Bucket`或黑洞，参考[What is the Unix/Linux “bit bucket”?](https://alvinalexander.com/linux-unix/what-is-bit-bucket-dev-null/)

如果我们希望将stderr的输出丢弃：

```shell
cmd 2> /dev/null
```

#### 重定向到描述符

通常情况下，直接使用`cmd > output.file`，只会将成功结果输出到`output.file`中，错误仍然会显示在shell内，此时如果希望将错误也输出到文件，可以使用：

```shell
cmd > output.file 2>&1
```

因为`&1`在默认情况下表示stdout，所以`2>&1`是将stderr重定向到stdout上，而在前一步`cmd > output.file`中，stdout已被重定向到`output.file`，所以stderr也被重定向到`output.file`

思考：如果将命令顺序调整会发生什么？

```shell
cmd 2>&1 > output.file
```

在没有其他操作的情况下，这个命令和直接执行`cmd > output.file`效果是一样的，因为在第一步`cmd 2>&1`中，stderr被重定向到stdin，即监视器，第二步中，将stdout重定向到`output.file`，错误仍然会在监视器显示，但是结果会被输出到文件中

#### exec命令与重定向

## 管道

了解了重定向和输入、输出后，再理解管道就显得非常容易。
管道使用`|`描述，实际使用方法为`cmd1 | cmd2 | cmd3`，管道的意义是将上一个命令的stdout作为下一个命令的stdin，实现连续执行命令的效果：

```goat
+------+ stdout/stdin  +------+ stdout/stdin  +------+
| cmd1 +-------------->| cmd2 +-------------->| cmd3 |
+------+               +------+               +------+
```

遇到执行错误的情况，会发生什么？
管道只能处理stdout的数据，无法处理stderr的数据，即便前一个命令发生错误，后续的命令仍然会被执行，但是只会处理stdout中输出的数据。

## 总结

懒得写了，其实整篇都在总结。
