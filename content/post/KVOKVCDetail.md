---
title: "KVOKVCDetail"
date: 2020-08-14T20:01:44+08:00
tags: ["KVC&KVO", "面试"]
categories: ["面试总结"]
author: "Zhushuai"
draft: false
---

# KVO实现原理

### 1.KVO的简介

> KVO的全称是Key-Value Observing , 可以用于监听某个对象属性值的变化

要监听Person中的age属性， 我们就创建一个observer用来监听age的变化，当我们age一旦发生改变，就会通知observer。



### 2. KVO的简单实现

```objective-c
///> DLPerson.h 文件

#import <Foundation.h>

NS_ASSUME_NONNULL_BEGIN

@interface DLPerson : NSObject

@property(nonatomic, strong) int age;

@end

NS_ASSUME_NONNULL_END
```

```
///> ViewController.m

#import "ViewController.h"
#improt "DLPeronson.h"
@interface ViewController ()
@property (nonatomic, strong) DLPerson *person1;
@property (nonatomic , strong) DLPerson *person2;
@end

@implementation ViewController

-(void)viewDidLoad {
	[super viewDidLoad];
	self.person1 = [[DLPerson alloc]init];
	self.person2 = [[DLPerson alloc]init];
	
	self.person1.age = 1;
	self.person2.age = 2;
	
	// person1 添加KVO监听
	NSKeyValueObservingOptions options = NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld;
	[self.person1 addObserver:self forKeyPath:@"age" options: options context:nil];
}

-(void)touchesBegin:(NSSer<UITouch *>*)touches withEvent:(UIEvent *)event{
	self.person1.age = 10;
	self.person2.age = 20;
}

-(void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey, id> *)change context:(void *)context{
	NSLog(@"监听到了%@的%@属性发生了变化%@",object,keyPath,change);
}

-(void)dealloc{
	//使用结束后移除
	[self.person1 removeObserver:self forKeyPath:@"age"];
}
```



### 3. KVO本质分析

我们打印一下person1 和 person2 的指针看一下他们的实例对象isa指针指向的类对象是什么？

我们会发现person1的isa指针打印出来的是: **_NSKVONotifying_DLPerson_**

而person2的isa指针打印出来的是：**_DLPerson_**

当一个对象添加了KVO的监听，当前对象的isa的指针指向的就不是你原来的类，指向的是另外一个类对象。

![image-20200814204255717](https://tva1.sinaimg.cn/large/007S8ZIlgy1ghqmjv0hy2j310c0iin8s.jpg)

* NSKVONotifying_DLPerson 类是runtime动态创建的一个类，在程序运行的过程中产生的一个新的类
* NSKVONotifying_DLPerson 类是DLPerson的一个子类
* NSKVONotifying_DLPerson类存在自己的setAge: 、class 、 dealloc 、isKVOA .. 方法

当我们DLPerson的实例对象调用setAge方法时，实例对象的isa指针找到类对象，然后在类对象中寻找相应的对象方法，如果有则调用，如果没有则去superclass指向的父类对象中寻找相应的对象方法，如果有则调用，如果未找到相应的方法则会报: unrecognized selector sent to instance 错误



### 4. KVO的调用顺序

由于NSKVONotifying_DLPerson类不参与编译，所以可以在DLPerson.m中重写它父类的方法代码如下：

```
///> DLPerson.m 文件
#import "DLPerson.h"

@implementation DLPerson

-(void)setAge:(int)age{
	_age = age;
}

-(void)willChangeValueForKey:(NSString *)key{
	[super willChangeValueForKey: key];
	NSLog(@"willChangeValueForKey");
}

-(void)didChangeValueForKey:(NSString *)key{
	NSLog(@"did change value for key begin");
	[super didChangeValueForKey];
	NSLog(@"didChange value for key - end");
}
```



输出结果：

```
willChangeValueForKey
didChangeValueFor key begin
监听变化
didChangeValueForKey - end
```

总结： didChangeValueForKey: 内部会调用observer的observeValueForKeyPath:ofObject:change:contenxt:方法。



# 二、KVC的实现原理

### 1. 什么是KVC

> KVC == key value coding , 键值编码, 可以通过key来访问某个属性

常见的API有：

```
-(void)setValue:(id)value forKey:(NSString *)key;
-(void)setValue:(id)value forKeyPath:(NSString *)keyPath;

-(id)valueForKey:(NSString *)key;
-(id)valueForKeyPath:(NSString *)keyPath;   
```

简单实现：DLPerson 和 DLCat

```
///> DLPerson.h 文件
#import <Foundation/Foundation.h>
NS_ASSUME_NONNULL_BEGIN

/**
*DLCat
*/
@interface DLCat: NSObject
@property (nonatomic,assign) int weight;
@end

///DLPerson
@interface DLPerson : NSObject
@property (nonatmic, assign) int age;
@property (nonatomic, strong) DLCat *cat;
@end

NS_ASSUME_NONNULL_END
	
```

#### 1.1 -(void)setValue:(id)value forKey:(NSString *)key;

```
/// viewController.m

-(void)viewDidLoad{
	[super viewDidLoad];
	DLPerson *person = [[DLPerson alloc ] init];
	[person setValue:@20 forKey:@"age"];
	NSLog("%d",person.age);
}
```

#### 1.2 -(void)setValue:(id)value forKeyPath:(NSString *)keyPath

```
/// viewController. m
-(void)viewDidLoad{
	[super viewDidLoad];
	DLPerson *person = [[DLPerson alloc] init];
	person.cat = [[DLCat alloc] init];
	[person setValue:@20 forKeyPath:@"cat.weight"];
	NSLog(@"%@",person.age);
	NSLog(@"%@",person.cat.weight);
}
```

#### 1.3 - (id)valueForKey:(NSString *)key;

```
///> ViewController.m 文件

- (void)viewDidLoad {
    [super viewDidLoad];
    DLPerson *person = [[DLPerson alloc]init];
    person.age = 10;
    NSLog(@"%@",[person valueForKey:@"age"]);
}
复制代码
```

#### 1.4 - (id)valueForKeyPath:(NSString *)keyPath;

```
///> ViewController.m 文件

- (void)viewDidLoad {
    [super viewDidLoad];
    DLPerson *person = [[DLPerson alloc]init];
    person.age = 10;
    NSLog(@"%@",[person valueForKey:@"cat.weight"]);
}
```

### 2. setValue: forKey 的原理

![KVC image](https://tva1.sinaimg.cn/large/007S8ZIlgy1ghqnuir3u3j329t0u0qqx.jpg)

* 当我们设置setValue: forKey: 时
* 首先会查找setKey: , _setKey: 
* 如果有则调用
* 如果没有，先查看accessInstanceVariableDirectly方法

```
+ (BOOL)accessInstanceVariablesDirectly{
      return YES;   ///> 可以直接访问成员变量
  //    return NO;  ///>  不可以直接访问成员变量,  
  ///> 直接访问会报NSUnkonwKeyException错误  
  }

```

* 如果可以访问会按照_key, _isKey , key , iskey 的顺序查找成员变量
* 找到直接复制
* 未找到报错NSUnknownKeyException错误

_key、_isKey、key、iskey复制顺序

```
///> DLPerson.h 文件

@interface DLPerson : NSObject{
    @public
    int _age;
    int _isAge;
    int age;
    int isAge;
}

@end
```

### 3 . valueforKey 原理

![valueForKey](https://tva1.sinaimg.cn/large/007S8ZIlgy1ghqo18hd21j31yw0pe7ni.jpg)

kvc取值按照 getKey、key、iskey、_key 顺序查找方法

存在直接调用

没找到同样，先查看accessInstanceVariablesDirectly方法

```
+ (BOOL)accessInstanceVariablesDirectly{
      return YES;   ///> 可以直接访问成员变量
  //    return NO;  ///>  不可以直接访问成员变量,  
  ///> 直接访问会报NSUnkonwKeyException错误  
  }
复制代码
```

如果可以访问会按照 _key、_isKey、key、iskey的顺序查找成员变量

找到直接复制

未找到报错NSUnkonwKeyException错误