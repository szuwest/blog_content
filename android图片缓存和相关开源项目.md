Date: 2015-05-16
Title: Android图片缓存和相关开源项目
Category: Android
Tags: Android源码学习 图片缓存 ImageLoader Fresco

#Android图片缓存和相关开源项目

现在几乎所有大一点的Android项目都会用到图片缓存。而Android应用的内存占用大户就是图片，几乎所有内存问题都会涉及到图片问题，而已图片为主的应用也会涉及到性能问题。我觉得每一个资深的Android工程师都要对图片缓存技术有所了解，并且有自己想法。图片缓存是在面试别人时必问的问题。

图片缓存可以分为两点：内存缓存和磁盘缓存，当然还有必然要涉及的图片解码，图片下载，这里主要讲讲内存缓存和磁盘缓存，附带说一下图片解码和下载。

##内存缓存
由于一张图片基本上会被重用或者在多个界面显示，而图片的下载和解码都是比较耗时的动作，要给用户比较好的体验，将一张图片缓存在内存中就尤为必要。如果一个应用图片比较多，要将所有图片都缓存在内存中显然不现实，特别是比较低端的机器上，内存非常有限。所以这个内存缓存必然有所限制，那么问题来了，缓存池设置多大，并且满了之后怎么处理成了要点。我们知道Android系统中给每个应用设置了最大堆，超出了这个最大堆限制就会报OutOfMemory错误。所以一般就是根据这个最大堆来设置图片缓存池的大小，业内普遍做法是取最大堆内存的8分之一，这个经验值，我觉得可以应用情况做调整。而缓存池满了之后该又要加入新的图片，怎么对已在池中的图片移除，有很多方法。最常用的是LRU（least recently used）算法。我一般会问面试者这个算法是怎么实现的，看过代码的人或者算法学得比较好的人就能答出。还有用的比较多大算法是使用频率算法，移除最大图片算法等。然而我认为比较好的内存缓存技术还应结合弱引用来用。这里涉及到什么事强引用，软引用和弱引用也是我必考项。为什么还要结合弱引用来使用比较好呢？如果一个应用要显示的图片比较多，而强引用池又比较小，那么强应用池中的图片可能很快被移除，但是这时这张图片又还在某个界面中显示着，然后在一个新的界面中又要显示同一张图片，这时强引用池中已被移除，它就会重新去磁盘或网络中加载。而有弱引用池的话就可以避免这个问题。

内存缓存还有一个要注意的问题就是图片的解码和重用。例如说一张超大图片的话解码是不能直接解码到内存中的，还有可能要根据图片最终在界面上显示的大小来进行解码，还有就是针对不需要alpha值的图片采用RGB_565来解码，对一张已解码的不再需要图片内存重复利用起来，不用重新开辟内存，这些就是非常细但是很重要的点。

##磁盘缓存
磁盘缓存主要是针对网络图片，因为网络下载是比较耗时，对于已经下载的图片没有必要再下载一次。磁盘缓存主要涉及图片保存和磁盘缓存空间大小问题。图片从网络下载下来时，以什么文件保存，还有要不要对图片质量进行压缩也是讲究的。普遍做法是将图片的URL进行一次hash，将这个hash字符串作为文件名保存起来。下次只要对某个图片URL计算出它的hash字符串，看看对应的文件存不存在，就知道图片是否已存在磁盘缓存里。由于用户的磁盘也不是无限的，对于已图片为为主的应用，是要考虑对这个磁盘缓存空间做限制的。这里同样也有LRU算法等。

##多线程问题
对于多图片的应用，特别是在ListView或GridView中显示图片的应用来说，多线程加载图片是必然涉及的。因为图片不能在主线程中解码，否者在滑动列表的会被卡死。解码图片和下载图片都是很耗时的动作，必须放在子线程中进行。如何做一个滑动起来很流畅的图片列表，也是我必问的问题。这里可以涉及很多知识就不讲了。但是在图片加载环节，必然要考虑采用一个线程池来加载和解码图片。


----


````
图片的加载和缓存是一个很复杂的问题，如果这些都要自己写，要考虑东西很多，要做得好相当不容易。想起在2012年的时候，那个时候开源项目还没有现在那么多，我们就是自己写的。但是现在基本上不自己写了，因为有很多相关的优秀的开源项目可以采用。我只要选一个开源框架，加一些配置，或者根据自己项目的需要进行改写或者改造就行了。这里最受欢迎的开源项目应该是Universal-Image-Loader了，其他用得比较多的还有glide, IoUtils,Volley等，还有Facebook最近开源的Fresco。
````

##Universal-Image-Loader
UIL[github地址](https://github.com/nostra13/Android-Universal-Image-Loader)是我正在用的图片缓存开源项目。这是一个很优秀的项目，代码结构很好，配置性非常高，非常灵活，使用也非常简单。这也许就是它流行的原因。来看看它的经典结构：


![框架图](https://github.com/nostra13/Android-Universal-Image-Loader/raw/master/wiki/UIL_Flow.png)

有人将这种结构称为三级缓存（内存缓存，磁盘缓存，网络缓存？），UIL的结构很清晰，也很经典。我觉得其他图片缓存框架也是大同小异。我已在至少两个项目用这个框架了，不过我都有修改。第一项目中因为我们的图片下载是要经过安全校验的，所以我在UIL的download engine里加入了一个回调，通过回调获取校验的参数，然后设置到cookie中，ImageLoader不用关心校验方法。现在这个项目也在用UIL，由于我们项目的特殊性，对UIL做了更多改动。不过UIL框架很灵活，很好改动。这就是开源项目好处，我们站在巨人们的肩膀上。

虽然好处多多，但是我发现UIL也有不足之处。

* 框架太过灵活，相对的性能降低了。
* 不支持gif图片解码

第一个算不上太大问题，但是对于追求高效率的应用来说，特别是GridView中，用UIL会有性能影响。因为UIL的每次调用都创建很多对象，有些对象因为最求框架灵活引入的，实际是可以减少的。第二个是硬伤，不过现在支持框很少。貌似只有Facebook新开源的Fresco。

##Fresco
Fresco是Facebook刚开源不就的项目，我看了它的介绍非常兴奋。首先它解决UIL不支持gif图片解码的问题。然后它对图片进行重用，还有它的解码是在native中的，内存效率非常高。我最近非常想试试这个新框架。不过实在太忙没有时间。我打算找个时间研究一下它，并用它替换掉UIL。

先说到这吧，我也翻译这个项目官方介绍的那篇博客，再说。