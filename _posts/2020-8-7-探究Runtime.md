---
title: 探究OC Runtime
tags: iOS OC
---

### Runtime理解

OC是基于C的，在C的基础上扩展出了面向对象的能力，支持其面向对象能力的核心就是OC的runtime机制。OC中我们编写的类，最终按照某种数据结构存储在可执行文件的__DATA段中，然后可执行文件被操作系统加载到进程空间，此时进程空间中就有了各个类和函数的信息。runtime是一个动态链接库，在程序执行过程中，我们通过runtime提供的各种能力，来访问、操作、读写各个类、方法、实例对象。

**总结来说：OC的面向对象能力 = 类和方法的数据结构（数据结构也由runtime定义）+ 算法（runtime提供的各种操作）。**

#### 举例

我们写一个简单的OC程序

```c
#import <Foundation/Foundation.h>

@interface testClass : NSObject

@property(nonatomic, assign) NSInteger testInt;
-(void)testMethod;

@end

@implementation testClass

-(void)testMethod
{
	return;
}

@end

int main(int argc, const char * argv[])
{
	testClass *obj = [[testClass alloc] init];
	[obj testMethod];

	return 0;
}
```

我们生成二进制文件后，可以看到。

在__DATA,objc_data中，存储了类的信息。一共五个字段，比较中的字段有3个，isA指向该类的元类，super class指向该类的父类，data指向calss_ro_t字段，该字段进一步包含了更丰富的信息。

<center><img src="https://tva1.sinaimg.cn/large/007S8ZIlly1ggjmn1d0iqj319t0u0jvl.jpg" width="1000"></center>

这里data指向 0000000100001108，该地址在__DATA,ojct_const中，该段中存储了一些不可变信息，包括方法列表、属性列表、协议列表等。这里我们看下0000000100001108地址的值，这个数据结构中包含了类的类名，实例对象的size，方法列表、变量列表的地址等信息。其中base method又指向0000000100001078，我们再看下0000000100001078（还在该段中）

<center><img src="https://tva1.sinaimg.cn/large/007S8ZIlly1ggjmusoz1zj31dc0u00x3.jpg" width="1000"></center>

这块数据描述了testClass中所有方法的信息，包括方法的名称、类型、起始地址等。

<center><img src="https://tva1.sinaimg.cn/large/007S8ZIlly1ggjmwt8yq3j31i00u0q7f.jpg" width="1000"></center>

**可以看到：我们用OC所写的程序的各种信息，换了中描述方式（组织成新的数据结构格式）被存储到可执行文件当中了，后面在执行的时候，就可以通过解析这些数据结构来获取信息，帮助程序更好的执行**。例如，我们在程序中调用了testMethod方法，如果这是一个C程序，那么在汇编代码中，就会直接call到该函数的地址上。但这里不是，这里是去调用了objc_msgSend方法（在runtime中实现），告诉objc_msgSend程序想要的方法名是testMethod(通过rsi寄存器传参)，然后objc_msgSend帮我们去找到testMethod的入口地址（通过分析刚才提到的各种数据结构）并跳转执行。

<center><img src="https://tva1.sinaimg.cn/large/007S8ZIlly1ggjn37ronlj31ke0osdiv.jpg" width="1000"></center>

我们通过这里例子，简单的感受了一些runtime机制，后面我们就更深入的对runtime进行一些探究。此时我们也知道了，探究runtime，其实就是搞懂runtime所定义的各种数据模型，以及利用这些数据模型所进行的各种操作。


