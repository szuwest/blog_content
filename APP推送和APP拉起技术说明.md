Date: 2020-08-04
Title: APP推送和APP换起技术方案探讨
Category: 技术
Tags: 推送 唤起 拉起

# APP推送和APP换起技术方案探讨

### 目标：
尽量将APP两端设计成统一的方案，减少服务器适配工作量，最好可扩展，能复用。

### 方案说明：
我们知道APP推送目的是将对用户有用的信息，主动推给APP，作为通知信息展示，用户通过点击通知进入APP相关页面。

这里只要涉及服务器端和APP端两个技术点：服务器推送什么样的信息过来，APP这边如何解析接收到的信息，打开相应的界面和解析数据。

服务器只有一个，所以尽量让服务器发送给两端的数据较为一致，这需要一开始就应该想好整个系统的设计方案。

安卓和iOS是两个完全不同的平台，相关技术栈也不一样。不过基本都要依赖厂商的通讯通道来个APP发通知：苹果有自己的APN通道，Google也有自己的通道，当然在中国，各个厂商有自己的通道。有很多第三方SDK都整合多家厂商的通道。我们使用了信鸽（腾讯云移动推送）的SDK。之所以选他家，主要是他们集成了主流的厂商，文档也还可以（以前他们有免费版）。

为了能拉起APP不同的界面，我们需要定义一套规则。iOS的方案比较固定，只能通过json参数，这是它系统框架决定的。而安卓就不太一样，不同的厂商支持不同的方式，APP存活状态和未启动状态也不太一样。不过有一个方案是所有情形都适用的：就是通过定义Activity的scheme方式。

scheme方式是一种URI，我们经常用的URL地址也是一种URI。URI在iOS或者安卓都经常使用来拉起页面。例如我们的APP主页可以定义URI为: wegene://com.wegene.app/main。这里wegene为schema，host为com.wegene.app，path为/main，还可以携带参数。wegene://com.wegene.app/main?index=1表示首页的第二个tab（index从0开始算）。

这样的话，服务器要推送信息的时候，只需设置path和相关参数就行了。iOS端和安卓端专门弄一个组装函数来组装path和参数：安卓端拼凑URI，设置为action；iOS端将path和参数组成json就行。

安卓端需要通过intentFilter设置好action，然后在activity中获取参数。这里需要注意的是如果APP是未启动的状态，那么这个activity将是第一个被启动的页面，越过了你的启动屏甚至首页，所以要考虑是否要登陆鉴权，必要数据是否已经获取到等问题。iOS端则需要从通知回调里拿到数据，解析出来path和参数信息。这里面要考虑冷启动的时候，根据path来跳转相关界面时，首页是否已经初始化完成。不然可能会出问题，例如要获取NavigationController来push操作，但是NavigationController还未初始化完成。

安卓端每个页面都要设置intentFilter和定义path，而且越过了主界面。当用户看完推送界面并关闭了那个页面，很多APP都会进而打开APP主页。安卓其实也可以统一先打开主页，然后再由主页打开相关页面。这里需要服务器通知配置URI为首页的URI，然后将要跳转的path作为参数附在URI后面。当然这里的path值需要URL编码，跟其他参数一样都要编码。然后在主页通过intent的data里解析出来path和其他参数，进而打开相关页面。

通过设置特殊scheme的方式来进行推送跳转有很多好处。例如iOS和安卓两端通用；不但支持推送拉起，也支持其他APP或者网页拉起（当然网页还有更好的方式）。如果path设置的跟网页的路由一直，那就更容易三端保持一致性了。

### APP唤起
我们经常希望自己的APP能被其他应用唤起，进而提高活跃度。APP唤起主要有三种情形：

* 由系统唤起，例如收到推送通知
* 有第三方APP唤起，例如有合作APP唤起
* 通过网页端直接唤起

通过推送唤起上面已经说了，通过第三方APP唤起其实也基本一样，主要通过预定义的scheme唤起。安卓端是跟推送完全一致的，iOS端推送是系统自动唤起的，第三方APP是通过scheme来唤起通信的。网页端其实也可以通过scheme方式来唤起，只是网页端多了些限制（会先弹出一个系统确认框）。

不过网页端有更好的实现方式：直接通过URL来唤起APP相应界面。安卓端这种技术叫APP Link，iOS端叫 Universal Link。其实安卓的APP Link应该是参考iOS的UniversalLink来做的，所以他们的技术基本一致，都要要服务器验证。这里不讲具体的实现细节了。

#### 微信唤起APP，hybrid APP，推送
这里特别说一下微信唤起APP。以前要在微信里拉起自己的APP，很困难，会因为他限制了scheme方式和UniversalLink（AppLink）方式。不过最近微信开放了他的一些能力。例如APP里可以直接拉起微信小程序不受数量限制，微信里可以拉起第三个APP了。这些都是通过微信SDK实现的。

这跟hybrid APP，推送有什么关系呢？其实没太大关系。但是如果你的APP是hybrid APP，有内嵌网页，经常要拦截URL跳转原生界面，也经常要推送消息跳转原生界面，就可以将他们都联系起来。

微信唤起APP可以传入一些参数，这个参数如果是当前网页URL就好了。然后APP里内嵌网页经常要拦截网页URL，然后推送信息里也可以专门定义一个URL参数来指定要跳转的页面。这里就可能专门建立一个类来处理URL的跳转了，因为他们的跳转逻辑是一致的。这样就可以做到最大的逻辑复用，减少bug。


----------------
如果你觉得这篇文章有用，请打赏小钱喝杯咖啡^_^
![打赏](https://raw.githubusercontent.com/szuwest/szuwest.github.io/master/images/2018-02-21%20133111.jpg)