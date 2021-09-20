Title: 浅谈ReactNative技术的优劣
Date: 2018-02-06
Category: 技术
Tags: react-native

# 浅谈ReactNative技术的优劣
去年大概10月份开始使用ReactNative技术来开发项目，现在也已经有几个月时间了。也上线了三个ReactNative开发的项目。一直想谈谈使用ReactNative的开发感受。

* 使用react-native确实可以提高界面开发效率，因为两个平台，只需要编写一套代码。react-native在这方面做了很多工作，让界面开发跟web开发基本一致。web开发人员可以很方便的写react-native界面，不需要二次学习。而且在react-native里面的像素是逻辑像素，可以比较好的适配界面。
* react-native界面开发方式跟web开发是基本一致的，这跟native开发界面有很大不同。react-native界面开发是声明式开发，而且是继承于react框架，而纯native开发人员根本不了解这个框架，需要学习成本。它的View布局层次跟原生的布局层次不太一样。我自己也搞了挺久才搞清楚。
* react-native很适合那些纯http数据交互的应用。即那些数据和内容都是从服务器拉取，APP只是展示和消费的场景应用。这样不但界面可以react-native来写，HTTP请求也可以很方便的用ReactNative来写。
* react-native不适合用于那些需要复杂的通信方式的应用，也不适用于那些强多媒体资源的应用，例如要做相册，音视频播放的应用。因为react-native上不方便用socket的那些通信方法，需要自己封装。然后内存问题也不太好把控。对于相册，多媒体播放，ReactNative也没有封装。
* react-native对于web转native开发是很有用，对纯native开发转react-native开发不是很友好，主要是思维方式发生比较大的变化。
* react-native不能隔绝native开发的知识。要做APP开发，最终还是需懂得native开发的一些知识，react-native并不能完全屏蔽这些。一个纯web开发想要转react-native开发，还是需要懂native开发的人来帮助和指导，或者自己需要先学习native相关知识。

# react-native开发会是未来的方向吗
这个我也不敢说。不过ReactNative开发未来会代替掉一些native发，这是一定会发生的，或许已经正在发生。不过react-native版本最新的还是0.51版本，还有很多需要完善的地方。不过想在JavaScript貌似有大一统的趋势。我们公司现在都是用js来做PC客户端的。有个叫electron的框架貌似很流行，可以开发windows和Mac的桌面APP。只要会js，就可以写Mac客户端，这还真是我之前没有想过的。这样看，真的貌似js有大一统的趋势。

不过，不管怎样过，我觉得react-native还是无法完全替代原生开发，毕竟有很多东西需要用原生来开发。而且，react-native本身就是基于原生的，它只是做了一层转化而已。所以原生的开发能力不会削弱。

如果你觉得这篇文章有用，请打赏小钱喝杯咖啡^_^
![打赏](https://raw.githubusercontent.com/szuwest/szuwest.github.io/master/images/2018-02-21%20133111.jpg)