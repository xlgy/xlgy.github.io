---
layout:     post
title:      iOS 深入探究runtime
subtitle:   iOS 深入探究runtime
date:       2020-06-15
author:     LXY
header-img: img/home1.jpeg
catalog: true
tags:
    - iOS
---


本文篇幅比较长，创作的目的为了自己日后温习知识所用，希望这篇文章能对你有所帮助。如发现任何有误之处，肯请留言纠正，谢谢。



### 一、深入代码理解 instance、class object、metaclass
![](https://github.com/xlgy/xlgy.github.io/blob/master/_posts/image/runtime1.png?raw=true)


**1、instance对象实例**

我们经常使用id来声明一个对象，那id的本质又是什么呢？查看objc/objc.h文件
```
/// An opaque type that represents an Objective-C class.
typedef struct objc_class *Class;

/// Represents an instance of a class.
struct objc_object {
    Class _Nonnull isa  OBJC_ISA_AVAILABILITY;
};

/// A pointer to an instance of a class.
typedef struct objc_object *id;
```
我们创建的一个对象或实例其实就是一个struct objc_object结构体，而我们常用的id也就是这个结构体的指针。

这个结构体只有一个成员变量，这是一个Class类型的变量isa，也是一个结构体指针，**isa指针就指向对象所属的类**。

***一个 NSObject 对象占用多少内存空间？***
一个NSObject实例对象只有一个isa指针，所以一个isa指针的大小，他在64位的环境下占8个字节，在32位环境上占4个字节。

```
 NSObject *obj = [[NSObject alloc] init];
 NSLog(@"class_getInstanceSize--%zd", class_getInstanceSize([obj class]));
```
输出结果：
```
class_getInstanceSize--8
```

**2、class object（类对象）/metaclass（元类）**

看结构体objc_class的定义

```
struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;

#if !__OBJC2__
    Class super_class                                        OBJC2_UNAVAILABLE;
    const char *name                                         OBJC2_UNAVAILABLE;
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list *ivars                             OBJC2_UNAVAILABLE;
    struct objc_method_list **methodLists                    OBJC2_UNAVAILABLE;
    struct objc_cache *cache                                 OBJC2_UNAVAILABLE;
    struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;
#endif

} OBJC2_UNAVAILABLE;
/* Use `Class` instead of `struct objc_class *` */
```



- ```Class superclass```;——用于获取父类，也就是父类对象，它也是一个Class类型
- ```cache_t cache```;——是方法缓存
- ```class_data_bits_t bits```;——用于获取类的具体信息，看到bits
- ```class_rw_t *data()```函数，该函数的作用就是获取该类的可读写信息，通过class_data_bits_t的bits.data()方法获得，```class_rw_t```后面会介绍

```
class_rw_t* data() {
        return (class_rw_t *)(bits & FAST_DATA_MASK);
}
```

该结构体的第一个成员变量也是isa指针，这就说明了Class本身其实也是一个对象，我们称之为**类对象**。类对象中的元数据存储的都是如何创建一个实例的相关信息，那么类对象和类方法应该从哪里创建呢？就是从isa指针指向的结构体创建，类对象的isa指针指向的我们称之为元类(metaclass)，元类中保存了创建类对象以及类方法所需的所有信息。

**3、isa指针与superclass相关逻辑图**


![isa逻辑图](https://github.com/xlgy/xlgy.github.io/blob/master/_posts/image/runtime2.png?raw=true)


**4、总结 + 代码校验**


- 对象的isa指针指向类对象；
- 类对象的isa指针指向元类对象，和类同名；
- 元类的isa指针指向跟根元类 NSObject；
- 根元类 NSObject的isa指针指向自己；
- 类对象的Superclass还是类对象
- 元类对象的Superclass还是元类对象；
- 根元类NSObject元类 的(Superclass)是 NSObject类对象；
- NSObject类对象的(Superclass)是nil；


isa验证

```
    NSString *string = @"字符串";
    Class class1 = object_getClass(string);//NSString类对象
    Class metaClass = object_getClass(class1);//NSString元类
    Class rootMetaClass = object_getClass(metaClass);//根元类
    Class rootRootMetaClass = object_getClass(rootMetaClass);//根元类
    NSLog(@"%p 实例对象 ",string);
    NSLog(@"%p 类 %@",class1,NSStringFromClass(class1));
    NSLog(@"%p 元类 %@",metaClass,NSStringFromClass(metaClass));
    NSLog(@"%p 根元类 %@",rootMetaClass,NSStringFromClass(rootMetaClass));
    NSLog(@"%p 根根元类 %@",rootRootMetaClass,NSStringFromClass(rootRootMetaClass));
    
    Class rootMetaClass_superclass = rootMetaClass.superclass;//根元类的superclass
    NSLog(@"根根元类的superclass:%@",NSStringFromClass(rootMetaClass_superclass));
```

输出结果：

```
0x102d48078 实例对象 
0x1d80e3d10 类 __NSCFConstantString
0x1d80e3cc0 元类 __NSCFConstantString
0x1d80c66c0 根元类 NSObject
0x1d80c66c0 根根元类 NSObject
根根元类的superclass:NSObject

```

superclass验证

```
    NSString *string = @"字符串";
    Class class1 = object_getClass(string);//NSString类对象
    Class class2 = class1.superclass;
    NSLog(@"%@ 的superclass是 %@",NSStringFromClass(class1),NSStringFromClass(class2));
    Class class3 = class2.superclass;
    NSLog(@"%@ 的superclass是 %@",NSStringFromClass(class2),NSStringFromClass(class3));
    Class class4 = class3.superclass;
    NSLog(@"%@ 的superclass是 %@",NSStringFromClass(class3),NSStringFromClass(class4));
    Class class5 = class4.superclass;
    NSLog(@"%@ 的superclass是 %@",NSStringFromClass(class4),NSStringFromClass(class5));
    Class class6 = class5.superclass;
    NSLog(@"%@ 的superclass是 %@",NSStringFromClass(class5),NSStringFromClass(class6));
```
输出结果：
```
 __NSCFConstantString 的superclass是 __NSCFString
 __NSCFString 的superclass是 NSMutableString
NSMutableString 的superclass是 NSString
NSString 的superclass是 NSObject
NSObject 的superclass是 (null)
```

### 二、class_rw_t 与 class_ro_t 

**1、class_ro_t 一"码"当先：**

```
struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;
#ifdef __LP64__
    uint32_t reserved;
#endif

    const uint8_t * ivarLayout;
    
    const char * name;
    method_list_t * baseMethodList;
    protocol_list_t * baseProtocols;
    const ivar_list_t * ivars;

    const uint8_t * weakIvarLayout;
    property_list_t *baseProperties;

    method_list_t *baseMethods() const {
        return baseMethodList;
    }
};
```

- uint32_t instanceSize;——instance对象占用的内存空间
- const char * name;——类名
- const ivar_list_t * ivars;——类的成员变量列表


class_ro_t存储了当前类在编译期就已经确定的属性、方法以及遵循的协议，里面是没有分类的方法的。那些运行时添加的方法将会存储在运行时生成的class_rw_t中。
ro即表示read only，是无法进行修改的。


**2、class_rw_t 一"码"当先：**

```
// 可读可写
struct class_rw_t {
    // Be warned that Symbolication knows the layout of this structure.
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro; // 指向只读的结构体,存放类初始信息

    /*
     这三个都是二位数组，是可读可写的，包含了类的初始内容、分类的内容。
     这三个二位数组中的数据有一部分是从class_ro_t中合并过来的。
     */
    method_array_t methods; // 方法列表（类对象存放对象方法，元类对象存放类方法）
    property_array_t properties; // 属性列表
    protocol_array_t protocols; //协议列表

    Class firstSubclass;
    Class nextSiblingClass;
    
    //...
    }


```

**3、class_rw_t生成时机**

class_rw_t生成在运行时，在编译期间，class_ro_t结构体就已经确定，objc_class中的bits的data部分存放着该结构体的地址。在runtime运行之后，具体说来是在运行runtime的realizeClass 方法时，会生成class_rw_t结构体，该结构体包含了class_ro_t，并且更新data部分，换成class_rw_t结构体的地址。

类的realizeClass运行之前：

![](https://github.com/xlgy/xlgy.github.io/blob/master/_posts/image/runtime3.png?raw=true)


然后在加载 ObjC 运行时的过程中在 realizeClass 方法中：
- 从 class_data_bits_t 调用 data 方法，将结果从 class_rw_t 强制转换为 class_ro_t 指针
- 初始化一个 class_rw_t 结构体
- 设置结构体 ro 的值以及 flag
- 最后设置正确的 data。

```
const class_ro_t *ro = (const class_ro_t *)cls->data();
class_rw_t *rw = (class_rw_t *)calloc(sizeof(class_rw_t), 1);
rw->ro = ro;
rw->flags = RW_REALIZED|RW_REALIZING;
cls->setData(rw);
```

但是，在这段代码运行之后 class_rw_t 中的方法，属性以及协议列表均为空。这时需要 realizeClass 调用 methodizeClass 方法来将类自己实现的方法（包括分类）、属性和遵循的协议加载到 methods、 properties 和 protocols 列表中。

realizeClass 方法执行过后的类所占用内存的布局：

![](https://github.com/xlgy/xlgy.github.io/blob/master/_posts/image/runtime4.png?raw=true)


细看两个结构体的成员变量会发现很多相同的地方，他们都存放着当前类的属性、实例变量、方法、协议等等。区别在于：class_ro_t存放的是编译期间就确定的；而class_rw_t是在runtime时才确定，它会先将class_ro_t的内容拷贝过去，然后再将当前类的分类的这些属性、方法等拷贝到其中。所以可以说class_rw_t是class_ro_t的超集，当然实际访问类的方法、属性等也都是访问的class_rw_t中的内容。


**4、method_t**

上面我们剖析了class_rw_t、class_ro_t这两个重要部分的结构，并且主要关注了其中的方法列表部分，而从上面的分析，可发现里面最基本也是重要的单位是method_t，这个结构体包含了描述一个方法所需要的各种信息。

```
struct method_t {
    SEL name;
    const char *types;
    IMP imp;
};
```

变量介绍可以参考之前文章：[iOS 代码注入—— hook 实践](https://xlgy.github.io/2020/06/16/iOS-hook-%E5%AE%9E%E8%B7%B5-%E5%8A%A8%E6%80%81%E6%B3%A8%E5%85%A5/)
   
### 三、Runtime 初始化函数

**1、一"码"当先**

```
/***********************************************************************
* _objc_init
* Bootstrap initialization. Registers our image notifier with dyld.
* Called by libSystem BEFORE library initialization time
**********************************************************************/

void _objc_init(void)
{
    static bool initialized = false;
    if (initialized) return;
    initialized = true;
    
    // fixme defer initialization until an objc-using image is found?
    environ_init();
    tls_init();
    static_init();
    lock_init();
    exception_init();

    _dyld_objc_notify_register(&map_images, load_images, unmap_image);
}
```

```_dyld_objc_notify_register(&map_images, load_images, unmap_image)```。这个函数里面的三个参数分别是另外三个函数：

- ```map_images``` -- Process the given images which are being mapped in by dyld.(处理那些正在被dyld映射的镜像文件)
- ```load_images``` -- Process +load in the given images which are being mapped in by dyld.（处理那些正在被dyld映射的镜像文件中的+load方法）
- ```unmap_image``` -- Process the given image which is about to be unmapped by dyld.(处理那些将要被dyld进行去映射操作的镜像文件)

我们查看一下map_images方法，点进去：

```
/***********************************************************************
* map_images
* Process the given images which are being mapped in by dyld.
* Calls ABI-agnostic code after taking ABI-specific locks.
*
* Locking: write-locks runtimeLock
**********************************************************************/
void
map_images(unsigned count, const char * const paths[],
           const struct mach_header * const mhdrs[])
{
    mutex_locker_t lock(runtimeLock);
    return map_images_nolock(count, paths, mhdrs);
}
```

**2、+load方法**
Apple文档是这样描述的：


![](https://github.com/xlgy/xlgy.github.io/blob/master/_posts/image/runtime5.png?raw=true)


load函数调用特点如下:
当类被引用进项目的时候就会执行load函数(在main函数开始执行之前）,与这个类是否被用到无关,每个类的load函数只会自动调用一次.由于load函数是系统自动加载的，因此不需要调用父类的load函数，否则父类的load函数会多次执行。

1. 当父类和子类都实现load函数时,父类的load方法执行顺序要优先于子类
2. 类中的load方法执行顺序要优先于类别(Category)
3. 当有多个类别(Category)都实现了load方法,这几个load方法都会执行,但执行顺序不确定(其执行顺序与类别在Compile Sources中出现的顺序一致)
4. 当然当有多个不同的类的时候,每个类load 执行顺序与其在Compile Sources出现的顺序一致

**代码验证：**
- 新建两个类 Class1、Class2
- 新建Class1子类Class1_Son
- 新建Class1子两个分类Class1+category1、Class1+category2

**Compile Sources文件顺序如图：**


![](https://github.com/xlgy/xlgy.github.io/blob/master/_posts/image/runtime6.png?raw=true)


每一个类和分类里面都实现+(void)load方法

```
+(void)load
{
    NSLog(@"%s",__FUNCTION__);
}
```

打印结果如下：

```
2022-01-07 15:41:21.334615+0800 loadTest[44548:3453142] +[Class1 load]
2022-01-07 15:41:21.335476+0800 loadTest[44548:3453142] +[Class1_Son load]
2022-01-07 15:41:21.335626+0800 loadTest[44548:3453142] +[Class2 load]
2022-01-07 15:41:21.335733+0800 loadTest[44548:3453142] +[Class1(category2) load]
2022-01-07 15:41:21.335897+0800 loadTest[44548:3453142] +[Class1(category1) load]
```

**3、+initialize方法**

Apple文档是这样描述的：


![](https://github.com/xlgy/xlgy.github.io/blob/master/_posts/image/runtime7.png?raw=true)


initialize函数调用特点如下:
initialize在类或者其子类的第一个方法被调用前调用。即使类文件被引用进项目,但是没有使用,initialize不会被调用。由于是系统自动调用，也不需要再调用 [super initialize] ，否则父类的initialize会被多次执行。假如这个类放到代码中，而这段代码并没有被执行，这个函数是不会被执行的

1. 父类的initialize方法会比子类先执行
2. 当子类未实现initialize方法时,会调用父类initialize方法,子类实现initialize方法时,会覆盖父类initialize方法.
3. 当有多个Category都实现了initialize方法,会覆盖类中的方法,只执行一个(会执行Compile Sources 列表中最后一个Category 的initialize方法)

**代码验证：**
- 新建两个类 Class1
- 新建Class1子类Class1_Son
- 新建Class1子两个分类Class1+category1、Class1+category2

**Compile Sources文件顺序如图：**


![](https://github.com/xlgy/xlgy.github.io/blob/master/_posts/image/runtime6.png?raw=true)


每一个类和分类里面都实现+(void)initialize方法

```
+(void)initialize
{
    NSLog(@"%s",__FUNCTION__);
}
```

执行代码：

```
[Class1 new];
```

打印结果:

```
2022-01-07 15:55:37.264982+0800 loadTest[45454:3474908] +[Class1(category1) initialize]
```

Compile Sources最后的分类会覆盖类的initialize方法，符合预期!!!
执行代码：

```
[Class1_Son new];
```


打印结果:

```
2022-01-07 15:57:45.806635+0800 loadTest[45541:3478338] +[Class1(category1) initialize]
2022-01-07 15:57:45.806706+0800 loadTest[45541:3478338] +[Class1_Son initialize]
```

先执行父类的initialize，再执行子类的initialize，符合预期!!!


**特殊情况：子类不实现initialize，执行代码：**

```
[Class1_Son new];
```

打印结果:

```
2022-01-07 15:59:47.870490+0800 loadTest[45632:3481492] +[Class1(category1) initialize]
2022-01-07 15:59:47.870554+0800 loadTest[45632:3481492] +[Class1(category1) initialize]
```

由于子类没有实现，所以子类调用initialize的走到的是父类的initialize方法，所以父类的initialize方法走了两边

### 四、分类底层原理

根据map_images函数，继续点进去看，可以看到如下代码：

```
// Discover categories. 
    for (EACH_HEADER) {
        category_t **catlist = 
            _getObjc2CategoryList(hi, &count);
        bool hasClassProperties = hi->info()->hasCategoryClassProperties();

        for (i = 0; i < count; i++) {
            category_t *cat = catlist[i];
            Class cls = remapClass(cat->cls);

            if (!cls) {
                // Category's target class is missing (probably weak-linked).
                // Disavow any knowledge of this category.
                catlist[i] = nil;
                if (PrintConnecting) {
                    _objc_inform("CLASS: IGNORING category \?\?\?(%s) %p with "
                                 "missing weak-linked target class", 
                                 cat->name, cat);
                }
                continue;
            }

            // Process this category. 
            // First, register the category with its target class. 
            // Then, rebuild the class's method lists (etc) if 
            // the class is realized. 
            bool classExists = NO;
            if (cat->instanceMethods ||  cat->protocols  
                ||  cat->instanceProperties) 
            {
                addUnattachedCategoryForClass(cat, cls, hi);
                if (cls->isRealized()) {
                    remethodizeClass(cls);
                    classExists = YES;
                }
                if (PrintConnecting) {
                    _objc_inform("CLASS: found category -%s(%s) %s", 
                                 cls->nameForLogging(), cat->name, 
                                 classExists ? "on existing class" : "");
                }
            }

            if (cat->classMethods  ||  cat->protocols  
                ||  (hasClassProperties && cat->_classProperties)) 
            {
                addUnattachedCategoryForClass(cat, cls->ISA(), hi);
                if (cls->ISA()->isRealized()) {
                    remethodizeClass(cls->ISA());
                }
                if (PrintConnecting) {
                    _objc_inform("CLASS: found category +%s(%s)", 
                                 cls->nameForLogging(), cat->name);
                }
            }
        }
    }
```

根据代码：

```
category_t *cat = catlist[i];
```

一开始的那个catlist是一个二维数组，里面的成员也是一个一个的数组，也就是代码里面的cat所指向的数组，它的类型是category_t *，说明cat数组里面装的就是category_t，一个cat里面装的就是某个class所对应的所有category。

***那么什么决定了这些category_t在cat数组中的顺序呢？***

答案是category文件的编译顺序决定的。先参与编译的，就放在数组的前面，后参与编译的，就放在数组后面。我们可以在xcode-->target-->Build Phases-->Compile Sources列表查看和调整category文件的编译顺序


加载分类的最后，执行方法:remethodizeClass(cls->ISA());

```
static void remethodizeClass(Class cls)
{
    category_list *cats;
    bool isMeta;

    runtimeLock.assertLocked();

    isMeta = cls->isMetaClass();

    // Re-methodizing: check for more categories
    if ((cats = unattachedCategoriesForClass(cls, false/*not realizing*/))) {
        if (PrintConnecting) {
            _objc_inform("CLASS: attaching categories to class '%s' %s", 
                         cls->nameForLogging(), isMeta ? "(meta)" : "");
        }
        
        attachCategories(cls, cats, true /*flush caches*/);        
        free(cats);
    }
}
```

然后在这里面找到一个方法attachCategories，看名字就知道，附着分类，也就是把分类的内容添加/合并到class里面，感兴趣的可以自己查看一下这个方法，这个理就不做解释了。


**category方法覆盖**

category的方法没有“完全替换掉”原来类已经有的方法，也就是说如果category和原来类都有methodA，那么category附加完成之后，类的方法列表里会有两个methodA。category的方法被放到了新方法列表的前面，而原来类的方法被放到了新方法列表的后面，这也就是我们平常所说的category的方法会“覆盖”掉原来类的同名方法，这是因为运行时在查找方法的时候是顺着方法列表的顺序查找的，它只要一找到对应名字的方法，就会罢休^_^，殊不知后面可能还有一样名字的方法。

### 五、方法缓存

**1、数据结构**

它的底层是通过散列表（哈希表）的数据结构来实现的，用于缓存曾经调用过的方法，可以提高方法的查找速度。
首先，回顾一下正常情况下方法调用的流程。假设我们调用一个实例方法[obj XXXX];

- obj -> isa -> obj的Class对象 -> method_array_t methods -> 对该表进行遍历查找，找到就调用，没找到继续往下走
- obj -> superclass -> obj的父类 -> isa -> method_array_t methods -> 对父类的方法列表进行遍历查找，找到就调用，没找到就重复本步骤
- 直到NSObject -> isa -> NSObject的Class对象 -> method_array_t，如果还是没有找到就会crash

如果XXXX方法在程序内会被频繁的调用，那么这种逐层便利查找的方式肯定是效率低下的，因此**苹果设计了cache_t cache**，当XXXX第一次被调用的时候，会按照常规流程查找，找到之后，就会被加入到cache_t cache中，当再次被调用的时候，系统就会直接现到cache_t cache来查找，找到就直接调用，这样便大大提升了查找的效率。

```
struct cache_t {
    struct bucket_t *_buckets;
    mask_t _mask;
    mask_t _occupied;
}
```

- struct bucket_t *_buckets; —— 用来缓存方法的散列/哈希表
- mask_t _mask; —— 这个值 = 散列表长度 - 1
- mask_t _occupied; —— 表示已经缓存的方法的数量

_buckets散列表里面的存储单元是bucket_t，

```
struct bucket_t {
private:
    cache_key_t _key;
    IMP _imp;
}
```

- cache_key_t _key; —— 这个key实际上就是方法的SEL，也就是方法名
- IMP _imp; —— 这个就是方法对应的函数的内存地址

**2、缓存逻辑**

1. 当一个对象接收到消息时[obj message];，首先根据obj的isa指针进入它的类对象class里面。
2. 在obj的class里面，首先到缓存cache_t里面查询方法message的函数实现，如果找到，就直接调用该函数。
3. 如果上一步没有找到对应函数，在对该class的方法列表进行二分/遍历查找，如果找到了对应函数，首先会将该方法缓存到obj的类对象**class**的cache_t里面，然后对函数进行调用。
4. 在每次进行缓存操作之前，首先需要检查缓存容量，如果缓存内的方法数量超过规定的临界值(设定容量的3/4)，需要先对缓存进行2倍扩容，**原先缓存过的方法全部丢弃**，然后将当前方法存入扩容后的新缓存内。
5. 如果在obj的**class**对象里面，发现缓存和方法列表都找不到mssage方法，则通过class的superclass指针进入它的父类对象**father_class**里面
6. 进入**father_class**后，首先在它的cache_t里面查找mssage，如果找到了该方法，那么会首先将方法缓存到消息接受者obj的类对象**class**的cache_t里面，然后调用方法对应的函数。
7. 如果上一步没有找到方法，将会对**father_class**的方法列表进行遍历二分/遍历查找，如果找到了mssage方法，那么同样，会首先将方法缓存到消息接受者obj的类对象**class**的cache_t里面，然后调用方法对应的函数。需要注意的是，**这里并不会将方法缓存到当前父类对象father_class的cache_t里面**。
8. 如果还没找到方法，则会通过**father_class**的superclass进入更上层的父类对象里面，按照(6)->(7)->(8)步骤流程重复。如果此时已经到了基类对象NSObject，仍没有找到mssage，则进入消息转发阶段


### 六、消息转发


**_objc_msgForward**是IMP类型的，用于消息转发的，当像一个对象发送消息，但他没有实现的时候，**_objc_msgForward**会尝试做消息转发。
**objc_msgSend**的动作比较清晰，在“消息传递”过程中，：首先在 Class 中的缓存查找 IMP （没缓存则初始化缓存），如果没找到，则向父类的 Class 查找。如果一直查找到根类仍旧没有实现，则用**_objc_msgForward**函数指针代替 IMP 。最后，执行这个 IMP 。

**_objc_msgForward**消息转发需要做的几件事：

1. 调用**+ (BOOL)resolveInstanceMethod:(SEL)sel(或 + (BOOL)resolveClassMethod:(SEL)sel)**方法，在此方法中添加响应selector以及IMP即可，允许用户在此时为该Class动态添加实现。如果有实现了，则调用并返回YES，那么重新开始**objc_msgSend**流程。对象会响应这个选择器，一般是因为它已经调用过class_addMethod。如果仍没实现，继续下面的步骤

2. 调用**- (id)forwardingTargetForSelector:(SEL)aSelector**方法，尝试找到一个能相应该消息的对象。如果获取到，则直接把消息转发给它，返回非nil对象。否则返回 nil ，继续下面的动作。

3. 调用 **- (NSMethodSignature \*)methodSignatureForSelector:(SEL)aSelector** 方法，尝试获得一个方法签名。如果能获取，则返回非nil：创建一个 NSlnvocation 并传给forwardInvocation:
调用**- (void)forwardInvocation:(NSInvocation \*)anInvocation**方法，将获取到的方法签名包装成 Invocation 传入，如何处理就在这里面了，并返回非nil。如果获取不到，则直接调用4抛出异常。

4. 调用**- (void)doesNotRecognizeSelector:(SEL)aSelector**，默认的实现是抛出异常。如果第3步没能获得一个方法签名，执行该步骤。


**第一步：Method resolution 方法解析处理阶段**

如果调用了对象方法首先会进行**+(BOOL)resolveInstanceMethod:(SEL)sel**判断
如果调用了类方法 首先会进行 **+(BOOL)resolveClassMethod:(SEL)sel**判断
两个方法都为类方法；


```
+ (BOOL)resolveClassMethod:(SEL)sel {
    ///这里动态添加方法
    return YES;
}
+ (BOOL)resolveInstanceMethod:(SEL)sel {
    ///这里动态添加方法
    return YES;
}
```


**_class_resolveInstanceMethod**源码解析


```
static void _class_resolveInstanceMethod(id inst, SEL sel, Class cls)
{
    SEL resolve_sel = @selector(resolveInstanceMethod:);

    if (! lookUpImpOrNilTryCache(cls, resolve_sel, cls->ISA())) {
        // Resolver not implemented.
        return;
    }

    BOOL (*msg)(Class, SEL, SEL) = (typeof(msg))objc_msgSend;
    bool resolved = msg(cls, resolve_sel, sel);

    // Cache the result (good or bad) so the resolver doesn't fire next time.
    // +resolveInstanceMethod adds to self a.k.a. cls
    IMP imp = lookUpImpOrNilTryCache(inst, sel, cls);

    if (resolved  &&  PrintResolving) {
        if (imp) {
            _objc_inform("RESOLVE: method %c[%s %s] "
                         "dynamically resolved to %p", 
                         cls->isMetaClass() ? '+' : '-', 
                         cls->nameForLogging(), sel_getName(sel), imp);
        }
        else {
            // Method resolver didn't add anything?
            _objc_inform("RESOLVE: +[%s resolveInstanceMethod:%s] returned YES"
                         ", but no new implementation of %c[%s %s] was found",
                         cls->nameForLogging(), sel_getName(sel), 
                         cls->isMetaClass() ? '+' : '-', 
                         cls->nameForLogging(), sel_getName(sel));
        }
    }
}
```

**从runtime的源码，```resolveInstanceMethod```的返回值对于消息转发流程没有任何意义，这个返回值只和debug的信息相关。**

这两个方法是最先走到的方法，可以在这两个方法中动态的添加方法，进行消息转发。这里有一个需要特别注意的地方，**类方法需要添加到元类里面**，原因这里就不赘述了。

代码实例：

```
#import "MissMethodClass.h"
#import <objc/runtime.h>

@implementation MissMethodClass

- (void)myTest:(NSString *)string1 withString:(NSString *)string2{
    NSLog(@"消息转发成功 string1:%@。string2:%@",string1,string2);
}

+ (BOOL)resolveInstanceMethod:(SEL)sel{
    if (sel == @selector(test:)) {
        Method myTestMethod = class_getInstanceMethod([self class], NSSelectorFromString(@"myTest:withString:"));
        class_addMethod([self class], sel, method_getImplementation(myTestMethod), method_getTypeEncoding(myTestMethod));
        return YES;
    }
    return [super forwardingTargetForSelector:sel];
}

@end
```

实例代码中在resolveInstanceMethod方法中处理方法名为**@selector(test:)**的消息，并动态添加方法实现，并且可以接收两个参数

调用代码：

```
    MissMethodClass *model = [MissMethodClass new];
    [model performSelector:@selector(test:) withObject:@"参数1" withObject:@"参数2"];
```

结果日志：

```
消息转发成功 string1:参数1。string2:参数2
```







**第二步:Fast forwarding 快速转发阶段**

```
- (id)forwardingTargetForSelector:(SEL)aSelector {
    return [xxx new];
}
```

这个里可以快速重定向成其他对象，已经让备用的对象去响应了该对象本身无法响应的一个SEL

**第三步：Normal forwarding 常规转发阶段**

```
//返回方法签名
-(NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector{
    if ([NSStringFromSelector(aSelector) isEqualToString:@"xxx"]) {
        return [[xxx new] methodSignatureForSelector:aSelector];
    }
    return [super methodSignatureForSelector:aSelector];
}

//处理返回的方法签名
-(void)forwardInvocation:(NSInvocation *)anInvocation{
    if ([NSStringFromSelector(anInvocation.selector) isEqualToString:@"xxx"]) {
        [anInvocation invokeWithTarget:[xxx new]];
    }else{
        [super forwardInvocation:anInvocation];
    }
}
```

***自动签名***

```
-(NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector{
    //如果返回为nil则进行自动签名
   if ([super methodSignatureForSelector:aSelector]==nil) {
        NSMethodSignature * sign = [NSMethodSignature signatureWithObjCTypes:"v@:"];
        return sign;
    }
    return [super methodSignatureForSelector:aSelector];
}


-(void)forwardInvocation:(NSInvocation *)anInvocation{
    //创建备用对象
    xxx * backUp = [xxx new];
    SEL sel = anInvocation.selector;
    //判断备用对象是否可以响应传递进来等待响应的SEL
    if ([backUp respondsToSelector:sel]) {
        [anInvocation invokeWithTarget:backUp];
    }else{
       // 如果备用对象不能响应 则抛出异常
        [self doesNotRecognizeSelector:sel];
    }
}

////触发崩溃
- (void)doesNotRecognizeSelector:(SEL)aSelector {

}
```

### 七、super的本质
**1、定义**
- super—— 是一个指向结构体指针struct objc_super *，它里面的内容是{消息接受者 recv， 消息接受者的父类类对象 [[recv superclass] class]}，objc_msgSendSuper会将消息接受者的父类类对象作为消息查找的起点。

**2、流程**
[obj message] -> 在obj的类对象cls查找方法 -> 在cls的父类对象[cls superclass]查找方法 -> 在更上层的父类对象查找方法 -> ... -> 在根类类对象 NSObject里查找方法

[super message] -> ~~在obj的类对象cls查找方法~~(跳过此步骤) -> (直接从这一步开始)在cls的父类对象[cls superclass]查找方法 -> 在更上层的父类对象查找方法 -> ... -> 在根类类对象 NSObject里查找方法

**3、实例**
```
 NSLog(@"[self class] = %@",[self class]);
```
- 接受者 当前class实例对象
- 最终调用的方法：基类NSObject的-(Class)class方法

```
 NSLog(@"[super class] = %@",[super class]);
```
- 接受者 当前class实例对象
- 最终调用的方法：基类NSObject的-(Class)class方法

```
 NSLog(@"[self superclass] = %@",[self superclass]);
```
- 接受者 当前class实例对象
- 最终调用的方法：基类NSObject的-(Class) superclass方法

```
 NSLog(@"[super superclass] = %@",[super superclass]);
```
- 接受者 当前class实例对象
- 最终调用的方法：基类NSObject的-(Class) superclass方法

**因此 [self class] 和 [super class]的值相等 ，[self superclass] 和 [super superclass] 的值相等**


### 八、给nil发送消息



我们知道 Objective-C 是以C语言为基础的，在C语言中对空指针进行操作会导致程序崩溃，为什么在 Objective-C 中给 nil 发送消息不会出现崩溃呢？

Objective-C中的函数调用都是通过objc_msgSend进行消息发送来实现的，而objc_msgSend会通过判断参数self来决定是否发送消息，如果传递给objc_msgSend的参数self为 nil，那么selector会被置空，该函数不会执行而是直接返回。

**特别说明**

iOS官方文档

```
Sending Messages to nil
In Objective-C, it is valid to send a message to nil—it simply has no effect at runtime. There are several patterns in Cocoa that take advantage of this fact. The value returned from a message to nil may also be valid:

- If the method returns an object, then a message sent to nil returns 0 (nil).

- If the method returns any pointer type, any integer scalar of size less than or equal to sizeof(void*), a float, a double, a long double, or a long long, then a message sent to nil returns 0.

- If the method returns a struct, as defined by the OS X ABI Function Call Guide to be returned in registers, then a message sent to nil returns 0.0 for every field in the struct. Other struct data types will not be filled with zeros.

- If the method returns anything other than the aforementioned value types, the return value of a message sent to nil is undefined.

```

在Objective-C中，给nil发送消息**是有效的，只是在运行时不会起作用**。Cocoa几个模式就利用了这一点。给nil发送消息的返回值也是有效的。

- 如果方法的返回值是对象，那给nil发送消息会返回0(nil)。
- 如果方法的返回值是指针类型，其指针类型大小是小于等于 sizeof(void*)， float，double，long double， long long，给nil发送消息将返回0。
- 如果方法的返回值为struct(结构体)，定义在OS X ABI Function Call Guide 里以寄存器形式返回的，那么给nil发送消息会返回的结构体中的各个字段都为0，其它结构体数据类型的就不会用0填充。
- 如果返回值不是上述描述的几种情况，给nil发送消息返回值是undefined。



### 九、（nil、Nil、NULL、NSNull）对比
- nil : 指向 Objective-C 中对象的空指针
- Nil : 指向 Objective-C 中类的空指针
- NULL ：指向其他类型的空指针，如一个c类型的内存指针
- NSNull ：在集合对象中，表示空值的对象



```
至此，runtime相关的知识点全部总结完毕，该文章将会持续更新迭代！！
看到就是缘分😁，如发现任何有误之处，肯请留言纠正，谢谢。

```

