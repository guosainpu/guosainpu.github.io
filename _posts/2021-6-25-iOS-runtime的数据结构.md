---
layout:     post
title:      iOS-runtime的数据结构
date:       2021-6-25
author:     "guosai"
tags:
    - iOS
    - objc
---

### 对象 & 类对象的关系

对象和类对象的整体关系图：

<center><img src="https://i.loli.net/2021/07/01/9cAEhLUD8PzRCfJ.png" width="1000"></center>

注：新版runtime中新增了rwe来优化rw

### 对象(objc_object)

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

以 x86_64 架构下为例，可以在 ObjC 源码中可以看到这样的定义：

```c
#   define ISA_MASK        0x00007ffffffffff8ULL
#   define ISA_MAGIC_MASK  0x001f800000000001ULL
#   define ISA_MAGIC_VALUE 0x001d800000000001ULL
#   define ISA_BITFIELD                                                        \
      uintptr_t nonpointer        : 1;                                         \
      uintptr_t has_assoc         : 1;                                         \
      uintptr_t has_cxx_dtor      : 1;                                         \
      uintptr_t shiftcls          : 44; /*MACH_VM_MAX_ADDRESS 0x7fffffe00000*/ \
      uintptr_t magic             : 6;                                         \
      uintptr_t weakly_referenced : 1;                                         \
      uintptr_t deallocating      : 1;                                         \
      uintptr_t has_sidetable_rc  : 1;                                         \
      uintptr_t extra_rc          : 8
#   define RC_ONE   (1ULL<<56)
#   define RC_HALF  (1ULL<<7)

union isa_t {
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    Class cls;
    uintptr_t bits;
#if defined(ISA_BITFIELD)
    struct {
        ISA_BITFIELD;  // defined in isa.h
    };
#endif
};
```

isa_t 是一个 union 的结构体，也就是说其中的 cls 和 bits 共用同一块内存地址空间，而 isa 总共会占据 64 位的内存空间。这个union可以表现为两种形式
1. raw isa：也就是没有结构体的部分，访问对象的 isa 会直接返回一个指向 cls 的指针，也就是在 iPhone 迁移到 64 位系统之前时 isa 的类型。
2. isa 不是指针，他是一个位段（field），但是其中也有 cls 的信息，只是其中关于类的指针都是保存在 shiftcls 中。我们下面要研究的就是这种情况

当处于位段情况下，不同bit具有不同的含义，其结构如下：

<center><img src="https://tva1.sinaimg.cn/large/008i3skNly1gs7giafpf4j30y40lgn1j.jpg" width="800"></center>

- **nonpointer**: 表示是否对 `isa` 指针开启指针优化
  - 0: 纯isa指针
  - 1: 不止是类对象地址,isa 中包含了类信息、对象的引用计数等
- **has_assoc**: 关联对象标志位，
  - 0: 没有
  - 1: 存在
- **has_cxx_dtor**: 该对象是否有 `C++` 或者 `Objc` 的析构器,如果有析构函数,则需要做析构逻辑, 如果没有, 则可以更快的释放对象
- **shiftcls**: 存储类指针的值。开启指针优化的情况下，在 `arm64` 架构中有 `33`位用来存储类指针
- **magic**: 用于调试器判断当前对象是真的对象还是没有初始化的空间
- **weakly_referenced**: 标志对象是否被指向或者曾经指向一个 `ARC` 的弱变量，没有弱引用的对象可以更快释放
- **deallocating**: 标志对象**是否正在释放内存**
- **has_sidetable_rc**: 当对象引用计数大于 `10` 时，则需要借用该变量存储进位
- **extra_rc**: 当表示该对象的引用计数值，实际上是引用计数值 **减 1**， 例如，如果对象的引用计数为 10，那么 extra_rc 为 9。如果引用计数大于 10， 则需要使用到上面的 `has_sidetable_rc`

isa的初始化：如上面所说，有两种方式的isa。

```c
inline void 
objc_object::initInstanceIsa(Class cls, bool hasCxxDtor)
{
    initIsa(cls, true, hasCxxDtor);
}

inline void 
objc_object::initIsa(Class cls, bool nonpointer, bool hasCxxDtor) 
{     
    if (!nonpointer) {
        isa = isa_t((uintptr_t)cls);
    } else {
        isa_t newisa(0);
        
        newisa.bits = ISA_MAGIC_VALUE;
        newisa.has_cxx_dtor = hasCxxDtor;
        newisa.shiftcls = (uintptr_t)cls >> 3;
        
        isa = newisa;
    }
}
```

shiftcls：

```c
// 将当前地址右移三位的主要原因是用于将 Class 指针中无用的后三位清除进而减小内存的消耗，因为类的指针要按照字节（8 bits）对齐内存，其指针后三位都是没有意义的 0。
newisa.shiftcls = (uintptr_t)cls >> 3;

// 通过掩码的方式获取类指针
objc_object::ISA() 
{
    return (Class)(isa.bits & ISA_MASK);
}
```

#### 对象的内存布局

在创建对象实例的时候，会根据其对应的类给实例对象分配内存，内存的构成是：isa_t + ivars（实例变量）。并且ivars不仅包含当前类的ivars，还包含继承链中所有父类的ivars，如下图。

ivars的内存布局是在编译时就已经决定了的，runtime不能动态的修改ivars，所以oc是不支持动态的添加实例变量的（动态创建的类除外）。但可以通过关联对象的方式实现，但关联对象也并没有改变ivars布局。

ivars的读写：当想要访问某个变量的时候，先通过对象指针，找到类指针，再找到类对象中ivars list信息，从ivars list信息中找到该变量的偏移地址值，通过该偏移地址值+对象首地址来定位目标变量的地址。

<center><img src="https://tva1.sinaimg.cn/large/008i3skNly1gs9gkhr3znj30u00yj74v.jpg" width="800"></center>

### 类对象(objc_class)

类对象（objc_class）继承自对象（objc_object），所以类对象中必然也有isa_t。除此以外还有其他三个字段。

1. ISA：指向该类的元类
2. superclass：指向该类的父类
3. cache：缓存，用于缓存方法指针
4. bits：也是一个类似filed的结构，其中不同bit存储了不同的信息，是我们主要研究的对象。

```c
struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
    ......
}    
```

继承关系：经典的集成关系图如下。

<center><img src="https://tva1.sinaimg.cn/large/008i3skNly1gs9honm4xhj30u00uyn03.jpg" width="800"></center>    

catch的大致工作流程如下：
1. 当前查找的 IMP 没有被缓存，调用 reallocate 方法进行创建-分配内存，然后使用 bucket 的 set 方法进行填充缓存。
2. 当前查找的 IMP 已经被缓存了，然后判断缓存容量是否已经达到 3/4 的临界点。
    - 如果已经到了临界点，则需要进行扩容，扩容大小为原来缓存大小的 2 倍。扩容后处于效率的考虑，会清空之前的内容，然后把当前要查找的 IMP 通过 bucket 的 set 方法缓存起来。
    - 如果没有到临界点，那么直接进行缓存。

<center><img src="https://tva1.sinaimg.cn/large/008i3skNly1gs9hmhdy5zj30u01060tj.jpg" width="800"></center>    

#### bits

bits的结构也是一个fileds的结构，class_rw_t * 指针存于第 [3, 47] 位，使用最后三位来存储关于当前类的其他信息

<center><img src="https://tva1.sinaimg.cn/large/008i3skNly1gsbwigtd66j314y078jrf.jpg" width="800"></center>

```
// objc_class 中的 data() 方法
class_data_bits_t bits;

class_rw_t *data() {
   return bits.data();
}

// class_data_bits_t 中的 data() 方法
uintptr_t bits;

class_rw_t* data() {
   return (class_rw_t *)(bits & FAST_DATA_MASK);
}
```

#### class_rw_t 和 class_ro_t

class_rw_t 和 class_ro_t 的指向关系如下:

<center><img src="https://tva1.sinaimg.cn/large/008i3skNly1gs9h51qyzuj314c0u0q4u.jpg" width="800"></center>

ObjC 类中的属性、方法还有遵循的协议等信息都保存在 class_rw_t 中，class_rw_t的内存是用calloc()分配的，所以rw在堆内存中，：

```c
// 可读可写
struct class_rw_t {
    // Be warned that Symbolication knows the layout of this structure.
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro; // 指向只读的结构体,存放类初始信息

    /*
     这三个都是二位数组，是可读可写的，包含了类的初始内容、分类的内容。
     methods中，存储 method_list_t ----> method_t
     二维数组，method_list_t --> method_t
     这三个二位数组中的数据有一部分是从class_ro_t中合并过来的。
     */
    method_array_t methods; // 方法列表（类对象存放对象方法，元类对象存放类方法）
    property_array_t properties; // 属性列表
    protocol_array_t protocols; //协议列表

    Class firstSubclass;
    Class nextSiblingClass;
    }
```

class_rw_t 中还有一个指向常量的指针 ro，其中存储了当前类在编译期就已经确定的属性、方法以及遵循的协议，ro在__DATA段中。

```c
struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;
    uint32_t reserved;

    const uint8_t * ivarLayout;

    const char * name;
    method_list_t * baseMethodList;
    protocol_list_t * baseProtocols;
    const ivar_list_t * ivars;

    const uint8_t * weakIvarLayout;
    property_list_t *baseProperties;
};
```

ro是在编译期确定的，rw是在运行时的realizeClass时确定的。

```c
// 从 class_data_bits_t 调用 data 方法，将结果从 class_rw_t 强制转换为 class_ro_t 指针
const class_ro_t *ro = (const class_ro_t *)cls->data();
// 在堆区创建一个rw
class_rw_t *rw = (class_rw_t *)calloc(sizeof(class_rw_t), 1);
// ro指向class_rw_o
rw->ro = ro;
// rw初始化
rw->flags = RW_REALIZED|RW_REALIZING;
cls->setData(rw);
```

#### list_array_tt

在 wr 中的 method_array_t、property_array_t、protocol_array_t，它们的父类是 list_array_tt ，list_array_tt 是一个二维数组的结构。

```c
/*
* list_array_tt<Element, List>
* Generic implementation for metadata that can be augmented by categories.
* 可以按类别扩展的元数据的通用实现。
*
* 实际应用：
* 1. class method_array_t : public list_array_tt<method_t, method_list_t>
* 2. class property_array_t : public list_array_tt<property_t, property_list_t>
* 3. class protocol_array_t : public list_array_tt<protocol_ref_t, protocol_list_t>
*
* Element is the underlying metadata type (e.g. method_t)
* Element 是基础元数据类型（例如: method_t）
*
* List is the metadata's list type (e.g. method_list_t)
* List 是元数据的列表类型（例如: method_list_t）
```

#### method_t

method_t是对方法、函数的封装，每一个方法对象就是一个method_t

```c
struct method_t {
    SEL name;  // 函数名
    const char *types;  // 编码（返回值类型，参数类型）
    IMP imp; // 指向函数的指针（函数地址）
};
```
- SEL: 方法编号。与C的函数指针不一样，函数指针直接保存了方法的地址，而SEL只是方法的编号，通过编号来寻找方法的地址
- types: types包含了函数返回值，参数编码的字符串。通过字符串拼接的方式将返回值和参数拼接成一个字符串，来代表函数返回值及参数。
- IMP: IMP代表函数的具体实现，存储的内容是函数地址

#### ivar_t

ivar_t 指明了成员变量的名称，类型，大小，还有一个重要的值就是偏移量，用于定位变量的位置。

```c
struct ivar_t {
    int32_t *offset;
    const char *name;
    const char *type;
    // alignment is sometimes -1; use alignment() instead
    uint32_t alignment_raw;
    uint32_t size;
};
```

#### property_t

property_t 包含了名称和属性

```c
typedef struct property_t *objc_property_t;

struct property_t {
    const char *name;
    const char *attributes;
};

```

#### protocol_t

```c
struct protocol_t : objc_object {
    const char *mangledName;            //协议名
    struct protocol_list_t *protocols;      //协议列表
    method_list_t *instanceMethods;         //实例方法
    method_list_t *classMethods;            //类方法
    method_list_t *optionalInstanceMethods;     //可选实例方法
    method_list_t *optionalClassMethods;          //可选类方法
    property_list_t *instanceProperties;            //实例属性
    uint32_t size;   // sizeof(protocol_t)
    uint32_t flags;
    // Fields below this point are not always present on disk.
    const char **_extendedMethodTypes;
    const char *_demangledName;
    property_list_t *_classProperties;              //类属性
};
```

#### category_t

```c
struct category_t {
    // 所属的类名，而不是Category的名字
    const char *name;
    // 所属的类，这个类型是编译期的类，这时候类还没有被重映射
    classref_t cls;
    // 实例方法列表
    struct method_list_t *instanceMethods;
    // 类方法列表
    struct method_list_t *classMethods;
    // 协议列表
    struct protocol_list_t *protocols;
    // 实例属性列表
    struct property_list_t *instanceProperties;
    // Fields below this point are not always present on disk.
    // 类属性列表
    struct property_list_t *_classProperties;
};
```

