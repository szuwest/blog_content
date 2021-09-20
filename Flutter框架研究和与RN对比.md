Date: 2018-03-16
Title: Flutter框架研究和与RN对比
Category: 技术
Tags: ReactNative Flutter

## Flutter是什么
现在技术更新迭代真的很快，每隔几年就会出现一些新的技术。当然，Flutter出现有有一点时间了，只不过还未真是发布，但是已经有一些人在使用了。这篇文章主要内容来自我在公司内部的一次分享会，所以大部分内容都是提炼。

* Flutter 是由 Google 的工程师团队打造的，用于创建高性能、跨平台的移动应用的框架。
* Flutter 针对当下以及未来的移动设备进行优化，专注于 Android and iOS 低延迟的输入和高帧率
* Flutter的设计跟react-native很像，但是比RN进了一步
* Flutter的开发语言是Dart

## 移动端跨平台开发技术演进
现在主流的移动开发平台是Android和iOS，之前还有过windows phone，每个平台的开发技术都不太一样。大家都是针对每个平台开发应用。自然有人就会觉得这样效率低下，想进行跨平台开发。从最开始的Hybrid混合开发技术，到RN的桥接技术，到现在新兴的Flutter技术，跨平台开发技术一直在演进。

以往最早的Hybrid开发，主要依赖于WebView。但是WebView是一个很重的控件，很容易产生内存问题，而且复杂的UI在WebView上显示的性能不好。react-native技术抛开了WebView，利用JavaScriptCore来做桥接，将js调用转为native调用，只牺牲了小部分性能获取的跨平台开发，这是一大进步。所以现在react-native很流行的原因。

![react-native原理图](https://res.infoq.com/articles/why-is-flutter-revolutionary/zh/resources/2.png)

上图react-native框架原理

Flutter实现跨平台采用了更为彻底的方案。它既没有采用WebView也没有采用JavaScriptCore，而是自己实现了一台UI框架，然后直接系统更底层渲染系统上画UI。所以它采用的开发语言不是JS，而Dart。据称Dart语言可以编译成原生代码，直接跟原生通信。

![Flutter原理图](https://res.infoq.com/articles/why-is-flutter-revolutionary/zh/resources/4.png)

上图是Flutter框架原理图

Flutter将UI组件和渲染器从平台移动到应用程序中，这使得它们可以自定义和可扩展。Flutter唯一要求系统提供的是canvas，以便定制的UI组件可以出现在设备的屏幕上，以及访问事件（触摸，定时器等）和服务（位置、相机等）。这是Flutter可以做到跨平台而且高效的关键。另外Flutter学习了RN的UI编程方式，引入了状态机，更新UI时只更新最小改变区域。

系统的UI框架可以取代，但是系统提供的一些服务是无法取代的。Flutter在跟系统service通信方式，采用的是一种类似插件式的方式，或者有点像远程过程调用RPC方式。这种方式据说也要比RN的桥接方式高效。

## Flutter与RN异同
简单总结一下Flutter与RN的异同。

* 都实现了移动开发跨平台
* 界面的编写都很类型，采用响应式视图，维护了一个状态机，只更新改变的最小区域界面
* 都支持热重载hot reload，开发调试非常方便
* 调用系统的service仍然需要封装接口，仍然还是需要懂得native开发
* RN采用JS语言开发，基于React，受众更多。Dart语言受众小
* Flutter的UI框架性能貌似更高一些，但是直接丢弃了原生UI框架。而RN还是可以自己利用原生框架，两个各有好处。Flutter的兼容性高，RN可以利用原生已有的优秀UI
* Flutter的第三方库还很少，RN发展的早，虽然也还不完善，但是比Flutter好
* RN的界面布局更像网页布局，而Flutter的布局更像native布局
* Flutter在跨平台这方面做得更彻底一些

## 我试用Flutter的感受
我自己按照官方教程写了一个简单的无限滚动的ListView，感觉Flutter的界面布局是完全自己搞了一套，跟RN的web风格不同，跟原生的也不太相同，这里需要一点学习成本。我还运行了官方自动的sample例子，倒是还不错，把Android的material风格控件都移植过来了。另外我发现Flutter的在调试模式和Release模式下性能差别很大。如果你调试时发现性能差，就最好试试release模式。

Flutter采用Dart语言开发不知道是好是坏。现在可是JS的天下，这是RN的优势之一，web开发人员很容易开发RN的界面，但是Flutter的一切都要从新学习。Dart语言我也还没习惯。

Flutter现在还处理Bata阶段，第三方库还很少。我用过一个AudioPlayer的第三方库，竟然出问题。所以很多东西都需要自己开发和封装。RN经过了一段时间积累，好用的库还有不少。现在用Flutter来开发APP，感觉还是有点太早。

Flutter官方吹的很大，说它是革命性的，也有一定道理。但是我觉得RN对于熟悉web开发的人来说，是更好的选择。但是对于纯native开发的移动开发人员，直接学习Flutter会更好，Flutter也比较适合本来就是做native开发的人。


----------------
如果你觉得这篇文章有用，请打赏小钱喝杯咖啡^_^
![打赏](https://raw.githubusercontent.com/szuwest/szuwest.github.io/master/images/2018-02-21%20133111.jpg)