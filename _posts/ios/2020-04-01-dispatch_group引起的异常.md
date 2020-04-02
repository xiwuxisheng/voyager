---
layout: post
title:  dispatch_group同步异常问题
categories: iOS
description: dispatch_group同步异常问题
keywords: dispatch_group, 异常, 多线程, 同步

---

### 引言

这次需要在这里梳理一下使用dispatch_group_t引起的同步异常问题，具体问题请看下面:

```
#0 Thread
SIGSEGV
SEGV_ACCERR

libdispatch.dylib   _dispatch_group_leave$VARIANT$armv81 + 8
Shikamaru   __57-[SKMCommunityCircleListViewController loadCircleDataNum]_block_invoke (SKMCommunityCircleListViewController.m:77)
Shikamaru   __43-[SKMCirclePageViewModel requestCircleNum:]_block_invoke (SKMCirclePageViewModel.m:218)
Shikamaru   __48-[YTKNetworkAgent requestDidSucceedWithRequest:]_block_invoke (YTKNetworkAgent.m:387)
libdispatch.dylib   __dispatch_call_block_and_release + 24
libdispatch.dylib   __dispatch_client_callout + 16
libdispatch.dylib   __dispatch_main_queue_callback_4CF$VARIANT$armv81 + 1008
CoreFoundation  ___CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__ + 12
CoreFoundation  ___CFRunLoopRun + 1924
CoreFoundation  CFRunLoopRunSpecific + 436
GraphicsServices    GSEventRunModal + 104
UIKitCore   UIApplicationMain + 212
Shikamaru   main (main.m:14)
libdyld.dylib   _start + 4
```

从中可以看出这里出现的异常信号SIGSEGV属于野指针信号，意思是程序无效内存中止信号，一般是表示内存不合法，二是产生的错误信息，也指向 _dispatch_group_leave$VARIANT$armv81 + 8，为了解决这个问题，那么这里进行了如下的流程分析：

* 检查代码是否实现了dispatch_group_enter和dispatch_group_leave的配对；
* 存在请求回调block造成了dispatch_group_enter和dispatch_group_leave的配对数量差；
* dispatch_group_enter和dispatch_group_leave的配对数量差造成的异常与引言异常的流程对比；
* 其他使用dispatch_group未发生异常的比较；
* group为空造成的异常与引言异常的流程对比；
* 引起为空的条件和场景，实际测试；
* 整体项目整改；

---

### 分析过程

#### 代码逻辑配对分析

通过该控制器vc的整体代码逻辑分析，这里是将group对象当作全局对象，然后将每一个请求当作方法进行了拆分，主要是为了方法重用性和独立性，那么也就是说这种方式不太可能出现配对的误差。

```
- (void)loadAllData {
    self.circleVM.postNextId = nil;
    [self resetPlayerAction];
    [MBProgressHUD showMessage:@"请稍等..."];

    [self loadCircleData];

    [self loadPostDataByArticleId];

    [self loadPostData:YES];

    SKMWeakSelf
    dispatch_group_notify(self.group, dispatch_get_main_queue(), ^{
        SKMStrongSelf
        [MBProgressHUD hideHUD];
        [strongSelf.dataSource removeAllObjects];
        if (strongSelf.circleVM.circleList) {
            [strongSelf.dataSource addObject:strongSelf.circleVM.circleList];
        }
        if (strongSelf.headerModel) {
            [strongSelf.dataSource addObject:strongSelf.headerModel];
        }
        [strongSelf.dataSource addObjectsFromArray:strongSelf.circleVM.postArr];
        strongSelf.dataArr = strongSelf.dataSource;
        [strongSelf actionWithRefresh];
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.5 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
           [strongSelf handleScrollPlay];
        });
    });
}
```

```
- (void)loadCircleData {
    dispatch_group_enter(self.group);
    SKMWeakSelf
    [self.circleVM requestCircleFirstPageList:^(SKMCircleInfoListModel *circleList, NSError * _Nonnull error) {
        SKMStrongSelf
        if (strongSelf) {
            dispatch_group_leave(strongSelf.group);
        }
    }];
}
```
剩下的就不一一举出了，看得出来代码逻辑上配对是没有问题的。

----

#### 运行实际的配对分析

这里的本意是存在请求的block回调没有调用或者多次调用造成了配对误差，这种猜想是根据如下信息资料来的。

```
NSArray *imageURLArray = @[@"1", @"2", @"3", @"4"];
dispatch_group_t group = dispatch_group_create();
[imageURLArray enumerateObjectsUsingBlock:^(id  _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
        dispatch_group_enter(group);
        [[SDWebImageDownloader sharedDownloader] downloadImageWithURL:[NSURL URLWithString:imageURLArray[idx]] options:SDWebImageDownloaderLowPriority progress:nil completed:^(UIImage * _Nullable image, NSData * _Nullable data, NSError * _Nullable error, BOOL finished) {
            dispatch_group_leave(group);
            NSLog(@"idx:%zd",idx);
        }];
    }];
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        NSLog(@"%@", imageURLArray);
    });
```

> 代码逻辑可以说是很简单：一个图片URL数组。使用SDWebImage多线程进行并发下载，直到所有图片都下载完成进行回调。但是就是这样一段代码居然会偶尔出现崩溃，和项目中其他地方使用到dispatch_group的地方进行过比较，也没发现有什么不同，没办法这时候只有先去看看dispatch_group的源码了,其中有一段是这样的。

```
dispatch_group_leave(dispatch_group_t dg) {
    dispatch_semaphore_t dsema = (dispatch_semaphore_t)dg;
    dispatch_atomic_release_barrier();
    long value = dispatch_atomic_inc2o(dsema, dsema_value);
    if (slowpath(value == LONG_MIN)) {
        DISPATCH_CLIENT_CRASH("Unbalanced call to dispatch_group_leave()");
    }
    if (slowpath(value == dsema->dsema_orig)) {
        (void)_dispatch_group_wake(dsema);
    }
}
```

> 通过源代码我们发现在调用dispatch_group_leave的时候是可能会发生crash的，这段代码的重点就是当这个value值和LONG_MIN相等的时候，这里会发生crash。我们需要关注下LONG_MIN这个数字，LONG_MIN = -LONG_MAX - 1。在dispatch_group_create里面发现了它的踪影：

```
dispatch_group_t
dispatch_group_create(void){
    dispatch_group_t dg = _dispatch_alloc(DISPATCH_VTABLE(group),sizeof(struct dispatch_semaphore_s));
    _dispatch_semaphore_init(LONG_MAX, dg);
    return dg;
}
```

>这两段代码的结合告诉了我们一个事实：当dq这个信号量加一导致溢出后，dispatch_group_leave就会Crash。通过查阅SDWebImageDownloader.m源码发现：

```
dispatch_barrier_sync(self.barrierQueue, ^{
    SDWebImageDownloaderOperation *operation = self.URLOperations[url];
    if (!operation) {
    operation = createCallback();

    // !!!!!!!特别注意这行!!!!!!!!!
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
```

>SDWebImage的下载器会根据URL做下载任务对应NSOperation映射，也即之前创建的下载回调Block。好，就是这行导致Crash的发生。为什么呢？
因为SDWebImage的下载器会根据URL做下载任务对应NSOperation映射，相同的URL会映射到同一个未执行的NSOperation。那么通过代码我当A组图片下载完成后，相同的url 回调是B组内 而不是A组内。此时B的计数为4 。当B 图片下载完后，结束计数为 5 。因为B图片enter 的次数为4 ,leave 的次数为5 ,因此会崩溃！

下面我们在看代码分析里面使用组的情况是AFN请求，那么会不会AFN请求也发生了类似回调block替换的情况，请看源码。

```
- (void)setDelegate:(AFURLSessionManagerTaskDelegate *)delegate
            forTask:(NSURLSessionTask *)task
{
    NSParameterAssert(task);
    NSParameterAssert(delegate);

    [self.lock lock];
//注意这行，进行回调的代理使用了task.taskIdentifier标识
    self.mutableTaskDelegatesKeyedByTaskIdentifier[@(task.taskIdentifier)] = delegate;
    [self addNotificationObserverForTask:task];
    [self.lock unlock];
}
```

```
@interface NSURLSessionTask : NSObject <NSCopying, NSProgressReporting>
//task标识是一个数字型
@property (readonly)                 NSUInteger    taskIdentifier;    /* an identifier for this task, assigned by and unique to the owning session */
@property (nullable, readonly, copy) NSURLRequest  *originalRequest;  /* may be nil if this is a stream task */
@property (nullable, readonly, copy) NSURLRequest  *currentRequest;   /* may differ from originalRequest due to http server redirection */
@property (nullable, readonly, copy) NSURLResponse *response;         /* may be nil if no response has been received */
```

也就是说AFN的每个请求都会分配一个数字标识进行回调区分，不可能发生类似SDWebImage类似的情况。那么现在从原理上确认group请求的配对不会发生异常。

----

#### 异常结果对比

我们再确认一下配对造成的异常与实际异常结果的对比，可以在对应的请求里多进行一次dispatch_group_leave操作，造成crash，看看异常堆栈信息。

![spec仓库](/assets/images/iOS/dispatch_group_crash.jpeg)

> 看起来是不是与引言里的bug堆栈信息很类似，那就是配对问题，但是前面我们已经确认配对不可能出现问题，但是别急，我们还需要再确认一下异常信号信息。我们发现这里发生crash后抛出的是`Thread 1: EXC_BREAKPOINT (code=1, subcode=0x10b4abf20)`信息，意思是由断点指令或其它trap指令产生，产生了异常中断信号，不对呀，这与实际结果的野指针信号不匹配呀。所以我们可以进一步确认异常的产生不是因为配对的问题。

----

#### 与其他使用diaptch_group但未发生异常的对比

```
dispatch_group_t group = dispatch_group_create();
    SKMWeakSelf
    dispatch_group_enter(group);
    [self.viewModel requestSurroundList:self.params completeBlock:^(SkmrSurroundInfoRsp *skmrSurroundInfoRsp, NSError * _Nonnull error) {
        dispatch_group_leave(group);
        if (skmrSurroundInfoRsp) {
            SKMStrongSelf
            strongSelf.skmrSurroundInfoRsp = skmrSurroundInfoRsp;
            [strongSelf.tableView reloadData];
        }
    }];
    dispatch_group_enter(group);
    [self.viewModel requestSearchCommunityList:self.params completeBlock:^(SkmrCommunityListRsp *skmrCommunityListRsp, NSError * _Nonnull error) {
        dispatch_group_leave(group);
        SKMStrongSelf
        if (skmrCommunityListRsp) {
            strongSelf.skmrCommunityListRsp = skmrCommunityListRsp;
            [strongSelf.tableView reloadData];
        }
    }];
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        SKMStrongSelf
        [strongSelf judgeNoResultAction];
        [strongSelf actionWithRefresh:YES];
    });
 ```

>上述是未发生异常的 dispatch_group的使用用法,是不是已经发现问题了，一个是局部定义的group，产生问题的使用的是全局定义的group,多了一个当前self的强引用。那么可以猜想的问题就只剩下一个了那就是self有可能为nil,那么下一步就是进行验证了，执行`dispatch_group_leave(nil)`。

#### 验证group为nil的结果对比

![spec仓库](/assets/images/iOS/dispatch_group_nil.jpeg)

>从图中也可以看出发生crash的流程一致，而且发出的异常信号是`EXC_BAD_ACCESS`，这是一个非法访问已经释放内存区域的异常，与原结论的SIGSEGV信号一致，那么大体上可以确认该异常起因就是这个。

----

#### 引起为空的条件和场景

上述告知了我们引起异常的原因是`dispatch_group_leave(nil)`了，那么为什么group为nil，也就是为什么self为nil，什么况下self为nil了但是block还会执行，加上这是在一个请求方法里面，那么就很容易猜想，控制器vc本身(self)推出释放了，但是请求已经发出，当然block回调还是会继续执行。而且要达到这种效果要么就是用户很快进来，未等请求完成立马退出，但是平常我们测试的网速都是很快的，要想实践出来，当然必须要靠弱网测试了。

![spec仓库](/assets/images/iOS/dispatch_group_weaknet.jpeg)

>图中可以看到设置成弱网请求后确实可以达成实现了我们的猜想，也复现了异常产生的场景。那么下一步就是如何解决了。实际上很简单了，那就是在执行`dispatch_group_leave(nil)`前我们先要做self的空判断，不为空则执行，否则直接跳过，经实际测试，可行，app正常运行。至此，关于异常的处理基本结束。

----

#### 完整项目收尾

上面已经分析了异常的分析流程、场景和解决方案，那么为了全面考虑，是否存在漏网之鱼，所以需要整体项目更改，但是这里为什么全局的group和局部定义的group区别这么大，按照原理是方法执行完后，局部变量自动释放，生命周期只在该方法内，所以用弱网环境验证一下，我们执行同样进来控制器vc不等请求完成立马退出，断点，结果如下。

![spec仓库](/assets/images/iOS/dispatch_group_aspect.jpeg)

>OK，发现控制器释放了，但是group依然存在，那么说明局部变量应该是block在运行时直接将group复制到了堆内存，其内存管理由block生命周期管理，而前者全局变量group由vc进行了强引用，其生命周期由vc控制，vc释放了，group自然也销毁了。所以两者才会存在同一场景上造成结果的不同。

----

### 总结

* dispatch_group_enter和dispatch_group_leave的配对必须配对。
* 注意使用dispatch_group时第三方库的影响。
* 注意group生命周期的管理，特别是弱网环境下的影响。
