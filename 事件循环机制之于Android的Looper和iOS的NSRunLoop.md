Date: 2016-12-04
Title: 事件循环机制之于Android的Looper和iOS的NSRunLoop
Category: 技术
Tags: Android iOS Looper NSRunLoop

##事件循环机制之于Android的Looper和iOS的NSRunLoop

Android和iOS同为手机操作系统，有很多相同之处。有很多设计几乎是一样的，统一种设计模式，不同的实现。例如事件模型，它大体逻辑是这样的：

````
function loop() {
    initialize();
    do {
        var message = get_next_message();
        process_message(message);
    } while (message != quit);
}
````
对于这样的一种模式, Android系统和iOS系统是一样的，只不过是不同的平台不同的实现方式。应该说很多平台或者框架都是采用这一种模式。Android系统对应的就是Looper机制，iOS对应的就是NSRunLoop机制。你看名字都是loop，这是它的核心。

对于移动开发者来说，理解平台是怎么实现这个事件循环机制，或者叫消息机制，是很重要的。因为在你的开发工作当中，无时无刻不在用着这种机制。不理解它的机制，充其量也只能算个初级程序员，无法深入理解系统。如果我们能了解不同平台对这一机制的实现，可以加深理解，可以提高自己。我既做过Android开发，也研究过它Looper机制。现在做iOS开发都有一段时间了，最近开研究它的NSRunLoop机制。我发现他们整体的设计思想确实一致的，但是不同的平台却也有很多差异性。

##共同的设计点
首先loop跟线程是一一对应的，这是基础。然后iOS或者Android都是一个应用一个进程（并非绝对），然后进程里有一个主线程。然后这个主线会绑定一个loop。这工作是有系统来完成的。这个loop一直在接收者系统的一些事件，或者用户产生的事件。如果没有事件或者处理完了事件，它就在等待中，等待别事件来临唤醒它处理。不管Android还是iOS，它们的更新界面操作必须在主线程中进行。所以我们在写代码的时候经常会出现将消息抛到主线程中来执行。

对于子线程，它默认是没有loop的，当它执行完它的任务后，线程就结束了。这也符合我们大多数的场景。如果我们要让子线程也绑定一个loop呢？或者说我们什么情况下需要子线程运行一个loop。我们大部分情况其实都不需要额外创建一个loop，因为没有用到，所以很多人就不去了解。这也是不行的。对于有周期性人任务的情况，用loop是很好的选择。例如我们需要一个后台线程一直运行，然后我们会定期或不定期给它发送一些任务，用loop来实现时比较好的。

线程默认没有loop，而且loop不能程序员自己手动调用构造函数创建，需要调用系统获取loop的方法。系统会把创建的loop跟当前的线程绑定在一起，存在一个全局的key-value的字典里，下次再获取当前线程的loop，就直接取出来用。

获取到loop后，我们让它进入事件循环，即执行类似上面的loop()方法，这是一个死循环，除非外部停止掉这个循环。这个循环里面有一个重要的点，就是没有消息的时候，它是进入睡眠等待状态，有消息来时候，会把它唤醒并执行消息处理。这个消息循环机制Android和iOS的实现差别还是挺大的。下面我们说一下，如何建立一个loop循环，然后如何给它发消息和一些需要注意的东西
##iOS的NSRunLoop
首先来看一下如何在iOS中创建一个loop并让它跑起来

````
	//创建并启动Thread
	self.testThread = [[NSThread alloc] initWithTarget:self selector:@selector(onTestRunLoop:) object:nil];
    [self.testThread setName:@"TestRunLoop"];
    [self.testThread start];
    
    //Thread的入口方法
- (void)onTestRunLoop:(id)obj {
    NSLog(@"--%@--", [NSThread currentThread]);
    [[NSRunLoop currentRunLoop] addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
    NSLog(@"***%@***",[NSRunLoop currentRunLoop]);
    [[NSRunLoop currentRunLoop] run];
}
````
####mode和source
上面最主要的代码是**[[NSRunLoop currentRunLoop] addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode]** 和 **[[NSRunLoop currentRunLoop] run]**，currentRunLoop这个方法会检查当前线程有没有loop，没有的话，就会自动创建。NSMachPort是一个事件源source，NSDefaultRunLoopMode代表一个模式Mode。当把source和Mode都设置之后，开始调用run方法，进入循环。

这里mode和source是必须的，否则NSRunLoop跑不起来。NSRunLoop必须运行在某一种mode中，mode可以切换，切换之前就会把之前的mode停掉。iOS系统定义了好几种mode，这个是它跟Android一个重要的不同点，可以说这个设计让它的效率比较高。例如主线程中，列表滑动的时候，NSRunLoop切换到TrackingRunLoopMode下，之前运行mode的source就停掉。这样里列表滑动起来就会很流畅。至于source，Port只是它的一种，属于Input Source，还有NSTimer也是source，属于Timer Source.
####observer和autoreleasepool
除了source，还有一个observer机制，这是iOS的特色，Android的loop机制中没有这些。iOS系统在loop循环中定义了一些重要的事件，然后你可以监听这些相关事件。例如即将进入Loop，即将处理 Source，即将进入休眠，刚从休眠中唤醒 这些事件都会通知到相关的观察者。这个机制我们开发者很少用到，但是在iOS系统中却很重要。其中一个就是AutoReleasePool机制。

我相信不少人在面试时会被问到NSRunLoop跟AutoReleasePool有什么关系，AutoReleasePool什么时候释放池里面的对象。我们第一反应很容易就认为是@autoreleasepool{}方法块执行之后。回答这个问题之前我们先看看别的。我们知道在iOS的main.m文件里一般是这样的：

````
int main(int argc, char * argv[]) {
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
````

AutoReleasePool包含了UIApplicationMain()方法，UIApplicationMain()方法里建立了main runloop,然后一直在那里循环处理消息或者等待，如果你再该方法之后再加一些语句，是没有办法执行的。按照之前的说法，那岂不是AutoReleasePool一直不会释放它里面的对象？显然这是错的。但是要理解AutoReleasePool什么时候释放它里面的对象，还是需要先了解一下AutoReleasePool的实现机制。AutoReleasePool实际上是有AutoReleasePage来实现的，而AutoReleasePage有点类似于堆栈，有一个push和pop操作。苹果在主线程的runloop监听了几个事件，分别是即将进入Loop，准备进入休眠和即将退出Loop，在即将进入Loop时进行push一个哨兵对象（或者叫边界对象），在即将休眠时是pop哨兵对象操作，然后再次push操作，在即将退出Loop时pop操作。在push哨兵对象之后，程序运行会push很对autorelease对象到AutoReleasePage中，pop操作就把这些对象释放，一直找到哨兵对象也把它pop掉为止。

所以每次runLoop进入休眠前AutoReleasePool就释放一次。AutoReleasePool堆栈式结构让它可以嵌套，互不影响。更新详细的AutoReleasePool资料可以查看最后的参考资料

####给RunLoop发消息
建立了一个runloop，给它发消息做任务才是我们的目的所在。怎么在别的线程给我们建立的runloop发消息呢？最直接的方法是通过performSelector:onThread:withObject:waitUntilDone:方法.这个是NSObject的方法。这个方式需要持有Thread对象。当然也可以同RunLoop添加Port或者Timer的方式，然后持有Port或者Timer，这个种方式不是那么直接。如果需要跨进程发送消息，则需要Port，但除了系统间的进程，跨进程通信并不常用。

现实的编程工作中，更常用的是往主线程的RunLoop发送消息。iOS系统提供了很多方式很方便我们操作。例如NSObject中就有performSelectorOnMainThread:withObject:waitUntilDone:方法，就是往MainRunLoop里发消息。还有dispatch_(a)syn(dispatch_get_main_queue(),block) GCD方法也是。同时还有很方便的获取mainThread和mainRunLoop的方法。

####退出机制
runloop什么时候退出呢？从API文档来看，苹果貌似没有提供主动停止runloop的接口，但是在run的时候可以指定超时时间（runUntilDate: 和 runMode:beforeDate）。从底层代码来看，外部是可以主动调用停止的，然后还有就是没有source/timer/observer的话，runloop也会退出。

####总结
NSRunLoop总体来说不简单，并且平时不太会用到。很多东西还是不太好理解。mode如何切换的？在一个线程中通过performSelector:onThread:withObject:waitUntilDone:向另外一个线程的runloop发送消息，是怎么发送过去的？还有source里又分很多种，observer在系统中还有那些应用等，还有很多东西可以挖掘。NSRunLoop 是基于 CFRunLoopRef 的封装，而CFRunLoopRef是开源的，最好的研究方式还是去阅读[源码](https://opensource.apple.com/source/CF/CF-855.17/CFRunLoop.c)

####参考资料
>[深入理解RunLoop](http://blog.ibireme.com/2015/05/18/runloop/)

>[[RunLoop已入门？不来应用一下？](http://www.jianshu.com/p/c0a550d2ac97)

> [iOS runloop 学习笔记(二)](http://www.jianshu.com/p/929d855c5a5a)

>[黑幕背后的Autorelease](http://blog.sunnyxx.com/2014/10/15/behind-autorelease/)

##Android的Looper

跟iOS不同，Android的源码是开发的，你可以很方便的看到Looper的源码。实际上大家研究Looper机制都是阅读源码的。整体来讲，我觉得Android的事件循环机制比较简洁明了。我们看一下Android如何创建一个Looper并让它跑起来

````
class LooperThread extends Thread {
	public Handler mHandler;

   public void run() {
       Looper.prepare();

       mHandler = new Handler() {
         	public void handleMessage(Message msg) {
              // process incoming messages here
           }
       };
       Looper.loop();
    }
 }
       
````
最主要的代码是**Looper.prepare()** 和 **Looper.loop()**，前一句的意思是创建一个Looper,并且将这个Looper和Thread存储到一个静态ThreadLocal变量中。注意如果当前线程已有Looper它会抛异常。后一句代码的作用就是进入一个死循环，不断的从MessageQueue中取出Message，然后拿到Handler，执行它的dispatchMessage方法。

####Looper，MessageQueue，Handler，Message
消息循环机制里最重要的是Looper和MessageQueue。看源代码就可以知道，MessageQueue是作为Looper的一个成员变量而存在，当Looper实例化的时候，它也被初始化，并且当前的Thread对象也当做成员变量存起来。Looper.loop()方法很简单，核心代码就那么几句:

````
for (;;) {
	Message msg = queue.next(); // might block
	if (msg == null) {
    // No message indicates that the message queue is quitting.
       return;
   }
   msg.target.dispatchMessage(msg);
   ......
   msg.recycleUnchecked();
}
````
所以Looper做的工作就是不断从MessageQueue总取出消息，然后调用Message的target，即Handler来处理（dispatchMessage）消息。取消息的时候，即queue.next()方法，如果当前queue里没有消息，它会在这里休眠等待，知道有消息过来。如果消息为空，就跳出循环返回了。重点在于MessageQueue，看它如何将消息发过来，又如何实现取消息的。

####用Handler给Looper发消息
下面是通过mHandler给Looper发消息，mHandler是上面再LooperThread的run方法里创建的，并且实现了Handler的handleMessage方法。注意mHandler是通过没有参数的构造函数创建的，这样的话它会获取当前线程的Looper和Looper的MessageQueue当做成员变量保存起来。

````
//发送消息
mHandler.sendMessage(Message.obtain());
mHandler.post(new Runnable() {
    @Override
	public void run() {
	//TODO
 	}
 });
````
可以给Handler发送Message或post一个Runable，我们也可以通过sendMessageDelayed方法发送延时消息，即指定消息多久后才执行。所有通过sendMessage或者它的重载方法发送的Message，最终都被传到上面的handleMessage中来处理。这样说来，Handler既是消息的发送者，也是消息的处理者，这样看起来好奇怪没有必要的样子。实际上不奇怪，你现在发出的消息，要下一个事件循环才会处理到，所以它的执行异步的。另外消息的执行是在Handler所绑定的那个Looper所在的线程，我们经常做的就是在子线程中调用在主线程Handler发送消息，这样就做到的线程切换。

####Message
很早之前我想过一个问题，就是一个线程中我们可以创建多个Handler，每个Handler都有的它的handleMessage方法，我们怎么保证某个handler发消息不会发到别的handler的handleMessage中呢。看了源代码后发现这个问题很简单：每个Message发送出去前，会将发送它的那个Handler保存到一个叫target参数中，Looper中就使用这个target来处理消息。但是这个target参数向开发者隐藏的，我们不看源代码不知道它的存在。Message类中还有好几个这样的参数。例如我们通过Handler的post的Runable最终是封装在Message的callback参数中，还有一个when参数表明每个消息的执行时间，还有一个Bundle类型的data数据，还有一个next参数。没错，Message其实是一个单链表的节点数据结构，这一点的使用体现MessageQueue中，MessageQueue本身没有再实现队列，是借助Message是实现的。还有Message中还有一个消息缓存池，这就是大家都推崇使用Message.obtain()来获取消息的原因。还有Message中还有一个Messenger类型的参数replayTo，这个参数用来跨进程通信的,在Handler中实现了这个机制。所以Message和Handler还真是有不少东西，要了解更多请看源代码或者后面的参考资料。

####MessageQueue和垃圾回收
MessageQueue算是这个机制的核心，因为消息都是通过它的enqueueMessage传进来，通过它的next方法取消息或者block等待。他还包含了一些native方法，这些方法就是对应了c++层的Looper,MessageQueue，这又是一个很深的知识点了。

enqueueMessage方法不算复杂，主要是对Message参数检查和加入队列，最终调用nativeWake方法，通知底层列表有变化。而next方法，则比较复杂，它有一个for死循环，并调用了nativePollOnce方法进行阻塞等待。当被唤醒之后，它从队列里找消息，找到了需要处理的消息就返回，没有找到的话，处理IdleHandler，处理完进行下一次循环。这里的IdleHandler很有意思，我开始不知道是啥，后来看到资料说我们的主线程ActivityThread就实现了IdleHandler，在里面做垃圾回收工作。这恰恰是跟iOS的RunLoop里面的Observer机制一样。都是在进行两次休眠之前进行一次垃圾回收，真是异曲同工啊！

####总结
Android的Looper机制说简单简单，说复杂复杂，但是好处是我们可以看到所有源码。我这里只讨论了上层我们常用的接口和机制，实际上有很多东西还值得我们研究。例如MessageQueue还可以放Barrier, MessageQueue相关的native方法涉及到的C++层的Looper机制，Handler和Messager是如何实现跨进程通信的，这些都是很高级的知识，以后有机会研究研究。

####参考资料
>[聊一聊Android的消息机制](https://my.oschina.net/youranhongcha/blog/492591)
>
>[Android消息处理机制(Handler、Looper、MessageQueue与Message)](http://www.cnblogs.com/angeldevil/p/3340644.html)
>
>[ Android 基于Message的进程间通信 Messenger完全解析](http://blog.csdn.net/lmj623565791/article/details/47017485)
>

##总结
iOS和Android同为当今最流行的移动操作系统，在很多系统设计方面都是非常像。或许说他们彼此借鉴，技术本来就是这样样子。消息循环机制本来就是一种设计模式，只是Android和iOS根据自己的平台做了一套实现。通过对比学习，自己收获很多，但是我也感觉自己还不够深入，现在只是学到点皮毛而已。继续努力。
