Date: 2018-11-03
Title: SVG图片在移动端的应用解决方案
Category: 技术
Tags:算法 SVG androidsvg glide Macaw 

## SVG图片在移动端的应用解决方案
近几年来SVG使用得越来越多，就连Android的官方库也加入VectorDrawable的支持。这个类就是用来支持向量图的。SVG图片在web端使用非常广泛，我第一次接触这个也是在做react-native的项目中使用的。当时我们要做一些动画，需要从一个形状变换成另一个形状，这种一般都是用矢量图来做的。当时设计师就给了我一些矢量图，于是我就开始研究这个东西。

在react-native中，有专门一个库叫[react-native-svg](https://github.com/react-native-community/react-native-svg)来处理这个。不过当时要做两个SVG形状的动画变化，并不是任何一个形状都可以的，需要遵循一定的标准。设计师给我的两个SVG文件并不能转换。后来是我自己根据文件里的一些关键参数自己在代码里直接画出来后，再做转换动画。

最近我在做的native项目中，也遇到了要用SVG的图片。我们的项目里要从服务器下载SVG图片来展示。我们要实现的这些文件是需要服务器动态配置的，也就是说我们不能预先打包进我们的APP里。所以我们这里的要提供一个解决方案，跟图片JPG图片一样显示，缓存。

这个需求跟之前我遇到的那个需求是很不一样的，之前的是设计师已经定义好图片，我们工程师直接拿到文件在程序里展示，不需要考虑下载和缓存之类的。这种需求其实很简单，我们实际上大部分的需求就是这种需求，网上有很多库可以完成这种需求。把SVG图片跟JPG等普通图片一样使用，网上的方案还真不多，特别是iOS方面。。。。

要像普通图片一样使用，就要考虑下载，本地缓存，内存缓存。像这种需求，我们移动端都会使用专门的图片框架，像安卓端的glide，UIL等，iOS端的SDWebImage等。但是这种库它是默认都不会考虑SVG图片。但是我们最好还是像使用这种框架来处理SVG图片。最好的方式就是把SVG的支持集成到这些库中。好在这些库的优点就是容易扩展。


### 安卓端的解决方案
由于我们的项目是采用glide框架来处理图片，所以这里就只讲在这个框架集成SVG图片的展示。

实际上glide真的是一个很强大的库，怪不得那么多人用它（早几年我们都是用UIL），它在它的sample例子里就提供了SVG的展示支持[svg](https://github.com/bumptech/glide/tree/master/samples/svg)。在这个例子里，采用的SVG解码方式是使用外部的解码库。它采用的解码库是[androidsvg](https://github.com/BigBadaboom/androidsvg)。这个库是web端移植过来的，所以它有很好的兼容性，是很不错的库，虽然它的star不是很多。Android就是好，有强大的Java社区，受益于这些社区，很多库都不错。而这方面iOS就那么好了，这个等一下再说。

按照它提供的sample来集成SVG的支持，不是很难。但是我遇到了其他问题。因为我们项目里的glide使用的是3.7版本，sample是基于4.8版本的。这两个版本在API上有很大区别，变化很大。所以我必须要先升级到4.8版本。等我升级完后，接入SVG的支持，然而SVG图片死活显示不出来。最终发现是我的AppModule无法生成。一直在文档，查资料，还是找不到问题，我是完全按照官方文档升级和集成的。最终我发现可能是跟AndroidX相关（还不知道AndroidX是什么的自己查）。我们项目升级到了AndroidX，它会影响一些annotation生成方式。我们的项目采用插件式框架。我们很多通用库是放在一个commomlibrary的Module中，glide也是。APP module就只是一个壳。但是一些annotation的声明一定要放在APP module中才行。所以我们把 

````
annotationProcessor 'androidx.annotation:annotation:1.0.0'
annotationProcessor 'com.github.bumptech.glide:compiler:4.8.0' 
````
放在了APP module中，最终那些自动生成的代码才会真正的生成。这样就可以很愉快的用glide来显示SVG图片了。

````
public class GlideSvgUtil {

    //显示网络中的svg文件
    public static void showSvg(ImageView imageView, String url) {
        Glide.with(imageView.getContext()).as(PictureDrawable.class).listener(new SvgSoftwareLayerSetter()).load(url).into(imageView);
    }

    //把svg放入到raw中，通过rawid来显示
    public static void showSvgRes(ImageView imageView, int rawId) {
        Uri uri = Uri.parse(ContentResolver.SCHEME_ANDROID_RESOURCE + "://" + imageView.getContext().getPackageName() + "/"
                + rawId);
        Glide.with(imageView.getContext()).as(PictureDrawable.class).listener(new SvgSoftwareLayerSetter()).load(uri).into(imageView);
    }

    //把svg的XML加载到字符串中来显示
    public static void showSvgContent(ImageView imageView, String svgContent) {
        Glide.with(imageView.getContext()).as(PictureDrawable.class).listener(new SvgSoftwareLayerSetter()).load(svgContent.getBytes()).into(imageView);
    }
}
````

### iOS端的解决方案
iOS的方案还不是很好解决，我没有发现有哪一个图片框架是集成了SVG或者提供了集成的例子的。而且我们的同事还发现了一个问题。我们服务器提供的SVG图片在很多库解码出来后没有了颜色，是黑白的。很诡异的问题，然而安卓端没有这问题。我们找很多库，像SVGKit，SwiftSVG，PocketSVG，Macaw，这些库都是超过1000star的，都无法正常显示。我基本确定是我们SVG文件的兼容性问题，我问我们的设计师他是怎么生成SVG文件的。他说是用sketch导出来的。这些文件在web和安卓的库，还有甚至xcode里都是能正常显示的。这里我提供一个不能正常显示的图片。


````
<svg id="图层_1" data-name="图层 1" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 80 80">
<defs><style>.cls-1{fill:#ff9595;}.cls-2{fill:#ffcc80;}</style></defs>
<title>彩色图标</title>
<path class="cls-1" d="M43,77A27.14,27.14,0,0,0,58.64,27.67a24.55,24.55,0,0,1-6.4,8A24.44,24.44,0,0,0,34.55,3a24.48,24.48,0,0,1,.87,6.49c0,21.19-25.57,21.72-25.57,42.37C9.85,64.9,20.47,77,43,77Z"/><path class="cls-2" d="M51,66c0-9.28-11.79-11.85-11.79-21.74A12.3,12.3,0,0,1,40,39.95,20.79,20.79,0,0,0,34.77,76.4,54.34,54.34,0,0,0,43,77c.86,0,1.7,0,2.53-.12A13.47,13.47,0,0,0,51,66Z"/>
</svg>
````
因为是颜色不能正常显示，我猜肯定是defs标签里的内容不能正常解析。我稍微改了下文件，就能正常显示了。我还发现对SVG显示支持的比较好的是Macaw，其他的貌似都有点小问题。所以我把这个问题在Macaw上提交了。我不知道他们会什么时候修改，所以我就自己去研究他们的源代码，准备自己来解决了。很快我就发现他们在预解析阶段，解析defs标签的时候，就没有考虑style这种子标签。所以我就加上去了，就两三行代码（他们的代码架构挺好），然后就正常显示了。后来我又到Macaw网站上看，他们已经回复我了，并且已经支持！前后也就一个多小时！他说defs标签里一般不会放style标签，不过很多其他库支持，所以他们也就支持了。我去看他改的代码，几乎跟我改一样。所以我就放弃自己的改动，采用cocoapods直接拉取他们的master上的最新代码。

解码的问题解决了，但是怎么集成到图片框架里呢？我们项目是采用swift开发的，我们采用的图片处理框架是Kingfisher。我去Kingfisher的网站上看，喵神很厉害，已经有很好的文档教我们怎么扩展图片解码器。我一看，另一个问题来了，解码器是在工作线程中进行的，并且要返回一个UIImage。Macaw这个库并没有提供一个可以在子线程中将SVG文件转为UIImage的方法。他们的方法是将SVG显示在一个UIView里然后截图。。。也有人将这个问题提了。我看他们源码发现有支持的，但是没有放出来，无法使用。

我最终也只能采用曲线救国的方法了。还是用他们的方法，我在解码线程里同步切到主线程中生成UIImage，然后再在子线程中返回这个UIImage。好消息是，最近两天，他们终于支持将一个文件直接转换为UIImage了具体方案看[这里](https://github.com/exyte/Macaw/issues/468)，不过我还没测试，他们也还没发新版。

SVG解码器

````
import Foundation
import UIKit
import Kingfisher
import Macaw

struct SVGProcessor:  ImageProcessor {
	var size = CGSize(width: 32, height: 32)
	init(_ size: CGSize) {
		if size.width == 0 || size.height == 0 {
			print("不支持size为0的情况m，将采用默认值32")
		} else {
			self.size = size
		}
	}
	
	let identifier: String = "com.wegene.future"
	
	func process(item: ImageProcessItem, options: KingfisherOptionsInfo) -> UIImage? {
		switch item {
		case .image(let image):
			print("already an image")
			return image
		case .data(let data):
			let svgContent = String.init(data: data, encoding: String.Encoding.utf8)
			var img: UIImage?
			DispatchQueue.main.sync {//现在Macaw库暂时只支持这种方式生成UIImage,下一版他们支持后台线程生成UIImage的方式，以后再做修改
				let rootNode = try! SVGParser.parse(text: svgContent!)
				let macawView = MacawView(node: rootNode, frame:CGRect(origin: CGPoint.zero, size: size))
				UIGraphicsBeginImageContextWithOptions(size, true, UIScreen.main.scale)
				macawView.layer.render(in: UIGraphicsGetCurrentContext()!)
				img =  UIGraphicsGetImageFromCurrentImageContext();
				UIGraphicsEndImageContext();
			}
			return img
		}
	}
}
````


UIImageView的扩展

````
extension UIImageView {
	func setSvgImage(_ url: String) {
		let processor = SVGProcessor(self.frame.size)
		let _url = URL(string: url)
		self.kf.setImage(with: _url, options:[.processor(processor)])
	}
}
````

谢谢阅读。

----------------
如果你觉得这篇文章有用，请打赏小钱喝杯咖啡^_^
![打赏](https://raw.githubusercontent.com/szuwest/szuwest.github.io/master/images/2018-02-21%20133111.jpg)