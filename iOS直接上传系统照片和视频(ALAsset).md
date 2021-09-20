Date: 2016-12-01
Title: iOS直接上传系统照片和视频（ALAsset）
Category: iOS
Tags: 上传 ALAsset NSInputStream

##iOS直接上传系统照片和视频（ALAsset）

我之前有一篇文章里讲过上传系统照片和视频的事（[iOS系统相册上传不得不说的那些事儿](http://szuwest.github.io/iosxi-tong-xiang-ce-shang-chuan-bu-de-bu-shuo-de-na-xie-shi-er.html)），需要将ALAsset从系统相册里导出到沙盒文件里，然后再将这个文件上传。这里需要无端端写一次文件的时间就不说了，最要命的是还要占据额外的磁盘空间。我们的用户一般要备份相片视频的时候，往往是手机空间不足的时候。这个时候存储空间很紧张，你备份还需要额外的空间。如果一个视频很大，例如2G，那么手机上需要有空闲的2G空间才能导出视频，才能备份。真是硬伤。

这个问题一直在我心头，卡了我很久，之前我有网上找解决方法，没有找到，很多都是说将ALAsset导出文件到沙盒的事。我自己也有想过要怎么解决。我想过从ALAsset中读取一段一段的NSData数据，然后分别将这些数据上传。这样也许是可以的，但貌似需要服务器那边能将这些包组合起来。另外客户端这些做起来也挺复杂。还有一种方法是我不使用AFN库，自己直接用NSURLConnection来写，跟服务器建立连接后不断的从ALAsset中读取数据写入跟服务器建立的连接。这个是参考Java的写法。可是我又不知从何动手。并且之前都很忙，没有时间想这么多。

最近我比较闲了，我想正好有时间来解决这个问题。这是我心中的一块石头，我要把它拿掉。

这次我直接在github上找，说不定能找到一些有用的代码。果真让我找到了 [FMAssetStream](https://github.com/formal-method/FMAssetStream)这个库,这个库的做法很简单，定义了一个子类来扩展**NSInputStream**，重载了一个最重要的方法 :

```
- (NSInteger)read:(uint8_t *)buffer maxLength:(NSUInteger)len {
    if(read >= size) {
        return 0;
    }

    NSUInteger bytesRead = [self.assetRepresentation getBytes:buffer fromOffset:read length:len error:nil];
    read += bytesRead;
    
    // Update stream status when it's consumed
    if(read >= size) {
        streamStatus = NSStreamStatusAtEnd;
    }
    
    if(self.progressDelegate) {
        [self.progressDelegate progressBytes:read totalBytes:size];
    }
    
    return bytesRead;
}
```
这个方法做的事情很简单，就是读取一定长度的数据到buffer里。而这个buffer里数据就是上传文件的时候一次读取并写入的数据。**我怎么就没有想到扩展NSInputStream类这种做法呢**，我心里大骂自己，这很自然的一种做法呀，理应就该这么做的。上传文件一般都是用流，文件流是系统提供的，但是我们自己也可以扩展它来处理特别的情况呀。而且我看过AFN库的源代码，它也是扩展了NSInputStream来做多部分数据上传。

知道了扩展NSInputStream来从ALAsset中读取数据，还有一个问题需要注意，就是断点续传。关于断点续传我也写过一篇文章[iOS上传文件支持断点续传](http://szuwest.github.io/iosshang-chuan-wen-jian-zhi-chi-duan-dian-xu-chuan.html)。然而FMAssetStream这个库写得很简单，没有支持断点续传，它也没有集成AFN。我想应该也有人做过这些事情，如果没有，那再自己写。

果然我找到一个更加完善的库[POSInputStreamLibrary](https://github.com/pavelosipov/POSInputStreamLibrary)，这个库既支持了断点续传，而且还可以很方便的跟AFN一起结合使用。这正是我要找的库，并且这个库更加完善和可配置，代码也写得很好。

我最终使用了这个库，改完相关代码之后，上传文件起来飞快。感谢这个库的作者。我心中的横了很久的一块石头终于落地。

