

# 多线程常见崩溃

## 0x0 Block 回调的崩溃
在MRC环境下，使用Block 来设置下载成功的图片。当self释放后，weakSelf变成野指针,接着就悲剧了

```
 __block ViewController *weakSelf = self;
 [self.imageView imageWithUrl:@"" completedBlock:^(UIImage *image, NSError *error) {
        
        NSLog(@"%@",weakSelf.imageView.description);
 }];
 
```

## 0x1 多线程下Setter 的崩溃
Getter & Setter 写多了，在单线程的情况下，是没有问题的。但是在多线程的情况下，可能会崩溃。因为***[_imageView release];*** 这段代码可能会被执行两次，oops!

UIKit 不是线程，所以在不是主线程的地方调用UIKit 的东西，有可能在开发阶段完全没问题，直接免测。但是一到线上，崩溃系统可能都是你的崩溃日志。Holy shit!

解决办法：[通过hook 住setNeedsLayout，setNeedsDisplay，setNeedsDisplayInRect来检查当前调用的线程是否是主线程](https://gist.github.com/steipete/5664345)。


```
- (void)setImageView:(UIImageView *)imageView
{
    if (![_imageView isEqual:imageView])
    {
        [_imageView release];
        _imageView = [imageView retain];
    }
}

```


## 0x2 更多Setter 类型的崩溃
property 的属性，写的最多的就是nonatomic，一般情况下也是没有问题的！

```
@interface ViewController ()
@property (strong,nonatomic) NSMutableArray *array;
@end

```

跑一下下面这段代码,你会看到:
***malloc: error for object 0x7913d6d0: pointer being freed was not allocated***

```
for (int i = 0; i < 100; i++) {
        dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
            self.array = [[NSMutableArray alloc] init];
        });
    }    
```

原因就是：对象被重复relaese 了。查看一下[runtime 源码](https://github.com/opensource-apple/objc4/blob/master/runtime/objc-accessors.mm)

![](./1.png)

解决办法：属性声明为atomic.

一个更为常见的例子：

```
if(handler == nil)
{
	hander = [[Handler alloc] init];
}

return handler;
```

如果A,B两个线程同时访问到if语句, 此时handler == nil条件满足, 两个线程都走到下一句初始化实例.
此时A线程先完成初始化并赋值(这个实例我们叫它a), 然后继续往后走到其他逻辑.而这时候, B线程开始做初始化并赋值(这个实例我们叫它b), handler将指向B线程初始化出来的对象. 而A初始化出来的实例a因为引用计数减少1(减少到0)而被释放. 但在A线程中, 代码还会尝试访问a所在的地址, 这个地址里的内容因为被释放而变得无法预测, 从而导致野指针.

问题还有一个很关键的点, 在一个对象的某个方法的调用过程中, 这个对象的引用计数并不会增加, 到导致它如果被释放, 后续的执行过程中对这个对象的访问就可能会导致野指针[1].

```
Exception Type:  SIGSEGV
Exception Codes: SEGV_ACCERR at 0x12345678
Triggered by Thread:  1
```

简单加个锁就可以解决问题了：

```
 @synchronized(self){
 	if(handler == nil)
	{
		hander = [[Handler alloc] init];
	}
 }
return handler;
```


## 0x3 多线程下对变量的存取
```
if (self.xxx) {
    [self.dict setObject:@"ah" forKey:self.xxx];
}
```

大家第一眼看到这样的代码，是不是会认为是正确的？因为在设置key的时候已经提前进行了self.xxx为非nil的判断，只有非nil得情况下才会执行后续的指令。但是，如上代码只有在单线程的前提下才是正确的。

假设我们将上述代码目前执行的线程为Thread A，当我们执行完if (self.xxx)的语句之后，此时CPU将执行权切换给了Thread B，而这个时候Thread B中调用了一句self.xxx = nil。 ***使用局部变量可以解决这个问题***

```
__strong id val = self.xxx;
if (val) {
    [self.dict setObject:@"ah" forKey:val];
}
```

这样，无论多少线程尝试对self.xxx进行修改，本质上的val都会保持现有的状态，符合非nil的判断。

## 0x4 dispatch_group 的崩溃
```dispatch_group_enter``` 和 leave 必须是匹配的，不然就会crash . 在多资源下载的时候，往往需要使用多线程并发下载，全部下载完之后通知用户。开始下载，```dispatch_group_enter``` ,下载完成```dispatch_group_leave``` 。 非常简单的流程，但是当代码复杂到一定程度或者是使用了一些第三方库的时候，就很大可能出问题。


```
dispatch_group_t serviceGroup = dispatch_group_create();
dispatch_group_notify(serviceGroup, dispatch_get_main_queue(), ^{
    NSLog(@"Finish downloading :%@", downloadUrls);
});

// t 是一个包含一堆字符串的数组 
[downloadUrls enumerateObjectsUsingBlock:^(id  _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
    dispatch_group_enter(serviceGroup);
    SDWebImageCompletionWithFinishedBlock completion =
    ^(UIImage *image, NSError *error, SDImageCacheType cacheType, BOOL finished, NSURL *imageURL) {
        dispatch_group_leave(serviceGroup);
        NSLog(@"idx:%zd",idx);
    };
    [[SDWebImageManager sharedManager] downloadImageWithURL:[NSURL URLWithString: downloadUrls[idx]]
                                                    options:SDWebImageLowPriority
                                                   progress:nil
                                                  completed:completion];
}];
```

使用多线程进行并发下载，直到所有图片都下载完成（可以失败）进行回调，其中图片下载使用的是SDWebImage.发生崩溃的场景是：***有10 张图片，分开两次下载（A & B）。其中在B组里面有一张图片和A组下载的图片重复了。假设A组下载对应GroupA ,B组GroupB***


下面截取SDWebImage源码：

```
dispatch_barrier_sync(self.barrierQueue, ^{
    SDWebImageDownloaderOperation *operation = self.URLOperations[url];
    if (!operation) {
        operation = createCallback();

        // ****注意这行****
        self.URLOperations[url] = operation;

        __weak SDWebImageDownloaderOperation *woperation = operation;
        operation.completionBlock = ^{
          SDWebImageDownloaderOperation *soperation = woperation;
          if (!soperation) return;
          if (self.URLOperations[url] == soperation) {
              [self.URLOperations removeObjectForKey:url];
          };
        };
    }

// ****注意这行****
id downloadOperationCancelToken = [operation addHandlersForProgress:progressBlock completed:completedBlock];
}
```
SDWebImage的下载器会根据URL做下载任务对应NSOperation映射，相同的URL会映射到同一个未执行的NSOperation。当A组图片下载完成后，相同的url 回调是 GroupB 而不是Group A。此时Group B的计数为1 。当B 组图片全部下载完后，结束计数为 5+1 。因为enter 的次数为5 ,leave 的次数为6 ,因此会崩溃！


## 0x5 最后一个持有者释放后的崩溃

对象A被 manager 持有，在A中调用[Manager removeObjectA]。A对象的retainCount -1,当retainCount 等于零时，对象A已经开始释放了。在调用removeObjectA 后，紧接着调用[self doSomething],就会崩溃。

```
- (void)finishEditing
{
	[Manager removeObject:self];
	[self doSomething];
}

```
这种情况一般会发生在数组或者字典包含对象，而且是对象的最后持有者。当在对象处理不好，就会有上面的崩溃。还有一种情况就是，当数组或者字典里面的对象已经被释放了，当遍历数组或者取字典里面的值发生崩溃。这种情况，会让人很崩溃，因为有时候堆栈是这样的：


```
Thread 0 Crashed:
0   libobjc.A.dylib                 0x00000001816ec160 _objc_release :16 (in libobjc.A.dylib)
1   libobjc.A.dylib                 0x00000001816edae8 __ZN12_GLOBAL__N_119AutoreleasePoolPage3popEPv :508 (in libobjc.A.dylib)
2   CoreFoundation                  0x0000000181f4c9fc __CFAutoreleasePoolPop :28 (in CoreFoundation)
3   CoreFoundation                  0x0000000182022bc0 ___CFRunLoopRun :1636 (in CoreFoundation)
4   CoreFoundation                  0x0000000181f4cc50 _CFRunLoopRunSpecific :384 (in CoreFoundation)
5   GraphicsServices                0x0000000183834088 _GSEventRunModal :180 (in GraphicsServices)
6   UIKit                           0x0000000187236088 _UIApplicationMain :204 (in UIKit)
7   Tmall4iPhone                    0x00000001000b7ae4 main main.m:50 (in Tmall4iPhone)
8   libdyld.dylib                   0x0000000181aea8b8 _start :4 (in libdyld.dylib)

```
产生这种堆栈可能的场景是：
**释放Dictionary的时候，某个值(value)因为被其他代码提前释放变成野指针, 此时再次被释放触发Crash. 如果可以在每个Dictionary释放的时候, 把所有的key/value打出来, 如果某个key/value刚好被打出来之后, crash就发生了, 那么挂就挂在刚被打出来的key/value上.**


## 0x6 对象的释放线程要和它处理事情的线程一致

对象A在主线程监听Notification事件，如果这个对象被其它线程释放了。此刻，如果对象A 正在执行notification 相关的操作，再访问对象相关资源就野指针了，发生crash.


## 0x7 performSelector:withObject:afterDelay:

调用此方法，如果不是在主线程，那么必须要确保当前线程的ruuloop是存在的，performSelector_xxx_afterDelay 依赖runlopp才能执行。另外使用 performSelector:withObject:afterDelay:和 cancelPreviousPerformRequestsWithTarget 组合的时候要小心。

> * afterDelay会增加receiver的引用计数，cancel则会对应减一
> * 如果在receiver的引用计数只剩下1 （仅为delay）时，调用cancel之后会立即销毁receiver，后续再调用receiver的方法就会crash

```
__weak typeof(self) weakSelf = self;
[NSObject cancelPreviousPerformRequestsWithTarget:self];
if (!weakSelf)
{
//NSLog(@"self被销毁");
    return;
}

[self doOther];
```
