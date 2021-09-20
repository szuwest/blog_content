Date: 2018-02-26
Title: react-native框架源码学习(iOS)(上)
Category: 技术
Tags: ReactNative 源码 iOS



# react-native框架源码学习(iOS)(上)

## 注意事项
要学习react-native框架iOS端源码，先要了解一下注意的事项。

* 一、 首先需要非常熟悉objective-c语言，对OC的runtime机制更是必不可少。源码里大量用了runtime特性。然后也要熟悉c++语言。除了OC语言，里面还有大量的c++代码，OC与c++混合代码。对c++的template和std库也要熟悉。我本人对c++也熟悉，但是不够深入，对一些高级特性不是很了解，所以相关的一些代码看起来很吃力。最后当然也熟悉js语言。我大学时学习过js，但是那还真是远古时代了，js近年来发展很快，我对很多新特性不是很了解，这也对我学习有一定的阻碍。另外还有要比较了解iOS的JavaScriptCore框架，这个是js与OC通信的基础。
* 二、 要对react-native的版本了解。查看网上的资料时也要注意他们所说的RN版本。现在网上很多研究资料，他们所研究的版本都是很早之前的版本。新的版本跟老版本有很多细节和流程改变了。如果你发现有些文章里说的东西跟你所看到的对不上，那就是研究的版本不一致。
* 三、如果看不懂，多看看别人的分析文章。如果对某种语言或者框架库不熟悉，建议先把该补的补上。

我所研究的RN版本是**0.49**，而最新版是0.51. 最后我自己也不敢说自己完全看懂了源码，自己的理解完全正确，所以下面所说的也不一定完全正确。
## 核心原理
主要原理利用JavaScriptCore的通信机制来做一些基本的js和OC相互调用，但是RN还做了更多事情。
OC会将所有要暴露给JS调用的方法和属性生成一个配置表，然后会将这个配置表写入到JS端。RN中OC和JS都分别有一个桥接对象，他们相互调用都是通过这个桥接对象进行。
RN大致结构图：
![RN结构图](https://upload-images.jianshu.io/upload_images/1271831-7ba4d20000946a6b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

## 启动流程
研究源码当然得从启动流程开始。新建一个RN工程，它的AppDelagte入口方法代码大致如下：

````
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
  
  NSURL *jsCodeLocation;

  jsCodeLocation = [[RCTBundleURLProvider sharedSettings] jsBundleURLForBundleRoot:@"index.ios" fallbackResource:nil];

  RCTRootView *rootView = [[RCTRootView alloc] initWithBundleURL:jsCodeLocation
                                                      moduleName:@"VprScene"
                                               initialProperties:nil
                                                   launchOptions:launchOptions];
  rootView.backgroundColor = [[UIColor alloc] initWithRed:0.0/255 green:0.0/255 blue:0.0/255 alpha:1];

  self.window = [[UIWindow alloc] initWithFrame:[UIScreen mainScreen].bounds];
  UIViewController *rootViewController = [UIViewController new];
  rootViewController.view = rootView;
  self.window.rootViewController = rootViewController;
  [self.window makeKeyAndVisible];

  return YES;
}
````
最主要的代码就是初始化了RCTRootView，别的都是普通的代码。也就是说由RCTRootView来进行整个RN的初始化。

RCTRootView的initWithBundleURL:方法中初始化了一个RCTBridge，RCTBridge的初始化方法里调用了一个setUp方法，在这里又初始化了一个RCTCxxBridge（早期版本是RCTBatchedBridge，它们都是RCTBridge的子类）对象，然后调用了它的start方法。这是个关键方法。RCTBridge与RCTCxxBridge类似一个代理模式，RCTBridge提供了对外接口，实际上调用了RCTCxxBridge的实现。这样的好处是方便替换内部实现，而接口保持稳定。

RCTCxxBridge的start方法(为了方便理解，删掉了不必要的代码):

````
- (void)start
{
  ......

  // Set up the JS thread early
  _jsThread = [[NSThread alloc] initWithTarget:[self class]
                                      selector:@selector(runRunLoop)
                                        object:nil];
  _jsThread.name = RCTJSThreadName;
  _jsThread.qualityOfService = NSOperationQualityOfServiceUserInteractive;
  [_jsThread start];

  dispatch_group_t prepareBridge = dispatch_group_create();

  [self registerExtraModules];
  // Initialize all native modules that cannot be loaded lazily
  [self _initModules:RCTGetModuleClasses() withDispatchGroup:prepareBridge lazilyDiscovered:NO];

  // This doesn't really do anything.  The real work happens in initializeBridge.
  _reactInstance.reset(new Instance);

  __weak RCTCxxBridge *weakSelf = self;

  // Prepare executor factory (shared_ptr for copy into block)
  std::shared_ptr<JSExecutorFactory> executorFactory;
  .....
    if (!executorFactory) {
      BOOL useCustomJSC =
        [self.delegate respondsToSelector:@selector(shouldBridgeUseCustomJSC:)] &&
        [self.delegate shouldBridgeUseCustomJSC:self];
      // The arg is a cache dir.  It's not used with standard JSC.
      executorFactory.reset(new JSCExecutorFactory(folly::dynamic::object
        ("OwnerIdentity", "ReactNative")
        ("UseCustomJSC", (bool)useCustomJSC)
      ));
    }
  ......
  // Dispatch the instance initialization as soon as the initial module metadata has
  // been collected (see initModules)
  dispatch_group_enter(prepareBridge);
  [self ensureOnJavaScriptThread:^{
    [weakSelf _initializeBridge:executorFactory];
    dispatch_group_leave(prepareBridge);
  }];

	.....
    // Load the source asynchronously, then store it for later execution.
    dispatch_group_enter(prepareBridge);
    __block NSData *sourceCode;
    [self loadSource:^(NSError *error, RCTSource *source) {
      if (error) {
        [weakSelf handleError:error];
      }

      sourceCode = source.data;
      dispatch_group_leave(prepareBridge);
    } onProgress:^(RCTLoadingProgress *progressData) {
	.....
    }];

    // Wait for both the modules and source code to have finished loading
    dispatch_group_notify(prepareBridge, dispatch_get_global_queue(QOS_CLASS_USER_INTERACTIVE, 0), ^{
      RCTCxxBridge *strongSelf = weakSelf;
      if (sourceCode && strongSelf.loading) {
        [strongSelf executeSourceCode:sourceCode sync:NO];
      }
    });
.....
}

````

这个方法很复杂，主要做了以下内容：

* 创建并开启了一个js线程，绑定了一个runloop，也即是说js代码都是在这个线程里执行。
* 准备所有要暴露给js调用的OC类,ModuleClass，为每个类封装到RCTModuleData里，如果需要在主线程中是创建某些类的实例，则会在主线程中去创建它。这些RCTModuleData会分别存储在一个字典和数组里。
* 准备JS运行环境，初始化JSExecutorFactory，并在js线程中创建js的RCTMessageThread，初始化_reactInstance（Instance，这是native跟jsBridge的桥梁）和JSCExecutor
* 加载JS源码
* 以上全做完之后，执行JS源码

注意代码里用到一个叫prepareBridge的dispatch_group_t，dispatch_group_t主要是用来做异步代码同步用的。有很多初始化工作是异步并行的，运行源码肯定是在所有准备工作之后才能进行，所以用了dispatch_group_t和dispatch_group_notify机制来确保这个问题。接下来我们逐步去分析上面说的几个事情。

### JS线程
大家常说JavaScript是单线程的，在RN里就是这样的。它创建了一个线程，然后绑定到了一个runloop，然后JS就是在这个线程里执行。在RCTCxxBridge里专门有个方法将block放在JS线程中执行

````
/**
 * Ensure block is run on the JS thread. If we're already on the JS thread, the block will execute synchronously.
 * If we're not on the JS thread, the block is dispatched to that thread. Any errors encountered while executing
 * the block will go through handleError:
 */
- (void)ensureOnJavaScriptThread:(dispatch_block_t)block
{
  RCTAssert(_jsThread, @"This method must not be called before the JS thread is created");

  // This does not use _jsMessageThread because it may be called early before the runloop reference is captured
  // and _jsMessageThread is valid. _jsMessageThread also doesn't allow us to shortcut the dispatch if we're
  // already on the correct thread.

  if ([NSThread currentThread] == _jsThread) {
    [self _tryAndHandleError:block];
  } else {
    [self performSelector:@selector(_tryAndHandleError:)
          onThread:_jsThread
          withObject:block
          waitUntilDone:NO];
  }
}
````
除此之外还有个RCTMessageThread类，专门用于在js线程中执行c++的同步和异步函数。

### 获取ModuleClass
**[self _initModules:RCTGetModuleClasses() withDispatchGroup:prepareBridge lazilyDiscovered:NO]**方法就是获取所有要暴露的类并做相应初始化。首先了解一下要将一个类暴露给JS需要怎么做。我们已RN里已经有的一个粘贴板类来看一下。

````
//RCTClipboard.h
@interface RCTClipboard : NSObject <RCTBridgeModule>

@end

//RCTClipboard.m
@implementation RCTClipboard

RCT_EXPORT_MODULE()

- (dispatch_queue_t)methodQueue
{
  return dispatch_get_main_queue();
}


RCT_EXPORT_METHOD(setString:(NSString *)content)
{
  UIPasteboard *clipboard = [UIPasteboard generalPasteboard];
  clipboard.string = (content ? : @"");
}

RCT_EXPORT_METHOD(getString:(RCTPromiseResolveBlock)resolve
                  rejecter:(__unused RCTPromiseRejectBlock)reject)
{
  UIPasteboard *clipboard = [UIPasteboard generalPasteboard];
  resolve((clipboard.string ? : @""));
}

@end

````

其实主要关键点就是类需要实现协议RCTBridgeModule，并在implementation里加RCT_EXPORT_MODULE()。在要暴露的方法在其前面加宏RCT_EXPORT_METHOD修饰。注意这里不支持有返回值的方法，还有也可以暴露一些属性的，但是这里不展开说明。

RCTBridgeModule协议定义了+ (NSString *)moduleName这个必须实现的方法，还有其他可选方法和属性。
看一下RCT_EXPORT_MODULE()宏到底干了什么

````
/**
 * Place this macro in your class implementation to automatically register
 * your module with the bridge when it loads. The optional js_name argument
 * will be used as the JS module name. If omitted, the JS module name will
 * match the Objective-C class name.
 */
#define RCT_EXPORT_MODULE(js_name) \
RCT_EXTERN void RCTRegisterModule(Class); \
+ (NSString *)moduleName { return @#js_name; } \
+ (void)load { RCTRegisterModule(self); }

````
这个宏里实现moduleName方法，定义了RCT_EXTERN void RCTRegisterModule(Class)，还有实现了load方法，在load方法里调用了RCTRegisterModule方法。RCTRegisterModule实现是在RCTBridge.m里面。

````
static NSMutableArray<Class> *RCTModuleClasses;
NSArray<Class> *RCTGetModuleClasses(void)
{
  return RCTModuleClasses;
}

/**
 * Register the given class as a bridge module. All modules must be registered
 * prior to the first bridge initialization.
 */
void RCTRegisterModule(Class);
void RCTRegisterModule(Class moduleClass)
{
  static dispatch_once_t onceToken;
  dispatch_once(&onceToken, ^{
    RCTModuleClasses = [NSMutableArray new];
  });

  RCTAssert([moduleClass conformsToProtocol:@protocol(RCTBridgeModule)],
            @"%@ does not conform to the RCTBridgeModule protocol",
            moduleClass);

  // Register module
  [RCTModuleClasses addObject:moduleClass];
}
````
实际上就是讲Class对象加入到全局唯一的一个数组里。也就是说，当系统在加载这个类的时候，就将这个类对象加入到了一个数组里，后面只要从这个数组里取所有要暴露给JS的类就可以了。

再来看看怎么暴露方法。

````
#define RCT_EXPORT_METHOD(method) \
  RCT_REMAP_METHOD(, method)
  

#define RCT_REMAP_METHOD(js_name, method) \
  _RCT_EXTERN_REMAP_METHOD(js_name, method, NO) \
  - (void)method;


#define _RCT_EXTERN_REMAP_METHOD(js_name, method, is_blocking_synchronous_method) \
  + (const RCTMethodInfo *)RCT_CONCAT(__rct_export__, RCT_CONCAT(js_name, RCT_CONCAT(__LINE__, __COUNTER__))) { \
    static RCTMethodInfo config = {#js_name, #method, is_blocking_synchronous_method}; \
    return &config; \
  }
  
````
这里的宏嵌套得让人眼花缭乱，在整个RN源码中运用了大量的宏，而且大多是高级用法，这个给阅读理解造成一定的困难。如果看不懂，最好先找专门的讲宏文章先看一看。这里主要作用就是声明了返回值为void的method方法，并且生成了一个以__rct_export__开头，包含了类名，行号等信息的为类名，返回值为RCTMethodInfo的类方法。这个方法主要是为后面生成配置信息用的。

**[self _initModules:RCTGetModuleClasses() withDispatchGroup:prepareBridge lazilyDiscovered:NO]**主要代码是

````
- (void)_initModules:(NSArray<id<RCTBridgeModule>> *)modules
   withDispatchGroup:(dispatch_group_t)dispatchGroup
    lazilyDiscovered:(BOOL)lazilyDiscovered
{
....
// Set up moduleData for automatically-exported modules
  NSArray<RCTModuleData *> *moduleDataById = [self registerModulesForClasses:modules];
  ......
   [self _prepareModulesWithDispatchGroup:dispatchGroup];
}
````

registerModulesForClasses:方法主要是遍历RCTModuleClasses，为每个class生成一个RCTModuleData并在一个字典和数组里存起来后面使用。_prepareModulesWithDispatchGroup:就是检查每个RCTModuleData是否需要在主线程中创建实例，是的话，就创建实例存起来。

### 准备JS相关类（JSCExecutor）
在start方法里面，它实例化了Instance(_reactInstance)和一个JSExecutorFactory，然后在js线程中调用了_initializeBridge:方法。

````
- (void)_initializeBridge:(std::shared_ptr<JSExecutorFactory>)executorFactory
{
.....
__weak RCTCxxBridge *weakSelf = self;
  _jsMessageThread = std::make_shared<RCTMessageThread>([NSRunLoop currentRunLoop], ^(NSError *error) {
    if (error) {
      [weakSelf handleError:error];
    }
  });
  .....
  if (_reactInstance) {
    // This is async, but any calls into JS are blocked by the m_syncReady CV in Instance
    _reactInstance->initializeBridge(
      std::make_unique<RCTInstanceCallback>(self),
      executorFactory,
      _jsMessageThread,
      [self _buildModuleRegistry]);
      ......
   }
}
````
可以看到这里主要创建了一个RCTMessageThread，并将相关属性传给_reactInstance进行初始化。_reactInstance的初始化需要一个ModuleRegistry，ModuleRegistry里面有包含了所有的RCTNativeModule，而每个RCTNativeModule里又包含了一个RCTModuleData.即将前面生成的所有RCTModuleData传给了_reactInstance。

在Instance的初始化方法initializeBridge里，最主要是创建了NativeToJsBridge的实例。而NativeToJsBridge的构造函数里，主要是调用了executorFactory来创建一个JSCExecutor。JSCExecutor是一个相当重要的类，走到这里很不容易，跳转了很多层。这一块也是最不好理解的，基本上c++和OC互调，而且使用了很多c++11的特性，我很不熟悉。然后也使用了folly库，这个库也不是很好理解。所以我准备把剩下的东西放到下一篇文章来讲。


## 参考资料

[深入浅出 JavaScriptCore](http://www.cocoachina.com/ios/20170720/19958.html)

[JavaScriptCore全面解析 （上篇）](https://cloud.tencent.com/developer/article/1004875)

[JavaScriptCore全面解析 （下篇）
](https://cloud.tencent.com/developer/article/1004876)

[理解React-Native(0.46) 中native和js通信原理(iOS)](https://www.jianshu.com/p/931367388a8d?utm_campaign=maleskine&utm_content=note&utm_medium=seo_notes&utm_source=recommendation)

[React Native通信机制详解](http://blog.cnbang.net/tech/2698/)



如果你觉得这篇文章有用，请打赏小钱喝杯咖啡^_^
![打赏](https://raw.githubusercontent.com/szuwest/szuwest.github.io/master/images/2018-02-21%20133111.jpg)