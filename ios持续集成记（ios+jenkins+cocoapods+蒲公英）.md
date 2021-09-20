Date: 2016-07-18
Title: iOS持续集成记(iOS+Jenkins+Cocoapods+蒲公英)
Category: 技术
Tags: Jenkins Cocoapods 蒲公英

##iOS持续集成记（iOS+Jenkins+Cocoapods+蒲公英）
原来我们iOS项目是有持续集成环境的，但是有一次升级系统还是升级xcode后，Jenkins就打不出可用的安装包了。原来负责整个打包机的同事又离职了，有一段时间我们都是手动打包给QA同事安装测试，现在领导要求必须把持续集成环境弄好，我们花了大约两周才基本弄好整个持续集成环境，现在把我整个过程中遇到的坑记录一下。

##重装系统
打包机自从某一次升级系统之后就变得非常慢，打个包需要1个小时左右，简直要命。所以新的环境必须要建立在重装后的系统上。因为要格式化硬盘，所以要制作系统启动盘，这个网上也有不少教程，这里有个需要注意的，下载macOS系统时，一定要从官方下载。我一开始是自己在百度云上下的，制作好安装盘后重启电脑，并且将硬盘格式化后安装系统，发现验证不通过，好在另外一个同事在官网下了一个，我在它的电脑上重新制作了一次安装盘。假设你只要一台Mac电脑，遇到这种情况你就悲剧了。


##git拉取代码的坑
系统装好了，Jenkins也装好了，然后配置拉去代码，可是代码死活拉不下来，为此我的同事搞了差不多一天，试了各种方法，也搞不下来代码，都是在一半的时候失败了。后来我也去弄了半天，同样的错误。主要原因可能是代码量太大了，因为其他的小工程可以git clone 下来。我突然想到换一下拉取协议试试，不采用HTTP方式，而是采用git@host/xxxx.git的方式来拉取代码，果然成功了。采用这种方式后，在Jenkins里需要配置ssh的RSA秘钥，在gitlab上面要配置公钥，但是我在Jenkins里配置私钥时，拷贝私钥时，只拷贝了中间那部分，导致一直验证不过，也折腾了很长一段时间才发现，并改正了过来。

## CocoaPods的坑
我们项目用了CocoaPods，而且pods里还有我们的私有库。虽然git clone代码下来了，但是pod install死活不成功。开始以为是pod环境问题，查了很多资料也解决不了。后来我问了一个遇到过这种情况同事，他说是Podfile的source问题，改成ssh协议拉取就可以了。看来仍然是git的问题。

CocoaPods也能拉取到了，终于可以进入build步骤了。在Jenkins里配置有一个要特别注意的地方，因为使用CocoaPods的工程编译的是workspace，而不是普通的project。Jenkins里xcode配置需要指定workspace的名称，才能正确编译。

##企业版编译和修改BundleID
由于我们打出来的包需要时外部可以随时安装的，所以需要是企业证书的安装包。然而企业包的BundleID是跟正常AppStore的包的BundleID不一样的。一般做法是在正常的AppStore包的BundleID后面直接加上Enterprise。我一开始修改了info.plist后，发现还是因为BundleID跟证书不匹配而编译失败，发现BundleID还是没有变。最终网上查资料才发现，光改了info.plist的BundleID还不够，还要修改project配置里的BundleID。我在网上找了很久也没有找到比较好的修改project里参数的方法。不过最终我在stackoverflow上找个一种方法，在xcodebuild的编译参数里，可以加入一个修改BundleID的参数，这个问题就这样解决了。<br/>
**Custom xcodebuild arguments**
>* CODE_SIGN_RESOURCE_RULES_PATH=$(SDKROOT)/ResourceRules.plist PRODUCT_BUNDLE_IDENTIFIER=BundleID

随便提一下，Jenkins的配置里有一个叫Change BundleID的配置项，貌似没有什么用。

##上传和发邮件
安装包打出来了，要上传到共享服务器和发布到外网给大家安装，还要发邮件通知QA和相关人员。这一步网上很多都是通过编写Python脚本来实现。我试过Jenkins的FTP插件配置，将安装包上传到我们内部共享服务器，但是失败了，没有太多时间研究就放弃了。转而研究上传蒲公英。我试过网上的一些方法，但是只有通过curl方式上传才成功的。但是注意curl命令后的的file参数有个@。例如 curl -F "file=@path"

为了上传ipa和发邮件成功，我还查阅了不少Python的语法和bash shell的语法，收获不少。每一次折腾都是进步。
