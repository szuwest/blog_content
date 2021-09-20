Date: 2016-12-08
Title: iOS后台下载和断点续传
Category: iOS
Tags: NSURLSession ASIHTTPRequest 后台下载

##iOS后台下载和断点续传
最近在重新整理我们项目里的iOS的后台下载，因为原来方法（ASIHTTPRequest方式）无法做到后台一直下载，这个问题被我老婆吐槽了好几次。所以我重新整理一下，用NSURLSession来下载，达到了比较好的效果。现在总结自己的一些经验。

###背景
项目最开始我们是用了NSURLSession来做后台下载的。但是有两个严重的问题

* 有时候偶现的不能下载，就是一启动下载就失败。一旦出现这种情况，无法恢复，怎么样都无法下载，所有任务都一样
* 下载速度很慢，只有几十到几百KB/S，网络正常，Android端同一个文件下载速度飞快
* 程序杀掉之后无法重新继续下载

按道理这些问题都不应该出现，可是我们的应用就是出现了这些问题，而且难以调试，同事调试了很久都没有结果。最后我也没认真研究，但是我发现ASIHTTPRequest可以很好地解决上面的几个问题，然后我就在原来的基础上加了ASIHTTPRequest的下载方式。

ASIHTTPRequest方式的下载不错，速度很快，也能很好的断点续传，但是跟Android的还是慢了一点，不过我觉得这可能是系统不同的原因。ASIHTTPRequest还有一个比较致命的弱点，就是不能后台下载，这对于下载大文件来说是必须的。在iOS平台要做后台下载，最好的方式还是使用NSURLSession。

所以我决定好好研究一下NSURLSession，并且改用这种方式。

###NSURLSession后台下载
实际上NSURLSession的后台下载真的很强大，苹果真做了件很好的事。当你创建一个后台下载任务的时候，实际上你就把这个下载交给系统来接管了。所以即使你把APP杀掉，下载也不会停止，很牛逼。而且如果出现手机网络变化之类的，系统会在后台帮你重试，当网络又正常了它会继续下载。所有估计一旦你启动了后台下载任务，要么下载完，要么你手动取消，要么服务器那边出错，不然这个下载不会停止。

虽然NSURLSession那么强，但是要做一个体验比较好的下载器还需要注意很多地方。
>
* 必须是支持后台下载的NSURLSessionConfiguration
* 必须用这个 (NSURLSession *)sessionWithConfiguration:(NSURLSessionConfiguration *)configuration delegate:(nullable id <NSURLSessionDelegate>)delegate delegateQueue:(nullable NSOperationQueue *)queue来创建NSURLSession，并且delegate不能为空。NSURLSession最好全局唯一
* 必须实现delegate几个重要的方法
* 最好实现 APPDelegate中的- (void)application:(UIApplication *)application handleEventsForBackgroundURLSession:(NSString *)identifier completionHandler:(void (^)())completionHandler方法，并且把completionHandler保存起来适时调用
* 在手动暂停或者失败后，要把resumeData保存起来，最好是保存的本地，重新启动下载是需要这个来继续下载

还有需要注意的是 在 - (void)URLSession:(NSURLSession *)session
      downloadTask:(NSURLSessionDownloadTask *)downloadTask
didFinishDownloadingToURL:(NSURL *)location 这个回调方法里，必须location指向的临时文件移动到你的沙盒目录中，因为这个方法一旦返回后，就会去删除这个临时文件。

根据上面这些，我自己做了一个下载任务管理器，可以创建多个任务，并可以配置多个任务并行下载。当一个下载任务完成后便会启动下一个等待中的任务。这样的话，你就可以创建完下载任务后，就关闭程序，该干啥就干啥去。下载完了它会发本地通知。

不过这个下载器在公司的项目里用，牵涉比较多，还支持ASIHTTPRequest下载，所以暂时没法开源。不过有一个demo包含了核心的思想。

demo在这里 [BackgroundDownloadDemo](https://github.com/szuwest/BackgroundDownloadDemo)


###参考资料：

>1.[WWDC 2013 Session笔记 - iOS7中的多任务](https://onevcat.com/2013/08/ios7-background-multitask/)

>2.[基于iOS 10、realm封装的下载器（支持存储读取、断点续传、后台下载、杀死APP重启后的断点续传等功能）](http://www.jianshu.com/p/b4edfa0b71d8#)

>3.[iOS使用NSURLSession进行下载（包括后台下载，断点下载）](http://www.jianshu.com/p/1211cf99dfc3)