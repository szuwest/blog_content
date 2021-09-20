Date: 2020-02-08
Title: 上传应用到苹果一直卡住验证中（authentication with the app store）问题解决
Category: iOS
Tags:  验证中 一直卡住

# 上传应用到苹果一直卡住验证中（authentication with the app store）问题解决
虽然最近因为新型冠状病毒导致的疫情一直在持续，我们都被迫在家办公。我们由于有一些紧急的改动需要发版，5号那天我们改好之后就准备上传APP审核。安卓APP上传到各个商店都还挺顺利，但是iOS的却遇到难题了，卡住了，卡在 **正在验证APP - 正在通过App Store进行认证**。

![验证中](./images/app_verifying.jpg)

自从升级了xcode11后，我一般都用**transporter**来上传IPA。因为xcode上传有时候也很慢，但是transporter却是很快。但是这次transporter卡住了，一直在**正在验证APP - 正在通过App Store进行认证**状态。过了很久都不动，平时几分钟就搞定了。后面我尝试了各种方法，试过xcode上传，命令行xcrun上传，搭梯子上传，通过手机热点上传，让同事他们上传，各种方法。搞了两三天，才最终搞定，真的是太悲催。感觉苹果在这块做得真是太傻逼了。


* 方法一：transporter，xcode上传都是卡住，xcrun上传报错
* 方法二：搭梯子上传，transporter报错：交付到App Store时出错。xcode也报类似错误
* 方法三：通过手机热点上传，一样卡住在同样的地方（移动信号和电信信号，没试过联通的，因为没联通手机。有人通过热点上传成功的可能是联通手机？）

网上很多帖子都在说iTMSTransporter缓存问题，其实这个思路是对的。但是绝大部分的文章都是很老之前的了，说的都是Application Loader的事。也有一小部分说transporter的缓存问题。我试了通过transporter.app里面的iTMSTransporter来下载缓存，同样也是卡住。也试过翻墙来下载，但是貌似没什么效果。不过这个也有可能我之前的那个梯子不太好，很慢导致的。为了解决这个慢的问题，我又花了100大钞弄了个新的梯子。新的梯子网速笔记快。

反正我各种方法试，交叉试，组合试，连续两天都没解决，在iOS开发群里喊没人回应，最终在cocoachina论坛看到一个有用的帖子[（地址在这）](http://www.cocoachina.com/bbs/read.php?tid-1795056.html)。在这个帖子了试了一些人的方法没解决。但是其中有一个人给了个Stack Overflow的帖子地址，最终按照那个帖子的方法解决了。**Stack Overflow**真TM香！还是**Stack Overflow**靠谱！

那个人分析得非常到位，我引用到这里。
> **Dec 10th 2019, Xcode Version 11.2.1, MacOS X 10.15.1**
<br>
I was facing exactly same issue yesterday and I thought it might be network issues, at least it looks like so. But this morning I had tried couple different networks and several VPN connections, none of them is working!
<br>
The highest voted answer here asks me to reset a cache folder named .itmstransporter under my home dir, the run a program iTMSTransporter under a specific folder, but I can't find both of them.
<br>
But soon I figured that it is the cache folder for the people who uses the legacy uploader program: Application Loader, which is deprecated by Apple and can be no longer found in Xcode 11. Then I found that the latest Xcode has located iTMSTransporter here:
<br>
````
/Applications/Xcode.app/Contents/SharedFrameworks/ContentDeliveryServices.framework/itms/bin/iTMSTransporter
````
<br>
And its cache folder is here:
<br>
````
/Users/your_user_name/Library/Caches/com.apple.amp.itmstransporter/
````
<br>
I removed my existed cache folder, and run iTMSTransporter without any parameter, it soon started to output logs and download a bunch of files, and finished in 2 or 3 minutes. Then I tried again to upload my ipa file, it works!!!
<br>
CONCLUTION:
<br>
>> * 1.Either the old Application Loader, or the latest Xcode, uses a Java program iTMSTransporter to process the ipa file uploading.
* 2.To function correctly, iTMSTransporter requires a set of jar files downloaded from Internet and cached in your local folder.
* 3.If your cache is somehow broken, or doesn't exist at all, directly invoking iTMSTransporter with functional parameters such as --upload-app in our case, iTMSTransporter DOES NOT WARN YOU, NOR FIX CACHE BY ITSELF, it just gets stuck there, SAYS NOTHING AT ALL! (Whoever wrote this iTMSTransporter, you seriously need to improve your programming sense).
* 4.Invoking iTMSTransporter without any parameter fixes the cache.
* 5.A functional cache is about 65MB, at Dec 10th 2019 with Xcode Version 11.2.1 (11B500)



这个人总结得非常好。具体帖子在[这里](https://stackoverflow.com/a/59261475)

我要补充一点是这个**com.apple.amp.itmstransporter**文件下原来的只有2.8兆左右，后面重新下载下来的68左右，相差很大，开始我没翻墙来下载，下载很慢，下载3.9兆后就不动了。命令行那里也没有打印出什么来。后来我停掉，然后打开梯子，然后重新运行命令，就下载很快了，命令行也打印出来很多信息，完成后那个缓存文件夹大约68M。然后最好关掉梯子，重新打开transporter来上传，这样就能正常上传了。



----------------
如果你觉得这篇文章有用，请打赏小钱喝杯咖啡^_^
![打赏](https://raw.githubusercontent.com/szuwest/szuwest.github.io/master/images/2018-02-21%20133111.jpg)