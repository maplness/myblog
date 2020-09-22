---
title: "OC中的锁"
date: 2020-08-15T13:31:09+08:00
categories: ["OC"]
tags: ["锁","多线程","OC"]
draft: false
---

# OC中用不同的方式实现锁

原文链接 http://www.tanhao.me/pieces/616.html/



今天一起来探讨一下objective-c中几种不同方式实现的锁，在这之前我们先构建一个测试用的类，假想它是一个我们的共享资源，method1与method2是互斥的。

```objective-c
@implementation TestObj

-(void)method1
{
	NSLog(@"%@",NSStringFromSelector(_cmd));
}

-(void)method2
{
	NSLog(@"%@",NSStirngFromSelector(_cmd));
}

@end
```



### 1. 使用NSLock实现的锁

```objective-c
//主线程中
TestObj *obj = [[TestObj alloc] init];
NSLock *lock = [[NSLock alloc] init];

//线程1
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT,0) ,^{
	[lock lock];
	[obj method1];
	sleep(10);
	[lock unlock];
});

//线程2
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT,0), ^{
	sleep(1);
	[lock lock];
	[obj method2];
  [lock unlock];
})
```

NSLock是Cocoa提供给我们的最基本的锁对象，这也是我们经常使用的。除了lock和unlock方法外，NSLock还提供了tryLock 和 lockBeforeDate方法。前一个方法会尝试加锁，如果锁不可用或已被锁住，不会阻塞线程，并返回NO。locakBeforeDate方法在锁制定Date前尝试加锁，如果在制定时间前都不能加锁，则返回NO。



### 2. 使用synchronized关键字构建的锁

当然在Objective-C中还可以用@synchronized指令快速的实现锁：

```objective-c
//主线程中
TestObj *obj = [[TestObj alloc] init];
//线程1
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    @synchronized(obj){
        [obj method1];
        sleep(10);
    }
});
//线程2
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    sleep(1);
    @synchronized(obj){
        [obj method2];
    }
});
```

@synchronized 指令使用的obj是该锁的唯一标识，只有当标识相同时，才为满足互斥，如果线程2中的@synchronized（obj）改为@synchronized(other)，线程2就不会被阻塞。



### 3. 使用C语言的pthread_mutex_t实现的锁

```objective-c
//主线程中
TestObj *obj = [[TestObj alloc] init];

__block pthread_mutex_t mutex;
pthread_mutex_init(&mutex,NULL);

//线程1
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT,0), ^{
		pthread_mutex_lock(&mutex);
    [obj method1];
    sleep(5);
    pthread_mutex_unlock(&mutex);
})

//线程2
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT,0), ^{
		sleep(1);
    pthread_mutex_lock(&mutex);
    [obj method2];
    pthread_mutex_unlock(&mutex);
})
```

pthread_mutex_t定义在pthread.h，所以记得#include



### 4. 使用GCD来实现的锁

GCD中提供了一种信号机制，使用它我们也可以来构建“锁”

```objc
//主线程中
TestObj *obj = [[TestObj alloc] init];
dispatch_semaphore_t semaphore = dispatch_semaphore_create(1);

//线程1
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
	dispatch_semaphore_wait(semaphore,DISPATCH_TIME_FOREVER);
	[obj method1];
	sleep(10);
	dispatch_semaphore_signal(semaphore);
})

//线程2
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
	sleep(1);
  dispatch_semaphore_wait(semaphore,DISPATCH_TIME_FOREVER);
  [obj method2];
  dispatch_semaphore_signal(semaphore);
})
```



### 5. NSRecursiveLock 递归锁

平时在代码中使用锁的时候，最容易犯的一个错误就是造成死锁，而最容易造成死锁的一种情形就是在递归或循环中。

```objc
//主线程中
NSLock *theLock = [[NSLock alloc] init];
TestObj *obj = [[TestObj alloc ] init];

//线程1
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULTY, 0) , ^{
	static void(^TestMethod)(int);
	TestMethod = ^(int value){
		[theLock lock];
		if(value > 0){
			[obj method1];
			sleep(5);
			TestMethod(value-1);
		}
		[theLock unlock];
	};
	
	TestMethod(5);
})

//线程2
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULTY,0),^{
	sleep(1);
	[theLock lock];
	[obj method2];
	[theLock unlock];
})
```

以上代码在线程1的递归block中，锁被多次的lock，所以自己也阻塞了，那么如何在递归或循环中正确的使用锁呢，此处的theLock如果换用NSRecursiceLock对象，问题就得到解决了。NSRecursiveLock可以在一个线程中多次lock，而不会造成死锁。递归锁会跟踪它被多少次lock，每次成功的lock都必须平衡调用unlock操作，只有所有的锁住和解锁操作平衡的时候，递归锁才会释放给其他线程获得。



### 6. NSConditionLock 条件锁

当我们在使用多线程的时候，有时一把只会lock和unlock的锁未必能完全满足我们的使用。因为普通的锁只关心锁与不锁，而不在乎用什么钥匙才能开锁，而我们在处理共享资源的时候，多数情况时只有满足一定条件的情况下才能打开这把锁。

```
//主线程中
NSConditionLock *theLock = [[NSConditionLock alloc] init];

//线程1
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULTY,0),^{
	for(int i =0; i<=2; i++){
		[theLock lock];
		NSLog(@"thread1:%d",i);
		sleep(2);
		[theLock unlockWithCondition: i]; 
	}
})

//线程2
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULTY,0),^{
	[theLock lockWhenCondition:2]
	NSLog(@"thread2");
	[theLock unlock];
})
```

在线程1的加锁使用了lock，所以是不需要条件的，所以顺利的就锁住了，但在unlock的时候使用了一个整型条件，它可以开启其它线程中正在等待这把钥匙的临界地，而线程2则需要一把被标识为2的钥匙，所以当线程1循环到最后一次的时候，才最终打开了线程2中的阻塞。但即便如此，NSConditionLock也跟其它的锁一样，是需要lock与unlock对应的，只是lock,lockWhenCondition:与unlock，unlockWithCondition:是可以随意组合的，当然这是与你的需求相关的。



### 7. NSDistributedLock 分布式锁

以上的锁都是在解决多线程之间的冲突，但如果遇上多个进程或多个程序之间需要构建互斥的情景怎么办呢？这个时候我们就需要用到NSDistributedLock, 从名字上就可以知道它是一个分布式的锁。NSDistriburedLock 的实现是通过文件系统。所以使用它才1可以有效的实现不同进程间的互斥，但NSDistributedLock并非继承于NSLock，它没有lock方法，只有tryLock , unlock, breakLock.所以如果需要lock的时候，必须自己实现一个trylock的轮询。



程序A：

```
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    lock = [[NSDistributedLock alloc] initWithPath:@"/Users/mac/Desktop/earning__"];
    [lock breakLock];
    [lock tryLock];
    sleep(10);
    [lock unlock];
    NSLog(@"appA: OK");
});
```



程序B：

```
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        lock = [[NSDistributedLock alloc] initWithPath:@"/Users/mac/Desktop/earning__"];
        while (![lock tryLock]) {
            NSLog(@"appB: waiting");
            sleep(1);
        }
        [lock unlock];
        NSLog(@"appB: OK");
    });
```



先运行程序A,然后立即运行程序B,根据打印你可以清楚的发现，当程序A刚运行的时候，程序B一直处于等待中，当大概10秒过后，程序B便打印出了appB:OK的输出，以上便实现了两上不同程序之间的互斥。/Users/mac/Desktop/earning__是一个文件或文件夹的地址，如果该文件或文件夹不存在，那么在tryLock返回YES时，会自动创建该文件/文件夹。在结束的时候该文件/文件夹会被清除，所以在选择的该路径的时候，应该选择一个不存在的路径，以防止误删了文件。