Date: 2015-03-10
Title: Android开发中何时使用多进程？
Category: Android
Tags: 技术问答 github 进程

#问答-Android开发中何时使用多进程？

我在github上回答了一个问题：
`Android开发中何时使用多进程？`


````
1.要想知道如何使用多进程，先要知道Android里的多进程概念。一般情况下，一个应用程序就是一个进程，这个进程名称就是应用程序包名。我们知道进程是系统分配资源和调度的基本单位，所以每个进程都有自己独立的资源和内存空间，别的进程是不能任意访问其他进程的内存和资源的。那如何让自己的应用拥有多个进程？很简单，我们的四大组件在AndroidManifest文件中注册的时候，有个属性是android:process，这里可以指定组件的所处的进程。默认就是应用的主进程。指定为别的进程之后，系统在启动这个组件的时候，就先创建（如果还没创建的话）这个进程，然后再创建该组件。你可以重载Application类的onCreate方法，打印出它的进程名称，就可以清楚的看见了。再设置android:process属性时候，有个地方需要注意：如果是android:process=":deamon"，以:开头的名字，则表示这是一个应用程序的私有进程，否则它是一个全局进程。私有进程的进程名称是会在冒号前自动加上包名，而全局进程则不会。一般我们都是有私有进程，很少使用全局进程。他们的具体区别不知道有没有谁能补充一下。

2.使用多进程显而易见的好处就是分担主进程的内存压力。我们的应用越做越大，内存越来越多，将一些独立的组件放到不同的进程，它就不占用主进程的内存空间了。当然还有其他好处，有心人会发现Android后台进程里有很多应用是多个进程的，因为它们要常驻后台，特别是即时通讯或者社交应用，不过现在多进程已经被用烂了。典型用法是在启动一个不可见的轻量级私有进程，在后台收发消息，或者做一些耗时的事情，或者开机启动这个进程，然后做监听等。还有就是防止主进程被杀守护进程，守护进程和主进程之间相互监视，有一方被杀就重新启动它。应该还有还有其他好处，这里就不多说了。

3.坏处的话，多占用了系统的空间，大家都这么用的话系统内存很容易占满而导致卡顿。消耗用户的电量。应用程序架构会变复杂，应为要处理多进程之间的通信。这里又是另外一个问题了。

````

详见：[问答-Android开发中何时使用多进程？](https://github.com/android-cn/interview-questions/issues/7)