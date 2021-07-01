---
layout:     post
title:      iOS-runtime的数据结构
date:       2021-6-25
author:     "guosai"
tags:
    - iOS
    - objc
---

### 对象 & 类对象

对象和类对象的整体关系图：

<center><img src="https://i.loli.net/2021/07/01/9cAEhLUD8PzRCfJ.png" width="1000"></center>

注：新版runtime中新增了rwe来优化rw

#### 对象(objc_object)

```c
struct objc_object {
private:
    isa_t isa;
......    
}    
```

我们平时用的所有对象都是id类型的，在runtime中 id 就是 objc_object 结构体，而 objc_object 的内容是一个 isa_t 的八字节数据，这个 isa_t 是一个共用体。并包含了以下信息：

1. 关于 isa 操作的相关方法：提供了 isa 操作相关的一些方法，比如通过 objc_object 这个结构体来获取它的 isa 所指向的类对象，包括通过类对象的 isa 指针获取它的元类对象一些遍历的方法
2. 弱引用相关方法：比如说，标记一个对象，它是否曾经有过弱指针
3. 关联对象相关方法：比说，这个对象，为它设置了一些关联属性，关联属性的一些方法也体现在 objc_object 结构体当中
4. 内存管理相关：比如， 在 MRC 下面经常使用到的 retain，relese，包括 ARC，MRC 下面都可以用到的 艾特aotorelesepool

#### isa_t

