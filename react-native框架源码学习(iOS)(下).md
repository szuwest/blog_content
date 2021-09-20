Date: 2018-03-2
Title: react-native框架源码学习(iOS)(下)
Category: 技术
Tags: ReactNative 源码 iOS



# react-native框架源码学习(iOS)(下)

如果没有看过上篇，请先看[react-native框架源码学习(iOS)(上)]()。

### JSCExecutor相关初始化
在上篇中说到Instance的初始化方法initializeBridge里，最主要是创建了NativeToJsBridge的实例。而NativeToJsBridge的构造函数里，主要是调用了executorFactory来创建一个JSCExecutor。NativeToJsBridge的构造函数还包含更多的东西。

````c++
NativeToJsBridge::NativeToJsBridge(
    JSExecutorFactory* jsExecutorFactory,
    std::shared_ptr<ModuleRegistry> registry,
    std::shared_ptr<MessageQueueThread> jsQueue,
    std::shared_ptr<InstanceCallback> callback)
    : m_destroyed(std::make_shared<bool>(false))
    , m_delegate(std::make_shared<JsToNativeBridge>(registry, callback))
    , m_executor(jsExecutorFactory->createJSExecutor(m_delegate, jsQueue))
    , m_executorMessageQueueThread(std::move(jsQueue)) {}
````

NativeToJsBridge的成员变量包括了一个m_delegate(JsToNativeBridge)，一个m_executorMessageQueueThread（MessageQueueThread）和m_executor（JSCExecutor），在它的构造函数里，将JsToNativeBridge创建了然后又传给jsExecutorFactory来创建JSCExecutor，JsToNativeBridge实际上是给JSCExecutor使用的。

````
JSCExecutor::JSCExecutor(std::shared_ptr<ExecutorDelegate> delegate,
                         std::shared_ptr<MessageQueueThread> messageQueueThread,
                         const folly::dynamic& jscConfig) throw(JSException) :
    m_delegate(delegate),
    m_messageQueueThread(messageQueueThread),
    m_nativeModules(delegate ? delegate->getModuleRegistry() : nullptr),
    m_jscConfig(jscConfig) {
  initOnJSVMThread();

  {
    SystraceSection s("nativeModuleProxy object");
    installGlobalProxy(m_context, "nativeModuleProxy",
                       exceptionWrapMethod<&JSCExecutor::getNativeModule>());
  }
}

void JSCExecutor::initOnJSVMThread() throw(JSException) {
	......
	m_context = JSC_JSGlobalContextCreateInGroup(useCustomJSC, nullptr, globalClass);
	.....
	installNativeHook<&JSCExecutor::nativeFlushQueueImmediate>("nativeFlushQueueImmediate");
  installNativeHook<&JSCExecutor::nativeCallSyncHook>("nativeCallSyncHook");
  ......
}
````
在initOnJSVMThread中，先创建了全局的JSGlobalContext,然后给这个context注册了nativeFlushQueueImmediate和nativeCallSyncHook回调函数，这两个函数专门给JS调用。这两个函数都将会在MessageQueue.js中被调用。到这里终于有看到JS相关的调用，这里说明一下核心的JS源码存放目录。在工程目录/node_modules/react-native/Libraries/BatchedBridge目录下，主要有BatchedBridge.js,MessageQueue.js,NativeModules.js三个文件。

在创建context的时候，使用了宏，最终会调用JavaScriptCore的代码。这个全局的context很重要，OC中要运行js代码就是靠他。而js代码中的global对象对应就是OC中的这个context。而nativeFlushQueueImmediate这个方法主要是将js传过来的队列里面的需要调用的方法一起调用了。nativeCallSyncHook这个方式主要是给JS直接同步调用OC方法来用的。为什么这么做后面再说，我们现在只需知道在创建context之后，就已经将OC的这两个方法注入到js的global对象中。

initOnJSVMThread()执行之后，往m_context中注册了nativeModuleProxy的JS对象，将这个对象绑定到了getNativeModule这个函数。

````
installGlobalProxy(m_context, "nativeModuleProxy",
                       exceptionWrapMethod<&JSCExecutor::getNativeModule>());
````

那么nativeModuleProxy在JS中到底是什么东西？这个定义在NativeModules.js中。

````js
let NativeModules : {[moduleName: string]: Object} = {};
if (global.nativeModuleProxy) {
  NativeModules = global.nativeModuleProxy;
} else {
	·······
}

module.exports = NativeModules;
````
我们可以看到NativeModules就是指向了nativeModuleProxy，即就是OC的代理。我们要调用OC某个类的方法，就是通过NativeModules来调用的。例如我们已经定义好另一个对JS暴露的录音类AudioRecorder,那么在JS中要调用这个类的方法，就可以 let AudioRecorder = NativeModules.AudioRecorder; AudioRecorder.record();这样调用。那么我们知道它其实是调用到了OC的JSCExecutor中的getNativeModule方法，getNativeModule方法其实主要作用根据传入的模块名字生成对应的模块配置，具体实现后面再说。这里说一下JS代码里有个else分支，这是NativeModules生成的另一种做法：把所有的OC要暴露的类和方法都存在remoteModuleConfig里，然后他们注入到NativeModules里（在OC里对应的代码在RCTObjcExecutor的构造函数里）。

到这里JSCExecutor的初始化完成。我们重新回到RCTCxxBridge的start方法。跟JSCExecutor同时进行的是加载js源码，jsBundle的加载是通过RCTJavaScriptLoader进行的，这里不进行讨论。当初始化和js源码加载完成后，就会执行js源码。

### js源码执行
在初始化过程中我们已经创建好全局的JavaScriptCore context，并在这个context中注入了nativeFlushQueueImmediate，nativeCallSyncHook，getNativeModule三个回调方法。现在将执行已加载的JS源码。它是执行链是这样的

* [RCTCxxBridge executeSourceCode: ]
* [RCTCxxBridge enqueueApplicationScript:]
   * void Instance::loadScriptFromString()
   * void NativeToJsBridge::loadApplication()
      * void JSCExecutor::loadApplicationScript()
      * void JSCExecutor::flush()
      * void JSCExecutor::bindBridge()


在 void JSCExecutor::loadApplicationScript方法最重要的代码如下


````
......
evaluateScript(m_context, jsScript, jsSourceURL);
.....
flush();

````
执行在context中执行JS源码，会初始化JS环境，BatchedBridge.js,NativeModules.js中的初始化代码也会执行。在BatchedBridge.js中，创建了一个名为BatchedBridge的MessageQueue，并设置到global的__fbBatchedBridge属性里，这个属性后面会用到。在初始化JS环境的时候，会加载到某些NativeModule，这些module才会被初始化，即调用到OC的getNativeModule方法。例如我打断点捕获到最开始初始化的一个NativeModule是PlatformConstants,它对应的OC类是RCTPlatform。当相关的Module都加载完之后，evaluateScript方法执行完，JS环境初始化完毕。然后就到执行flush方法。

````
void JSCExecutor::flush() {
  SystraceSection s("JSCExecutor::flush");

  if (m_flushedQueueJS) {
    callNativeModules(m_flushedQueueJS->callAsFunction({}));
    return;
  }

  // When a native module is called from JS, BatchedBridge.enqueueNativeCall()
  // is invoked.  For that to work, require('BatchedBridge') has to be called,
  // and when that happens, __fbBatchedBridge is set as a side effect.
  auto global = Object::getGlobalObject(m_context);
  auto batchedBridgeValue = global.getProperty("__fbBatchedBridge");
  // So here, if __fbBatchedBridge doesn't exist, then we know no native calls
  // have happened, and we were able to determine this without forcing
  // BatchedBridge to be loaded as a side effect.
  if (!batchedBridgeValue.isUndefined()) {
    // If calls were made, we bind to the JS bridge methods, and use them to
    // get the pending queue of native calls.
    bindBridge();
    callNativeModules(m_flushedQueueJS->callAsFunction({}));
  } else if (m_delegate) {
    // If we have a delegate, we need to call it; we pass a null list to
    // callNativeModules, since we know there are no native calls, without
    // calling into JS again.  If no calls were made and there's no delegate,
    // nothing happens, which is correct.
    callNativeModules(Value::makeNull(m_context));
  }
}

````
这是flush方法被第一次执行，所以m_flushedQueueJS为空，然后取到JS中的global对象中的__fbBatchedBridge对象。我们知道在JS初始化的时候，这个值已经被填上MessageQueue，所以这里会进入bindBridge()方法。

````
void JSCExecutor::bindBridge() throw(JSException) {
  SystraceSection s("JSCExecutor::bindBridge");
  std::call_once(m_bindFlag, [this] {
    auto global = Object::getGlobalObject(m_context);
    auto batchedBridgeValue = global.getProperty("__fbBatchedBridge");
    if (batchedBridgeValue.isUndefined()) {
      auto requireBatchedBridge = global.getProperty("__fbRequireBatchedBridge");
      if (!requireBatchedBridge.isUndefined()) {
        batchedBridgeValue = requireBatchedBridge.asObject().callAsFunction({});
      }
      if (batchedBridgeValue.isUndefined()) {
        throw JSException("Could not get BatchedBridge, make sure your bundle is packaged correctly");
      }
    }

    auto batchedBridge = batchedBridgeValue.asObject();
    m_callFunctionReturnFlushedQueueJS = batchedBridge.getProperty("callFunctionReturnFlushedQueue").asObject();
    m_invokeCallbackAndReturnFlushedQueueJS = batchedBridge.getProperty("invokeCallbackAndReturnFlushedQueue").asObject();
    m_flushedQueueJS = batchedBridge.getProperty("flushedQueue").asObject();
    m_callFunctionReturnResultAndFlushedQueueJS = batchedBridge.getProperty("callFunctionReturnResultAndFlushedQueue").asObject();
  });
}

````

从代码可以看出，它从__fbBatchedBridge（也即是MessageQueue）中将几个函数转成了OC对象，这个几个方法将在后面在需要的时候调用。这几个方法都定义在MessageQueue.js中。MessageQueue这个JS类很重要，代码也比较多，这里就不贴了。callFunctionReturnFlushedQueue这个函数主要就是执行传入JS中的模块和方法，并把JS中的queue返回。这个queue存储了JS要调用的OC模块的类和方法和相关参数。OC中获取到这个queue后，就会解析这个queue中的内容，得到相关模块的配置和参数，并进行动态调用。m_flushedQueueJS这个对象对应的JS函数是flushedQueue，这个函数就是将queue返回，然后清空。在bindBridge()执行完之后，立马执行了callNativeModules(m_flushedQueueJS->callAsFunction({}))。这里就是将js中的queue拿过来进行调用。

为了更好的理解OC与JS是如何相互调用的，还是要先说说是怎么生成模块配置的。JS如何调用到OC。

### JS调用OC
当JS要加载某个module的之后，会调用到JSCExecutor中的getNativeModule方法，然后它会找该module，如果第一次加载该module，就会去创建该module的配置，然后存起来，下次再取时就直接返回已将建好的。下面是涉及的方法调用。

* JSCExecutor::getNativeModule()
   * JSCNativeModules::getModule()
   * JSCNativeModules::createModule()
       * ModuleRegistry::getConfig()
           * RCTNativeModule::getMethods()
              * [RCTModuleData methods]


这里看一下关键方法JSCNativeModules::createModule方法

````
folly::Optional<Object> JSCNativeModules::createModule(const std::string& name, JSContextRef context) {
  ReactMarker::logTaggedMarker(ReactMarker::NATIVE_MODULE_SETUP_START, name.c_str());

  if (!m_genNativeModuleJS) {
    auto global = Object::getGlobalObject(context);
    m_genNativeModuleJS = global.getProperty("__fbGenNativeModule").asObject();
    m_genNativeModuleJS->makeProtected();
  }

  auto result = m_moduleRegistry->getConfig(name);
  if (!result.hasValue()) {
    return nullptr;
  }

  Value moduleInfo = m_genNativeModuleJS->callAsFunction({
    Value::fromDynamic(context, result->config),
    Value::makeNumber(context, result->index)
  });
  CHECK(!moduleInfo.isNull()) << "Module returned from genNativeModule is null";

  folly::Optional<Object> module(moduleInfo.asObject().getProperty("module").asObject());

  ReactMarker::logTaggedMarker(ReactMarker::NATIVE_MODULE_SETUP_STOP, name.c_str());

  return module;
}

````

这可以看到取了JS中global对象的__fbGenNativeModule属性，存到了m_genNativeModuleJS。而__fbGenNativeModule定义在NativeModules.js中，它对应的是genModule函数。OC中将调用这个函数，把OC的Module生成JS的module，返回给JS，并最终OC中也会把它存在m_objects（map）中。genModule函数需要一个config参数，这个config主要是由m_moduleRegistry->getConfig生成。这个config里只要包含了像JS暴露的constants,methods。我们这里主要看看它是怎么收集到我们要暴露的方法的。

关键代码在[RCTModuleData methods]中

````
- (NSArray<id<RCTBridgeMethod>> *)methods
{
  if (!_methods) {
    NSMutableArray<id<RCTBridgeMethod>> *moduleMethods = [NSMutableArray new];

    if ([_moduleClass instancesRespondToSelector:@selector(methodsToExport)]) {
      [moduleMethods addObjectsFromArray:[self.instance methodsToExport]];
    }

    unsigned int methodCount;
    Class cls = _moduleClass;
    while (cls && cls != [NSObject class] && cls != [NSProxy class]) {
      Method *methods = class_copyMethodList(object_getClass(cls), &methodCount);//注意这里取的是类方法

      for (unsigned int i = 0; i < methodCount; i++) {
        Method method = methods[i];
        SEL selector = method_getName(method);
        //主要这个前缀__rct_export__
        if ([NSStringFromSelector(selector) hasPrefix:@"__rct_export__"]) {
          IMP imp = method_getImplementation(method);
          auto exportedMethod = ((const RCTMethodInfo *(*)(id, SEL))imp)(_moduleClass, selector);
          id<RCTBridgeMethod> moduleMethod = [[RCTModuleMethod alloc] initWithExportedMethod:exportedMethod
                                                                                 moduleClass:_moduleClass];
          [moduleMethods addObject:moduleMethod];
        }
      }

      free(methods);
      cls = class_getSuperclass(cls);
    }

    _methods = [moduleMethods copy];
  }
  return _methods;
}

````

这里代码主要意思是先拿到所有类方法（或者叫静态方法），然后如果是特定前缀__rct_export__的方法，则是我们之前通过宏RCT_REMAP_METHOD（RCT_EXPORT_METHOD）定义生成的方法，然后获取这个方法的实现，并执行得到结果RCTMethodInfo。这个RCTMethodInfo里包含真正我们要暴露给JS调用的方法，用它来创建一个RCTModuleMethod。将所有需要的方法都生成之后，就保存了起来。这里RN的开发者很巧妙生成了一个静态方法，通过这个方法将真正需要的方法信息生成。为什么将这个特定方法定义为静态方法呢，我估计是静态方法比较少，查找的时候比查找实例方法快。

当配置信息都生成好之后，调用genModule方法生成JS的module。这里要特别说明一下genModule这个方法会调用到genMethod函数，genMethod函数会生成另外一个函数，而生成函数里，将module的方法调用放到了MessageQueue的队列中，等待OC被调用。这里特别说明一下在JS的代码中，传递的参数都是moduleId, methodId是数字类型，moduleId它代表的是该module在OC的modules数组中的下标，methodId代表的是该method在methodNames数组里的下标。相关函数都是挺复杂的，理解起来有点绕，可以多看看genModule和genMethod函数的实现。

所以，假设我们已有一个注册模块AudioRecord和暴露方法record.

````
 let AudioRecorder = NativeModules.AudioRecorder; 
 AudioRecorder.record();
````

两句代码会发生：去OC中getNativeModule(AudioRecorder)寻找JS的module，现在m_objects字段中查找，如果没有找到则会走创建流程，先到m_moduleRegistry生成改类的配置信息，然后将配置信息传给JS的genModule函数生成名为AudioRecorder的js对象，调用genMethod生成名为record对应的JS函数，赋值给AudioRecorder对象,并保存在OC的一个字典（m_objects）中和返回这个JS对象。在JS中拿到这个对象后调用record函数，在record函数里，BatchedBridge.enqueueNativeCall就会被调用，moduleID，moduleID等信息就传入MessageQueue。我们看看enqueueNativeCall函数实现。

````
enqueueNativeCall(
    moduleID: number,
    methodID: number,
    params: Array<any>,
    onFail: ?Function,
    onSucc: ?Function,
  ) {
    if (onFail || onSucc) {
		......
      // Encode callIDs into pairs of callback identifiers by shifting left and using the rightmost bit
      // to indicate fail (0) or success (1)
      onFail && params.push(this._callID << 1);
      onSucc && params.push((this._callID << 1) | 1);
      this._successCallbacks[this._callID] = onSucc;
      this._failureCallbacks[this._callID] = onFail;
    }

	......
    this._callID++;

    this._queue[MODULE_IDS].push(moduleID);
    this._queue[METHOD_IDS].push(methodID);

	......
    this._queue[PARAMS].push(params);

    const now = new Date().getTime();
    if (
      global.nativeFlushQueueImmediate &&
      (now - this._lastFlush >= MIN_TIME_BETWEEN_FLUSHES_MS ||
        this._inCall === 0)
    ) {
      var queue = this._queue;
      this._queue = [[], [], [], this._callID];
      this._lastFlush = now;
      global.nativeFlushQueueImmediate(queue);
    }
    Systrace.counterEvent('pending_js_to_native_queue', this._queue[0].length);
	//
    if (this.__spy) {
      this.__spyNativeCall(moduleID, methodID, params, {
        failCbId: onFail ? params[params.length - 2] : -1,
        successCbId: onSucc ? params[params.length - 1] : -1,
      });
    }
  }
````
JS所有异步调用OC方法都会走到这里（同步方法走的是callSyncHook），我们可以看到先把回调存到数组里，然后把要调用的模块类，方法和参数都存在了queue中，当然还有回调函数对应的_callID（这里很巧妙地用一个数字代表了两个回调函数）。这里还有个判断，如果距离上次OC主动来调用JS超过了5毫秒，就会主动调用OC一开始就注入的nativeFlushQueueImmediate回调函数。在nativeFlushQueueImmediate这个方法里，OC会把传过来的queue里面需要调用的方法全部调用。

* JSCExecutor::nativeFlushQueueImmediate
* JSCExecutor::flushQueueImmediate
   * JsToNativeBridge::callNativeModules
     * ModuleRegistry::callNativeMethod
         * RCTNativeModule::invoke
             * RCTNativeModule::invokeInne
                 * [RCTModuleMethod invokeWithBridge:module:arguments:]

这里调用链条也很长，我们知道最终会到RCTModuleMethod类中，它是通过NSInvocation来实现动态调用的。

没有超过5毫秒的话，JS不主动调用OC的module，那OC到底什么时候会主动调用JS？

### OC调用JS
在RCTCxxBridge中的start方法里，执行完JS源码后，会发送一个通知。在RCTRootView中监听了这个通知，执行了runApplication方法，这个方法里主动调用了JS。

````
- (void)runApplication:(RCTBridge *)bridge
{
  NSString *moduleName = _moduleName ?: @"";
  NSDictionary *appParameters = @{
    @"rootTag": _contentView.reactTag,
    @"initialProps": _appProperties ?: @{},
  };

  RCTLogInfo(@"Running application %@ (%@)", moduleName, appParameters);
  [bridge enqueueJSCall:@"AppRegistry"
                 method:@"runApplication"
                   args:@[moduleName, appParameters]
             completion:NULL];
}

````
这里是JS的程序入口，之后会创建视图之类的。我们看看OC是怎么调用JS的。

* RCTBridge:: enqueueJSCall()
   * RCTCxxBridge:: enqueueJSCall()
      * Instance::callJSFunction()
         * NativeToJsBridge::callFunction()
             * JSCExecutor::callFunction()
           
我们看看JSCExecutor是如何调用JS的。

````
void JSCExecutor::callFunction(const std::string& moduleId, const std::string& methodId, const folly::dynamic& arguments) {
  SystraceSection s("JSCExecutor::callFunction");
  // This weird pattern is because Value is not default constructible.
  // The lambda is inlined, so there's no overhead.
  auto result = [&] {
    JSContextLock lock(m_context);
    try {
      if (!m_callFunctionReturnResultAndFlushedQueueJS) {
        bindBridge();
      }
      return m_callFunctionReturnFlushedQueueJS->callAsFunction({
        Value(m_context, String::createExpectingAscii(m_context, moduleId)),
        Value(m_context, String::createExpectingAscii(m_context, methodId)),
        Value::fromDynamic(m_context, std::move(arguments))
      });
    } catch (...) {
      std::throw_with_nested(
        std::runtime_error("Error calling " + moduleId + "." + methodId));
    }
  }();
  callNativeModules(std::move(result));
}
````
OC调用JS是通过bindBridge()里取到的JS定义的函数callFunctionReturnFlushedQueue来调用的。这函数需要的参数就是moduleName,methodName和method调用需要的参数。JS中拿到这些参数会在_lazyCallableModules里找到对应的JS module来进行对应的方法调用。_lazyCallableModules存的是需要给OC调用的JS module。AppRegistry模块在执行JS源码的之后就注册到这个_lazyCallableModules里。callFunctionReturnFlushedQueue函数不但执行了OC要调用的JS模块，最后还把queue传回到了OC，我们知道这个queue里存的都是JS要调用的OC模块信息，所以OC拿到这个queue之后就执行了调用callNativeModules。RN的机制就是OC在调用JS模块之后，也把JS中待调用的OC模块一起执行了。

还有什么情况，OC会主动调用JS呢。通过在xcode中搜索enqueueJSCall方法的调用，可以看到主要是在RCTEventDispatcher，RCTEventEmitter和RCTTiming。从此可以知道主要是跟事件触发和定时器相关。这也很合理，因为在移动操作系统是基于事件触发机制。大部分事件都是空转，只有事件触发后才会进行相关调用。

## 总结
大致总结一下RN APP的启动流程：在APPDelegate的启动方法中创建了一个RCTRootView，然后在RCTRootView中创建了RCTBridge，RCTBridge中创建了RCTCxxBridge，并调用了start方法。在start方法里先创建了一个专门用来运行JS代码的Thread，这个Thread绑定到了一个runloop。然后将所有注册的module生成RCTModuleData，并根据需要调用他们的初始化方法。然后一边在JS线程中执行Intance的初始化方法，一边异步进行js源码的加载。在Intance的初始化方法里最终会创建一个JSCExecutor,在JSCExecutor里创建了一个全局的js context,并注入了几个OC的回调方法，包括NativeModuleProxy对应的getNativeModule方法。当JS源码都加载完，其他初始化也完成，就会在JS context中执行JS源码，建立JS环境和JS的初始化。这时候一些被JS调用到的OC module就会初始化，会调用之前注入JS中的getNativeModule方法，这个方法会把module的配置信息生成，并交给JS的genModule方法来生成JS的对象，并在OC中存起来。JS源码执行完之后，OC会调用flush方法，然后会调用到bindBridge方法，将MessageQueue.js中定义的几个方法存在OC中以备调用。然后将MessageQueue存储的OC待调用方法进行调用。到这里，JS源码执行完毕，会发一个通知，RCTRootView会收到这个通知，然后就调用了js中的AppRegistry的runApplication方法，到这里，JS的入口就被调用，界面就会渲染出来。

总结起来，JS和OC的相互通信是经过JavaScriptCore机制来进行的。OC将要暴露给JS的放类和方法生成配置信息，然后交个JS生成JS对象和方法，但是JS中对应的对象和方法并不是直接调用OC的方法，而且先放入一个队列中，交由OC来调用。OC拿到对应的配置信息然后进行动态调用。OC调用JS的对象和方法，是直接调用，JS中存储了要暴露的对象module.


## 参考资料

[深入浅出 JavaScriptCore](http://www.cocoachina.com/ios/20170720/19958.html)

[JavaScriptCore全面解析 （上篇）](https://cloud.tencent.com/developer/article/1004875)

[JavaScriptCore全面解析 （下篇）
](https://cloud.tencent.com/developer/article/1004876)

[理解React-Native(0.46) 中native和js通信原理(iOS)](https://www.jianshu.com/p/931367388a8d?utm_campaign=maleskine&utm_content=note&utm_medium=seo_notes&utm_source=recommendation)

[React Native通信机制详解](http://blog.cnbang.net/tech/2698/)



如果你觉得这篇文章有用，请打赏小钱喝杯咖啡^_^
![打赏](https://raw.githubusercontent.com/szuwest/szuwest.github.io/master/images/2018-02-21%20133111.jpg)