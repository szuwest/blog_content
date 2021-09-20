Date: 2017-09-26
Title: 适配iOS11与iPhone X需要注意事项
Category: 技术
Tags: iOS11 iPhoneX

#适配iOS11与iPhone X需要注意的事项

最近苹果发布了iOS11和iPhone X，我之前开发的一个应用界面出现了问题，然后需要适配。记录一下需要注意的事项。

##适配iOS11
第一个问题是iOS11导航栏改变了。除了加入largeTitle这种特性以外，还改变了很多东西，可以说是完全重写了导航栏。我遇到的问题是导航栏返回按钮图片变得很大，巨丑无比，所以得看看什么原因。

* 我看了下我导航栏的设置代码，还做了挺多事，而且做了很多hack。第一是通过 **[UINavigationBar appearance].backIndicatorImage** 和 **[UINavigationBar appearance].backIndicatorTransitionMaskImage**设置了返回按钮图片，还有 

````
[[UIBarButtonItem appearance] setBackButtonTitlePositionAdjustment:UIOffsetMake(NSIntegerMin, NSIntegerMin) forBarMetrics:UIBarMetricsDefault];
````
设置了把返回按钮右边的文字隐藏掉。还有通过hack 方式找到 _UINavigationBarBackIndicatorView把它的frame给改了。

因为iOS11的导航栏变化很大，_UINavigationBarBackIndicatorView这个类找不到了，所以frame修改就失败了。然后因为我发现我设置的返回按钮的那种图片本来是很大，因为设置了frame把它缩小了。而现在frame已经设置不了，所以就按图片原本的大小显示了。最后解决办法就是把图片大小设置成符合导航栏的大小。

图片大小是正常了，但是还是有别问题，因为返回按钮不是显示在垂直方向的中间，显示在偏下方了。最后发现是**[UIBarButtonItem appearance] setBackButtonTitlePositionAdjustment**这句代码导致的。所以我判断了如果是iOS11系统的话，就不再调用这句代码。

图片正常了，显示位置也正常了，但是上面那句代码去掉之后，返回按钮右边的文字显示出来了。这又是个麻烦事。
找了很多资料，还是没有找到理想的方案。最终在Stack Overflow上找到一种方式。在我的BaseViewController里viewDidLoad方法里，加入下面代码

````
self.navigationItem.backBarButtonItem = [[UIBarButtonItem alloc] initWithTitle:@"" style:UIBarButtonItemStylePlain target:nil action:nil];
````
这样可以完美去掉返回按钮右边的文字，没有副作用。

iOS11还有很多其他的改变，网上很多文章讲。我暂时只遇到了导航栏的问题，其他的还好。

##适配iPhone X
由于iPhone X是全面屏，并且分辨率变了，所有iPhone X上可能也需要适配。我的APP就是出现了一点问题。

iPhone X的UI上变化很大的一点是状态栏，它不再是20dp,而是44dp。如果你的界面上硬编码了这个，那就很有可能会出现问题。

我的APP上用了第三方的一个叫ViewPageController的类，这个类专门是用来做tab切换的。由于它在代码了写frame的时候，计算时参考状态栏的高度是20，所有iPhone X就出问题了。暂时我的解决办法就是：判断是iPhone X的话，就把状态栏的高度定为44.那么如果判断设备是iPhone X呢。

一般来说主要有两种方式：
一是通过屏幕来判断

````
if (UI_USER_INTERFACE_IDIOM() == UIUserInterfaceIdiomPhone) {
    CGSize screenSize = [[UIScreen mainScreen] bounds].size;
    if (screenSize.height == 812.0f)
        NSLog(@"iPhone X");
}
````
二是通过model来判断

````
NSString* modelID = [[[UIDevice currentDevice] modelIdentifier];
BOOL isIphoneX = [modelID isEqualToString:@"iPhone10,5"] || [modelID isEqualToString:@"iPhone10,6"];
````
第二种方式需要从官网上确认一下还没有没别的型号。


