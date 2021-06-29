---
layout:     post
title:      iOS-objc_init
date:       2021-6-22
author:     "guosai"
tags:
    - iOS
    - objc
---

### dyld & objc_init

前面我们分析到dyld负责加载App主二进制所依赖的动态库，并对APP、动态库做了一些链接和初始化工作。然后针对有初始化方法的动态库，dyld会调用它们的初始化方法，这里就包括runtime的初始化方法objc_init。这个过程如下图：

<center><img src="https://tva1.sinaimg.cn/large/008i3skNly1grtdbw0108j31040lxtdi.jpg" width="1000"></center>

### _objc_init

dyld_init是runtime的初始化方法，主要做了以下事情：

1. 环境初始化
2. read_image:对各个Image中Class相关数据的初始化和处理
3. load_image:调用+load方法

```c
    
    environ_init();
    tls_init();
    static_init();       // 运行 C++ 静态构造函数
    runtime_init();      
    exception_init();    // 初始化异常处理，注册异常崩溃的回调，当发生崩溃时，会来到 _objc_terminate 函数里面。
    cache_init();
    _imp_implementationWithBlock_init();

	// *重点方法*
    _dyld_objc_notify_register(&map_images, load_images, unmap_image);
```

### _dyld_objc_notify_register

objc_init 的核心逻辑是 _dyld_objc_notify_register 方法。objc 通过 _dyld_objc_notify_register 注册了三个回调方法：mapped, init, unmapped。

_dyld_objc_notify_register 是在 dyld 中实现的。objc 调用 _dyld_objc_notify_register后，dyld 会持有 mapped，init，unmapped 三个方法。然后执行如下逻辑：

1. dyld对当前已经mapped的Image，调用mapped方法
2. 对当前已经初始化的Image，调用init方法
3. 如果后续有map, init, unmap Image的行为（例如调用dyopen()），就再调用响应的 mapped，init，unmapped

```
// Note: only for use by objc runtime
// Register handlers to be called when objc images are mapped, unmapped, and initialized.
// Dyld will call back the "mapped" function with an array of images that contain an objc-image-info section.
// Those images that are dylibs will have the ref-counts automatically bumped, so objc will no longer need to
// call dlopen() on them to keep them from being unloaded.  During the call to _dyld_objc_notify_register(),
// dyld will call the "mapped" function with already loaded objc images.  During any later dlopen() call,
// dyld will also call the "mapped" function.  Dyld will call the "init" function when dyld would be called
// initializers in that image.  This is when objc calls any +load methods in that image.
//
void _dyld_objc_notify_register(_dyld_objc_notify_mapped    mapped,
                                _dyld_objc_notify_init      init,
                                _dyld_objc_notify_unmapped  unmapped);

void _dyld_objc_notify_register(_dyld_objc_notify_mapped    mapped,
                                _dyld_objc_notify_init      init,
                                _dyld_objc_notify_unmapped  unmapped)
{
    dyld::registerObjCNotifiers(mapped, init, unmapped);
}

void registerObjCNotifiers(_dyld_objc_notify_mapped mapped, _dyld_objc_notify_init init, _dyld_objc_notify_unmapped unmapped)
{
    // record functions to call
    sNotifyObjCMapped   = mapped;
    sNotifyObjCInit     = init;
    sNotifyObjCUnmapped = unmapped;

    // call 'mapped' function with all images mapped so far。
    // 对当前已经mapped的Image，调用mapped方法
    try {
        notifyBatchPartial(dyld_image_state_bound, true, NULL, false, true);
    }
    catch (const char* msg) {
        // ignore request to abort during registration
    }

    // <rdar://problem/32209809> call 'init' function on all images already init'ed (below libSystem)
    // 对当前已经初始化的Image，调用init方法
    for (std::vector<ImageLoader*>::iterator it=sAllImages.begin(); it != sAllImages.end(); it++) {
        ImageLoader* image = *it;
        if ( (image->getState() == dyld_image_state_initialized) && image->notifyObjC() ) {
            dyld3::ScopedTimer timer(DBG_DYLD_TIMING_OBJC_INIT, (uint64_t)image->machHeader(), 0, 0);
            (*sNotifyObjCInit)(image->getRealPath(), image->machHeader());
        }
    }
}

```

### read_image

map_image的核心方法是read_image，大概有350行代码，核心功能是对非懒加载类的初始化，大概做了以下事情：

1. readClass 主要是读取类（处理类的地址和名称，还没有 data 数据），并通过 addNamedClass 方法把类添加到已经创建好的 gdb_objc_realized_classes 哈希表中，方便以后查找类对象
2. 执行一大堆的初始化和修复任务
    - 创建哈希表
    - 修复selector
    - 修复类
    - 修复消息
    - ......
3. 把一大堆信息存入map
    - 类信息
    - 方法名
    - protocol
    - ......
4. 通过 realizeClassWithoutSwift 方法来初始化类（主要是对ro, rw, rwe的处理）
    - methodizeClass 方法中实现类的方法、属性、协议的序列化
    - attachCategories 方法中实现类以及分类的数据加载

### 懒加载类&非懒加载类

1. 非懒加载类：此时就要初始化的类（会对其调用realizeClassWithoutSwift），实现了+load方法的类都是非懒加载类。
2. 懒加载类：运行时第一次访问时才初始化的类。

<center><img src="https://tva1.sinaimg.cn/large/008i3skNly1grtdz1yue2j30z00u0ac0.jpg" width="700"></center>

### realizeClassWithoutSwift

realizeClassWithoutSwift 主要做了以下三件事情：

1. 读取 data 数据，并设置 ro 、rw
    - 读取 data 数据，并将其强转为 ro，以及初始化 rw，然后将 ro 拷贝一份到 rw 的 ro
    - ro 表示 read only，其在编译期的时候就已经确定了内存，包含了类的名称、方法列表、协议列表、属性列表和成员变量列表的信息。由于它是只读的，确定了之后就不会发生变化，所以属于 干净的内存(Clean Memory)
    - rw 表示 read Write，由于 OC 的动态性，所以可能会在运行时动态往类中添加属性、方法和协议
    - rwe 表示 read Write ext，是对rw做了进一步的改进。 rw 中只有 10% 左右的类真正更改了它们的方法、属性等，所以新增加了 rwe，即是类的额外信息，rw 和 rwe 都属于 脏内存(dirty memory)
2. 递归调用 realizeClassWithoutSwift 完成类的继承链关系
    - 递归调用 realizeClassWithoutSwift，**设置父类和元类**
    - 分别将父类和元类赋值给 class 的 superclass 和 classIsa
3. 调用 methodizeClass 方法，读取方法列表(包括分类)、协议列表、属性列表，然后赋值给 rw，最后返回 cls

### load_images

load_images 函数是对 load 方法的加载和调用，接下来我们就来看看底层是怎么对 load 处理的。

```c
void load_images(const char *path __unused, const struct mach_header *mh)
{
    // 如果这里没有+load方法，则返回时不带锁。
    if (!hasLoadMethods((const headerType *)mh)) return;

    recursive_mutex_locker_t lock(loadMethodLock);

    {
        mutex_locker_t lock2(runtimeLock);
        // 准备，查找 load 方法
        prepare_load_methods((const headerType *)mh);
    }

    // 调用 load 方法
    call_load_methods();
}
```

进到 prepare_load_methods 可以看到系统是如何加载 load 方法的。遵循的原则：先去加载父类的，然后再加载本类，之后再去加载分类的 load 方法。

```c
void prepare_load_methods(const headerType *mhdr)
{
    size_t count, i;
    runtimeLock.assertLocked();
    // 获取非懒加载类列表
    classref_t *classlist = 
        _getObjc2NonlazyClassList(mhdr, &count);
    for (i = 0; i < count; i++) {
        // 循环遍历去加载非懒加载类的 load 方法到 loadable_classes
        schedule_class_load(remapClass(classlist[i]));
    }
    // 获取非懒加载分类列表
    category_t **categorylist = _getObjc2NonlazyCategoryList(mhdr, &count);
    for (i = 0; i < count; i++) {
        category_t *cat = categorylist[i];
        Class cls = remapClass(cat->cls);
        if (!cls) continue;  // category for ignored weak-linked class
        if (cls->isSwiftStable()) {
            _objc_fatal("Swift class extensions and categories on Swift "
                        "classes are not allowed to have +load methods");
        }
        // 如果本类没有初始化就去初始化
        realizeClassWithoutSwift(cls);
        assert(cls->ISA()->isRealized());
        
        // 循环遍历去加载非懒加载分类的 load 方法到 loadable_categories
        // 和非懒加载类差不多，就是数组不一样
        add_category_to_loadable_list(cat);
    }
}

static void schedule_class_load(Class cls)
{
    if (!cls) return;
    assert(cls->isRealized());  // _read_images should realize

    if (cls->data()->flags & RW_LOADED) return;

    // 常规操作，递归调用父类加载 load 方法
    schedule_class_load(cls->superclass);
    // 将 load 方法加载到 loadable_classes
    add_class_to_loadable_list(cls);
    cls->setInfo(RW_LOADED); 
}

void add_class_to_loadable_list(Class cls)
{
    IMP method;
    loadMethodLock.assertLocked();
    // 获取 load 方法的 imp
    method = cls->getLoadMethod();
    // 如果没有 load 方法直接返回
    if (!method) return; 
    
    if (PrintLoading) {
        _objc_inform("LOAD: class '%s' scheduled for +load", 
                     cls->nameForLogging());
    }
    // 扩容
    if (loadable_classes_used == loadable_classes_allocated) {
        loadable_classes_allocated = loadable_classes_allocated*2 + 16;
        loadable_classes = (struct loadable_class *)
            realloc(loadable_classes,
                              loadable_classes_allocated *
                              sizeof(struct loadable_class));
    }
    // loadable_classes 添加 load 方法
    loadable_classes[loadable_classes_used].cls = cls;
    loadable_classes[loadable_classes_used].method = method;
    loadable_classes_used++;
}
```

load方法的调用

```c
void call_load_methods(void)
{
    static bool loading = NO;
    bool more_categories;
    loadMethodLock.assertLocked();

    // 保证只调用一次
    if (loading) return;
    loading = YES;

    void *pool = objc_autoreleasePoolPush();
    // do while 循环调用 load 方法
    do {
        // 1.重复调用非懒加载类的 load，直到没有更多的
        while (loadable_classes_used > 0) {
            call_class_loads();
        }

        // 2.调用非懒加载分类的 load 方法，和非懒加载类差不多
        more_categories = call_category_loads();

        // 3. 如果有类或更多未尝试的类别，则运行更多 load
    } while (loadable_classes_used > 0  ||  more_categories);

    objc_autoreleasePoolPop(pool);
    loading = NO;
}

static void call_class_loads(void)
{
    int i;
    // 取出 loadable_classes
    struct loadable_class *classes = loadable_classes;
    int used = loadable_classes_used;
    loadable_classes = nil;
    loadable_classes_allocated = 0;
    loadable_classes_used = 0;
    
    // 调用保存在 loadable_classes 里的 load 方法
    for (i = 0; i < used; i++) {
        Class cls = classes[i].cls;
        load_method_t load_method = (load_method_t)classes[i].method;
        if (!cls) continue; 

        if (PrintLoading) {
            _objc_inform("LOAD: +[%s load]\n", cls->nameForLogging());
        }
        // 发送 load 消息
        (*load_method)(cls, SEL_load);
    }
    
    // 释放内存
    if (classes) free(classes);
}

```

总结看 load 的调用，可以得出，load 的调用顺序是：**父类->本类->分类。**

### 加载流程图

<center><img src="https://tva1.sinaimg.cn/large/008i3skNly1grte95wrr1j324e0u077y.jpg" width="1000"></center>

