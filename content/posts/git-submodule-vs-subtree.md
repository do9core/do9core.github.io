+++
title = "Git中的Submodule和Subtree"
date = "2022-08-17T16:01:00+08:00"
author = "do9core"
tags = ["Git"]
description = "对比Git的Submodule与Subtree"
readingTime = true
+++

## 前言

`Sub Module`和`Sub Tree`是Git提供的工具，虽然都是进行多项目管理功能，但是有一些区别和不同的适用场景。

最近使用到Grpc，需要共享protoc定义文件。
因为需要同时给客户端和服务端使用，为了确保两端能使用到确定的版本，protoc文件需要单独的仓库来进行版本管理，这时候就需要在开发仓库中引入protoc定义仓库，形成依赖关系。

submodule和subtree都能实现这个管理逻辑，但是具体还是有一些区别，我们首先了解一下submodule和subtree的基本功能。

```plain
下文所有的【当前仓库】，均指代父级仓库，而【submodule/subtree】均指子级仓库

当前仓库
  - submodule1
  - submodule2
  - subtree1
  - subtree2
```

### 创建Submodule

通过Git命令可以很简单地创建submodule：

```shell
# 如果不指定DIR，则会创建一个名为仓库名称的目录保存submodule的内容
git submodule add <submodule Repo URL> <DIR>
```

此命令执行后，submodule会被创建到`<DIR>`指定的目录下，同时会生成一个名为`.gitmodules`（隐藏）文件，用于记录submodule的仓库地址及版本（commit）

因为创建了新文件，此时需要在当前仓库内进行一次commit，才能将导入的submodule提交到当前仓库内。

但是此提交并不会将submodule内到文件同时提交到本仓库中，它的功能类似于linux里面的链接，仅仅是建立了一个metadata指向submodule所在的地址和版本。

### 获取Submodule

#### 初次获取

在默认情况下，使用`git clone <Repo URL>`命令，并不会同时获取submodule的内容，克隆操作完毕后，submodule名称对应的目录是空的，不包含任何内容。

如果希望在克隆仓库时，同时获取所有submodule的内容，需要使用：

```shell
git clone <Repo URL> --recurse-submodules
```

或者，**在克隆完当前仓库后**，执行：

```shell
git submodule init
git submodule update
```

来获取submodule的内容

#### 获取变更

在submodule内执行通常的git操作(`git checkout/switch`)，然后在当前仓库执行：

```shell
git submodule update
```

即可，如果需要提交更改，直接在当前仓库内`add`并`commit`即可

### 向Submodule提交更新

submodule作为附属模块，其本身并不能感知到当前仓库

因为submodule本身是一个完整的git仓库，所以提交等操作和通常git操作没有区别，只需要记得切换到submodule目录下执行操作即可。

但是需要注意一点，当前仓库等submodule有变更后，如从远程拉取，或者在本地进行了修改并且已经提交后，需要在当前仓库中进行commit来更新`.gitmodules`引用的submodule版本（即commit）。

### 移除Submodule

安全地移除submodule，需要执行以下命令：

```shell
# 如果submodule中有本地变更，使用--force可以放弃变更并移除submodule配置
git submodule deinit <submodule DIR> [--force]
```

此命令会移除`.git/config`下的submodule配置，然后执行：

```shell
git rm <submodule DIR>
```

此命令会移除submodule目录，并同时移除`.gitmodules`中submodule的配置。
执行以上两步操作后，当前仓库的submodule数据已经清理完毕，此时再进行一次提交（commit）即可完成submodule的移除。

## Subtree

### 创建Subtree

首先将subtree添加到本地仓库的远程仓库列表中：

```shell
# -f 选项在添加仓库后执行fetch
# 也可以不加-f选项，单独执行git fetch <subtree NAME> <subtree BRANCH>
git remote add -f <subtree NAME> <subtree repo URL>
```

将此remote添加为subtree：

```shell
# 可以添加--squash选项来合并远程的多次commit
git subtree add --prefix=<subtree DIR> <subtree NAME> <subtree BRANCH>
```

### 获取Subtree

与Submodule不同，更新Subtree的操作需要使用独立命令`git subtree`：

```shell
# 获取subtree的远程仓库变更
git fetch <subtree NAME> <subtree BRANCH>
# 可以添加--squash选项，同上
git subtree pull --prefix=<subtree DIR> <subtree NAME>
```

### 向Subtree提交更新

同样地，提交Subtree的变更也需要使用独立命令`git subtree`：

```shell
git subtree push --prefix=<subtree DIR> <subtree NAME> <subtree BRANCH>
```

## 对比

### 实现方式

* submodule通过在当前仓库创建`.gitmodules`文件保存了的指向，相当于一个快捷方式（link）
* subtree则是将仓库的内容，直接添加到当前仓库，除此之外不会添加类似`.gitmodules`的额外文件，相当于一份拷贝（copy）。

### 操作

* submodule的操作和通常git仓库操作一样，只需要进入submodule所在目录，进行与其他仓库一致的操作即可
* subtree的操作则需要通过独立的`git subtree`命令来进行（详情参考上文）

### 仓库权限

两者不同的实现方式，导致对权限的需求也有所差异：

对于当前仓库的项目

* 如果使用submodule，那么开发者需要同时拥有**当前仓库**以及**所有submodule**的访问权限，才可以完成完整的构建过程，因为submodule本身不保存内容，需要开发者自行拉取。
* 如果使用subtree，那么开发者只需要拥有**当前仓库**的访问权限，即可完成整个构建流程，因为subtree的内容会被直接提交到当前仓库内部，当前仓库拥有subtree指定版本的完整拷贝。

## 总结

文章开头提到的Grpc项目：

* 因为我本身完全一个人开发前后端，拥有所有仓库的访问权限
* 同时仓库也不需要和其他人共享
* 开发过程中，可能需要频繁修改子项目（protoc定义）的内容

为了便于各个仓库的版本管理（尤其是protoc定义的管理），我选择使用submodule方案。

虽然submodule和subtree同为Git提供的多项目管理方案，但是也需要根据其实现方式和实际场景来决定要使用哪种方案。

---

## 参考资料

1. [Git中submodule的使用](https://zhuanlan.zhihu.com/p/87053283)
2. [git subtree用法](https://www.cnblogs.com/jingwhale/p/6054492.html)
3. [git subtree用法](http://cssor.com/git-subtree-usage.html)
