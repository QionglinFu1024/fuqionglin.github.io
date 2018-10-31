---
layout: post
title: 'NSOperation'
subtitle: 'NSOperation'
date: 2016-01-14
categories: 技术
cover: 
tags: 多线程
---

#### 初体验

* 执行步骤
    * 先将需要执行的操作封装到一个NSOperation对象中
    * 然后将NSOperation对象添加到NSOperationQueue中
    * 系统会自动将NSOperationQueue中的NSOperation取出来
    * 将取出的NSOperation封装的操作放到一条新的线程中执行
   
* NSInvocationOperation
 
<pre><code class="language-objectivec">- (void)demo1{
    
    // 不能直接用 --- Invocation事务 () + queue = 把事务添加到队列 ---> 然后去执行
    NSInvocationOperation *op = [[NSInvocationOperation alloc] initWithTarget:self selector:@selector(xixihha) object:nil];
    
    NSOperationQueue *queue = [[NSOperationQueue alloc] init];
    
    // 由系统调控
    [queue addOperation:op];
    
    // 手动吊起，直接放到线程操作，单独制定某个事物执行，已经加到队列的事物不能start
    [op start];
    
}

// 操作
- (void)xixihha{
    NSLog(@"123");
}

</code></pre>

* NSBlockOperation

<pre><code class="language-objectivec">- (void)demo2{
    
    // 操作优先级
    NSBlockOperation *op = [NSBlockOperation blockOperationWithBlock:^{
       //子线程执行
        NSLog(@"NSThread == %@",[NSThread currentThread]);
        // 和addExecutionBlock并发
        [NSThread sleepForTimeInterval:1];
        NSLog(@"123");
        //通讯
        [[NSOperationQueue mainQueue] addOperationWithBlock:^{
            NSLog(@"更新UI");
        }];
        
    }];
    
    // 不是越高越先执行完毕。只是CPU调度的频率高
//    op.qualityOfService =
    //添加执行代码
    [op addExecutionBlock:^{
        NSLog(@"456");
    }];
    //事物操作完成后回调
    op.completionBlock = ^{
        NSLog(@"完成");
    };
    
    NSOperationQueue *queue = [[NSOperationQueue alloc] init];
    // 由系统调控
    [queue addOperation:op];
    
    
}
</code></pre>

#### 属性研究

<pre><code class="language-objectivec">#import "ViewController.h"

@interface ViewController ()
@property (nonatomic, strong) NSOperationQueue *queue;
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    self.queue = [[NSOperationQueue alloc] init];
    
    [self demo2];
}


- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
    [self demo2];
}


- (void)demo2{
    
    self.queue.maxConcurrentOperationCount = 2;
    NSBlockOperation *bo1 = [NSBlockOperation blockOperationWithBlock:^{
        [NSThread sleepForTimeInterval:0.5];
        NSLog(@"请求token");
    }];
    
    NSBlockOperation *bo2 = [NSBlockOperation blockOperationWithBlock:^{
        [NSThread sleepForTimeInterval:0.5];
        NSLog(@"拿着token,请求数据1");
    }];
    
    NSBlockOperation *bo3 = [NSBlockOperation blockOperationWithBlock:^{
        [NSThread sleepForTimeInterval:0.5];
        NSLog(@"拿着数据1,请求数据2");
    }];
    
    //依赖，控制线程执行顺序
    [bo2 addDependency:bo1];
    [bo3 addDependency:bo2];
    
    [self.queue addOperations:@[bo1,bo2,bo3] waitUntilFinished:NO];
 
    NSLog(@"执行完了?我要干其他事");
}

/**
 关于operationQueue的挂起,继续,取消
 */
- (void)demo1{
    
    // GCD ---> 信号量  :  对于线程操作更自如  -- suspernd  cancel finish
    // 多线程世界
    self.queue.name = @"com.cooci.cn";
    self.queue.maxConcurrentOperationCount = 6;
    [self.queue addOperationWithBlock:^{
        for (int i = 0; i<1000; i++) {
            [NSThread sleepForTimeInterval:1];
            NSLog(@"%@--%d",[NSThread currentThread],i);
        }
    }];
    
}

- (IBAction)pauseOrContinue:(id)sender {

    // 下载任务 ---> task ---> 挂起
    // 继续 ---->
    // 打断  --->  后台
    
    if (self.queue.operationCount == 0) {
        NSLog(@"没有操作执行");
        return;
    }
    
    if (self.queue.suspended) {
        NSLog(@"当前挂起来了");
    }else{
        NSLog(@"执行....");
    }
    
    self.queue.suspended = !self.queue.isSuspended;
    
}

- (IBAction)cancel:(id)sender {
    
    [self.queue cancelAllOperations];
}

@end
</code></pre>

#### 缓存机制

* 首先加载内存，因为快
* 在加载磁盘，找到以后保存到内存
* 下载

#### 自定义NSOperation

* 非并发只需要实现main方法
* 并发

<pre><code class="language-objectivec">#import "KCWebImageDownloadOperation.h"
#import "NSString+KCAdd.h"

@interface KCWebImageDownloadOperation()
@property (nonatomic, readwrite, getter=isExecuting) BOOL executing;
@property (nonatomic, readwrite, getter=isFinished) BOOL finished;
@property (nonatomic, readwrite, getter=isCancelled) BOOL cancelled;
@property (nonatomic, readwrite, getter=isStarted) BOOL started;
@property (nonatomic, strong) NSRecursiveLock *lock;
@property (nonatomic, copy) NSString *urlString;
@property (nonatomic, copy) KCCompleteHandle completeHandle;
@property (nonatomic, copy) NSString *title;

@end

@implementation KCWebImageDownloadOperation
// 因为父类的属性是Readonly的，重载时如果需要setter的话则需要手动合成。
@synthesize executing = _executing;
@synthesize finished = _finished;
@synthesize cancelled = _cancelled;

- (instancetype)initWithDownloadImageUrl:(NSString *)urlString completeHandle:(KCCompleteHandle)completeHandle title:(NSString *)title{
    if (self = [super init]) {
        _urlString = urlString;
        _executing = NO;
        _finished  = NO;
        _cancelled = NO;
        _lock      = [NSRecursiveLock new];
        _completeHandle = completeHandle;
        _title = title;
    }
    return self;
}

- (void)start{
    /*
    延长生命周期
    大型循环
    自己创建管理线程
     */
    @autoreleasepool{
        [_lock lock];
        
        self.started = YES;
        [NSThread sleepForTimeInterval:1.5];
        NSLog(@"下载 %@",self.title);
        if (self.cancelled) {
            NSLog(@"取消下载 %@",self.title);
            return;
        }
        NSURL   *url   = [NSURL URLWithString:self.urlString];
        NSData  *data  = [NSData dataWithContentsOfURL:url]; // 这里不完美 等到后面写网络 直接写在task里面 网络请求的回调里面
        if (self.cancelled) {
            NSLog(@"取消下载 %@",self.title);
            return;
        }
        if (data) {
            NSLog(@"下载完成: %@",self.title);
            [data writeToFile:[self.urlString getDowloadImagePath] atomically:YES];
            [self done];
            self.completeHandle(data,self.urlString);
        }else{
            NSLog(@"下载图片失败");
        }
        [_lock unlock];
    }
}

- (void)done{
    self.finished = YES;
    self.executing = NO;
}

/**
 取消操作的方法 --- 需要进行判断
 */
- (void)cancel{
    
    [_lock lock];
    if (![self isCancelled]) {
        //关掉其他状态
        [super cancel];
        self.cancelled = YES;
        if ([self isExecuting]) {
            self.executing = NO;
        }
        if (self.started) {
            self.finished = YES;
        }
    }
    [_lock unlock];
}

#pragma mark - setter -- getter

- (BOOL)isExecuting {
    [_lock lock];
    BOOL executing = _executing;
    [_lock unlock];
    return executing;
}

- (void)setFinished:(BOOL)finished {
    [_lock lock];
    if (_finished != finished) {
        [self willChangeValueForKey:@"isFinished"];
        _finished = finished;
        [self didChangeValueForKey:@"isFinished"];
    }
    [_lock unlock];
}

- (BOOL)isFinished {
    [_lock lock];
    BOOL finished = _finished;
    [_lock unlock];
    return finished;
}

- (void)setCancelled:(BOOL)cancelled {
    [_lock lock];
    if (_cancelled != cancelled) {
        [self willChangeValueForKey:@"isCancelled"];
        _cancelled = cancelled;
        [self didChangeValueForKey:@"isCancelled"];
    }
    [_lock unlock];
}

- (BOOL)isCancelled {
    [_lock lock];
    BOOL cancelled = _cancelled;
    [_lock unlock];
    return cancelled;
}
- (BOOL)isConcurrent {
    return YES;
}
- (BOOL)isAsynchronous {
    return YES;
}

+ (BOOL)automaticallyNotifiesObserversForKey:(NSString *)key {
    if ([key isEqualToString:@"isExecuting"] ||
        [key isEqualToString:@"isFinished"] ||
        [key isEqualToString:@"isCancelled"]) {
        return NO;
    }
    //关闭自动监听
    return [super automaticallyNotifiesObserversForKey:key];
}



@end
</code></pre>




