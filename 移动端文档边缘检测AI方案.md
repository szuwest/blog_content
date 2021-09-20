Date: 2020-11-27
Title: 移动端文档边缘检测AI方案
Category: 技术
Tags: 边缘检测 HED OpenCV TensorFlowLite

# 移动端文档边缘检测AI方案
### 需求
先说一下我们的需求：
我们需要用户拿手机扫描自己的体检单，然后我们识别体检单的内容，结构化数据后存起来用。
这里面解决方案一般都是手机客户端拍一张照片上传到服务器识别。而这张照片的好处直接影响了服务器的识别准确率。如果照片里有掺杂这别的内容就更不好了，最好照片里只有文档内容。这里就使用到了边缘检测技术。

### 传统的边缘检测技术
在传统的边缘检测方案中，大部分都是采用OpenCV里的边缘检测算法。OpenCV库在图像处理和识别方面真是鼎鼎大名，运用十分广泛。OpenCV库里的用到的叫cany和findContours算法，而且cany还有好几个版本。但是这个算法也不完美，很容易识别错，因为现实的场景也很复杂。

### HED算法方案
现在哪里都流行用AI算法来优化。我在网上找到了fengjian大神的文章。原来我很早的时候就看到他写的[这篇](http://fengjian0106.github.io/2017/05/08/Document-Scanning-With-TensorFlow-And-OpenCV/)文章。然后就按照他的方案来开干。我运行了他的demo，大致能得到满意的效果。但是他的文章很旧了，用到的TensorFlow还是很老版本，而且只开源了iOS版本。

经过我们的摸索，还有参考了别的一些demo，我终于把它移植到了TensorFlowLite，而且我还做了安卓的版本。这样两端的解决方案一致，可以运用到正式项目中。

当然，我们自己也做了很多优化，还有自己重新收集了一些我们场景的照片来训练新的模型。最终达到了不错的效果。

我把最初移植做的demo放到github上，给有需要的人参考。这里最大优势就是安卓和iOS的方案都有。

[安卓版本](https://github.com/szuwest/doc_detect_android)

[iOS版本](https://github.com/szuwest/doc_detect_ios)

----------------
如果你觉得这篇文章有用，请打赏小钱喝杯咖啡^_^
![打赏](https://raw.githubusercontent.com/szuwest/szuwest.github.io/master/images/2018-02-21%20133111.jpg)
