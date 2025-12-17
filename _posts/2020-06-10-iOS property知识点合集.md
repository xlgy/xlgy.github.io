---
layout:     post
title:      iOS property知识点合集
subtitle:   iOS property知识点合集
date:       2020-06-10
author:     LXY
header-img: img/home1.jpeg
catalog: true
tags:
    - iOS
---


# 1、property 修饰符

**readonly**
此标记说明属性是只读的，默认的标记是读写，如果你指定了只读，在@implementation中只需要一个读取器。或者如果你使用**@synthesize**关键字，也是有**get方法**被解析。而且如果你试图使用点操作符为属性赋值，你将得到一个编译错误。

**readwrite**
此标记说明属性会被当成读写的，这也是**默认修饰符**。设置器和读取器都需要在@implementation中实现。如果使用**@synthesize**关键字，**get**和**set**方法都会被解析。

**assign**
此标记说明设置器直接进行赋值，这也是**默认值**。在使用垃圾收集的应用程序中，如果你要一个属性使用assign，且这个类符合NSCopying协议，你就要明确指出这个标记，而不是简单地使用默认值，否则的话，你将得到一个编译警告。这再次向编译器说明你确实需要赋值，即使它是可拷贝的。

**strong、weak**
默认strong
strong表示属性对所赋的值持有强引用表示一种“拥有关系”(owning relationship)，会先保留新值即增加新值的引用计数，然后再释放旧值即减少旧值的引用计数。只能修饰对象。如果对一些对象需要保持强引用则使用strong。

weak表示对所赋的值对象持有弱引用表示一种“非拥有关系”(nonowning relationship)，对新值不会增加引用计数，也不会减少旧值的引用计数。所赋的值在引用计数为0被销毁后，weak修饰的属性会被自动置为nil能够有效防止野指针错误。
weak常用在修饰delegate等防止循环引用的场景。



**copy**
它指出，在赋值时使用传入值的一份拷贝。拷贝工作由copy方法执行，此属性只对那些实行了NSCopying协议的对象类型有效。
**注意：如果是可变数组的属性使用此修饰符会变成不可变数组，容易造成crash**

**atomic/nonatomic**
**默认值：atomic**
指定合成存取方法是否为原子操作，可以理解为是否线程安全，但在iOS上即时使用atomic也不一定是线程安全的，要保证线程安全需要使用锁机制，超过本文的讲解范围，可以自行查阅。
可以发现几乎所有代码的属性设置都会使用nonatomic，这样能够提高访问性能，在iOS中使用锁机制的开销较大，会损耗性能。

**结论：基本数据默认的关键字是 atomic、readwrite、assign，普通的 OC 对象: atomic、readwrite、strong。**

# 2、@property、@synthesize和@dynamic

**@property的本质：**
```
@property = ivar(实例变量) + getter/setter（存取方法）;
```
完成属性定义后，编译器会自动编写访问这些属性所需的方法，此过程叫做“自动合成”(autosynthesis)。需要强调的是，这个过程由编译 器在编译期执行，所以编辑器里看不到这些“合成方法”(synthesized method)的源代码。除了生成方法代码 getter、setter 之外，编译器还要自动向类中添加适当类型的实例变量，并且在属性名前面加下划线，以此作为实例变量的名字。

我们每次在增加一个属性,系统都会在 ivar_list 中添加一个成员变量的描述,在 method_list 中增加 setter 与 getter 方法的描述,在属性列表中增加一个属性的描述,然后计算该属性在对象中的偏移量,然后给出 setter 与 getter 方法对应的实现,在 setter 方法中从偏移量的位置开始赋值,在 getter 方法中从偏移量开始取值,为了能够读取正确字节数,系统对象偏移量的指针类型进行了类型强转.

@property有两个对应的词，一个是 @synthesize，一个是 @dynamic。如果@synthesize 和 @dynamic都没写，那么默认的就是@syntheszie var = _var;


**@synthesize**

@synthesize表示如果属性没有手动实现setter和getter方法，编译器会自动加上这两个方法。

**@dynamic**

@dynamic告诉编译器：属性的 setter 与 getter 方法由用户自己实现，不自动生成。假如一个属性被声明为@dynamic var，而且你没有提供@setter 方法和@getter 方法，编译的时候没问题，但是当程序运行到 instance.var = someVar，由于缺 setter 方法会导致程序崩溃；或者当运行到 someVar = var 时，由于缺 getter 方法同样会导致崩溃。编译时没问题，运行时才执行相应的方法，这就是所谓的动态绑定。

# 3、 给分类(Category)添加属性


我们知道在一个类中用@property声明属性，编译器会自动帮我们生成成员变量和setter/getter，但分类的指针结构体中，根本没有属性列表。所以在分类中用@property声明属性，既无法生成成员变量也无法生成setter/getter。
因此：我们可以用@property声明属性，编译会通过，但run之后就会崩溃。

**如何添加**

既然报错的根本原因是使用了系统没有生成的setter/getter方法，可不可以在手动添加setter/getter来避免崩溃，完成调用呢？
其实是可以的。由于OC是动态语言，方法真正的实现是通过runtime完成的，虽然系统不给我们生成setter/getter，但我们可以通过runtime手动添加setter/getter方法。

既然报错的根本原因是使用了系统没有生成的setter/getter方法，可不可以在手动添加setter/getter来避免崩溃，完成调用呢？
其实是可以的。由于OC是动态语言，方法真正的实现是通过runtime完成的，虽然系统不给我们生成setter/getter，但我们可以通过runtime手动添加setter/getter方法。

虽然无法生成成员变量，但是我么可以利用**对象关联来实现**


```
 /*
     * id object 给哪个对象的属性赋值
       const void *key 属性对应的key
       id value  设置属性值为value
       objc_AssociationPolicy policy  使用的策略，是一个枚举值，和copy，retain，assign是一样的，手机开发一般都选择NONATOMIC
     */
     
     objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy);
```


例如：给类Person的分类Person (PersonExtention)添加属性name

```
.h文件
#import "Person.h"
@interface Person (PersonExtention)
@property (copy, nonatomic) NSString *name;
@end

.m文件
#import "Person+PersonExtention.h"
#import <objc/runtime.h>
@implementation Person (PersonExtention)
-(void)setName:(NSString *)name{
    objc_setAssociatedObject(self, @selector(name), name, OBJC_ASSOCIATION_COPY_NONATOMIC);
}
-(NSString *)name{
    return objc_getAssociatedObject(self, @selector(name));
}
@end
```

# 4、 给协议(Protocol)添加属性

给协议添加属性也是编译器不会自动生成实现setter/getter方法，但是遵循协议的类中是可以添加成员变量的，可以不用关联对象。

例如定义一个协议MyProtocal,里面添加属性name

```
@protocol MyProtocal <NSObject>

@property (nonatomic, copy) NSString *name;

@end
```

类Person遵循协议MyProtocal

```
#import <Foundation/Foundation.h>
#import "MyProtocal.h"

@interface Person : NSObject<MyProtocal>

@end

```

```
#import "Person.h"

@implementation Person
@synthesize name = _name;

@end

```

只需要
```
@synthesize name = _name;
```
即可

@synthesize表示如果属性没有手动实现setter和getter方法，编译器会自动加上这两个方法，成员变量是_name。