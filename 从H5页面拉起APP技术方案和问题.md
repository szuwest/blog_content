Date: 2020-03-07
Title: 探讨从H5页面拉起APP的技术方案和问题
Category: 技术
Tags: APP


# 探讨从H5页面拉起APP的技术方案和问题

从H5页面里拉起或叫唤起APP（APP需已经安装在手机上），有两种技术方案，一种通过HTTP链接（iOS叫 Universal Link，安卓叫APP Links技术）拉起技术，一种是通过自定义scheme拉起技术（也有叫深度链接deeplink的）。这两个技术适用不同的场景，也有不同的局限性。


## 一、通过HTTP链接拉起

要想通过HTTP(s)链接直接拉起APP，需要APP本身做一些配置和写一些代码实现，还需要在你网站服务器配置相关的json文件。例如配置服务器host，能打开页面的path等。有些了这些配置，苹果服务器或者谷歌的服务器验证了这些配置，就可以通过相关的HTTP链接来打开APP了

这里不管是iOS叫 Universal Link，还是安卓的APP Links，他们的技术方案是基本一样的，只是不同的平台稍微有些不一样的而已。基本规则是：

* 先按照要求生成一个配置文件，json格式的，里面包括了一些必要的信息，例如安卓的包名，apk签名等，iOS的appID,匹配URL的path
* 将配置文件放到你的网站根目录下的.wellknown目录下，让苹果或谷歌的服务器能从这里下载这个配置文件。这个文件是关键，因为苹果或谷歌要从你网站里拉取到这个配置文件。由于你的网站只有你自己有权限上传配置文件，从而保证了安全性。
* 在你的APP里配置相关网站host。还有你的APP被拉起后怎么处理（打开相应页面）

网络上有很多相关教程，官方文档也有，配置起来不难，测试起来有几个问题。

* 确保你的手机能跟苹果或者谷歌的服务器能正常通信
* 验证是在你安装APP的时候，只在这时候验证。所以你遇到奇怪的问题可以卸载重装试试。对于iOS来说，你修改了配置文件中的path后，要你的APP升级后或重新安装后才会生效
* iOS需要跨域才能跳转。就是说你配置的host是www.xxx.com,在Safari中打开www.xxx.com后，在这个页面跳转www.xxx.com/yyyy, 即使你在配置文件里配置了yyyy这个path，这里并不会拉起你APP。因为这里并没有跨域。但是在ww.xxx.com/yyyy这个页面上，你往下拖动，就会看到顶部有个banner（Safari自动生成的），可以通过这个banner打开APP。


## 二、通过自定义scheme拉起

首先让我们来看一下 URL 的组成：

````
[scheme:][//host][path][?query][#fragment]
````

这里一般做法是每个APP都自定义一个唯一的scheme，通过这个scheme来拉起APP。由于每个APP都会定义自己独有的scheme，所以一般还通过scheme来判断某个应用是否已安装在手机上。host是可选的，path是必须的。一般都是通过path来定义指定打开APP特定的页面，同时通过query来携带参数。HTTP链接其实就是一种特殊的scheme。

通常我们都是同构scheme来拉起别人的APP的。例如超级APP微信的scheme是weixin://,淘宝APP的scheme是taobao://。

scheme方式是安卓和iOS都支持的。我们微基因APP的scheme是wegene，host是com.wegene.app，首页的path是/main.要打开微基因APP首页，可以通过下面这个URL：wegene://com.wegene.app/main。我们可以在H5页面里配置这个URL超链接，点击这个超链接就可以打开APP了。

当然，APP里还是要做相应处理的。iOS要先在xcode里配置scheme，还有在AppDelegate里回调里会获取到这个URL，解析这个URL，获取到path，根据是/main，就可以知道是要打开首页了。而安卓则需要在你的MainActivity的intent-filter里配置data(scheme,host,path)，action，category等(该activity要设置export为true)

````
 <intent-filter>
                <action android:name="android.intent.action.VIEW" />

                <category android:name="android.intent.category.DEFAULT" />
                <category android:name="android.intent.category.BROWSABLE" />

                <data
                    android:host="com.wegene.app"
                    android:path="/main"
                    android:scheme="wegene" />
            </intent-filter>

````

在activity里还要从intent里获取URI的data数据。

## 三、浏览器问题

浏览器一般有几种，一种是手机自带浏览器（安卓厂商自带浏览器，iOS的Safari），一种是通用手机浏览器，例如UC浏览器，QQ浏览器，chrome，还有一种是超级APP自带浏览器，例如微信、QQ、微博他们APP内自带浏览器

超级APP自带浏览器问题：它们都禁止了通用链接方式，也基本禁用了scheme方式。但是他们提供了一种方式给用户选择：选择第三方浏览器打开。支持通用链接技术的APP可以用这种方式来被打开。

手机自带浏览器，通用浏览器问题：不同的浏览器APP实现方式不一样。总的来说，一般都支持通用链接方式和scheme方式。但是有些可能浏览器会弹出一个弹窗来让用户确认是否同意拉起APP。


## 四、安卓端问题
虽然安卓支持APP Links技术，但是这个技术依赖于谷歌服务器的验证，而谷歌服务器是被墙的，所以是在国内基本不能用。所以在安卓端能用的技术只有scheme技术了。

一般浏览器可以通过scheme来跳转，但是微信里是禁用了scheme方式的。这里一种解决方案是 微信选择第三方浏览器打开-->进入浏览器-->浏览器调用scheme-->进入APP相关页面。这里面浏览器页面就要做相关的检测和跳转配置了，页面多的话还要做不同页面的配置，开发量也不小。

还有一种方案，就是利用应用宝的微下载和他的APPLink机制（非谷歌的APPLinks）。实现方式是 微信浏览器打开应用宝配置的页面-->拉起应用宝-->应用宝通过我们配置的scheme和path拉起我们的APP-->进入APP相关页面。这里应用的技术仍然是scheme技术（deeplink），我们在应用宝里配置好scheme和path，由它来拉起我们的APP。但是前提是我们要向应用宝申请，而且需要APP符合他们的一定条件才会通过申请。


## 总结
总的来说要在要在H5中拉起APP，无非就是通过专属HTTP链接技术和自定义scheme技术。然而理想很丰满，现实很骨感。在国内的所有浏览器，无论是通用浏览器APP，还是超级APP自带的浏览器，都是会做一些改造和限制。当然我们最关注的是流量霸主微信，都想从他那里引流一些用户。可是微信确实最严厉的，禁止了HTTP链接拉起APP的技术，也禁用了scheme技术，只给自家APP或者合作方留了口子。现在的做法都是引导用户点击右上角的 第三个浏览器打开来拉起APP。

现在我们要想适配好全面的H5拉起APP技术，也挺难的，要考虑到微信微博等超级APP禁止所有的跳转技术，还有各式各样的浏览器支持程度也不同。现实的情况是我们做APP开发的不太懂网页开发，做网页开发的又不懂我们APP开发（还有相关的跳转技术），所以难上加难。倒是可以借助开源试试。有个还不错的开源库看上去还不错（我没仔细研究过）[callapp-lib](https://github.com/suanmei/callapp-lib)


----------------
如果你觉得这篇文章有用，请打赏小钱喝杯咖啡^_^
![打赏](https://raw.githubusercontent.com/szuwest/szuwest.github.io/master/images/2018-02-21%20133111.jpg)