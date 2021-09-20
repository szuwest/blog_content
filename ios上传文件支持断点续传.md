Date: 2016-09-30
Title: iOS上传文件支持断点续传
Category: iOS
Tags: 上传 断点续传 NSInputStream

##iOS上传文件支持断点续传

在挺久之前我写过一篇文章里提到上传文件的断点续传的问题，我没有找到好的方法。以前我采用的方式是用**NSFileHandl**的方法seekToFileOffset，移到已经文件已经上传了的部分，然后采用readDataToEndOfFile读取剩下部分到内存中NSData。但是这个方案问题是，如果文件很大，需读取的NSData很大，内存就会爆掉。所以最终没有采用这个方案。


最近我们来个新同事，技术能力很不错。我让他去研究一下这个问题。开始他找到的方案跟我之前的那个是一样的，我说这个我之前有考虑过，不能采用。然后他继续研究，后来发现了原来**NSInputStream**有相关按offset读取文件的接口。不，正确来说是**NSStream的**接口，而且有点隐蔽性质的。


`````
- (BOOL)setProperty:(id)property forKey:(NSStreamPropertyKey)key;

NSStreamFileCurrentOffsetKey
--Value is an NSNumber object containing the current absolute offset of the stream.

`````

只要对NSInputStream指定它的NSStreamFileCurrentOffsetKey的值为你想要的offset，文件在读取的时候就会从这个offset开始读取。

有了这个API，用AFN来支持上传的断点续传，也就很容易了。不过要支持这个时候AFN的progress就是需要上传部分的progress。要展示整个文件的progress的话，需要再转化一下。

现在断点续传的问题已解决，就还剩下 看不导出系统相册里的视频文件，直接读取上传这个问题没有解决了。看看什么时候能把这个问题也解决掉^_^