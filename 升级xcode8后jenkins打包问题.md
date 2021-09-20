Date: 2016-09-29
Title: 升级Xcode8后Jenkins打包问题
Category: iOS
Tags: Jenkins Xcode8 iOS10 framework

##升级Xcode8后Jenkins打包问题
上次说升级Xcode8之后，Jenkins自动打包就不行了，今天终于弄好了。前前后后话的时间有一天的时间才搞好，不容易。尝试了30多次才成功了，说多了都是泪。。现在记录一下。

###我们的需求
说一下我们的需求。我们开发人员采用的是正式签名，正式的BundleID，但是我们Jenkins自动打包出来的是企业版，用的是企业版BundleID。代码是同一份，但是Jenkins打包企业版时时要先修改BundleID和相关版本号之类的。在Xcode7时代，我已经将Jenkins配置好，可以正常打出企业版的安装包。但是升级到Xcode8之后，Jenkins打包会报错，即原来的配置已经不能打出企业包来了。

###解决方法
先说不能打包的原因，主要是签名方式冲突。Jenkins的配置是指定签名，而我们Xcode8采用的是自动签名。我们开发时采用自动签名，而且打企业版安装采用的签名文件跟我们开发时的签名文件时不一样的。所以Jenkins打包肯定得采用手动签名方式。

报错如下：

````
xxxxx has conflicting provisioning settings. xxxxx is automatically signed, but code signing identity xxxxxxxxxxxxx has been manually specified. 
Set the code signing identity value to "iPhone Developer" in the build settings editor, or switch to manual signing in the project editor.
````

所以解决方法也很直接，就是不采用自动签名。可是我找遍了国内外网上资料，没有人提供 自动关闭 自动签名 的方法。倒是有人在苹果的网站上提问如何关闭自动签名。我在国内一些网站上也有一些人提到这个问题，其中有人用 bashsell 的sed 文本替换命令来修改project.pbxproj里面的内容。

总之我的方向就是关闭自动打包，修改一些配置，然后让它可以打包。说起来简单，自动签名方式怎么关闭，需要改哪些配置，我不知道，只能摸索和不断的尝试。

一开始我尝试按照网上的一个列子来关闭自动签名方式。它是采用 sed 命令。由于我本身对bash shell 也不是很熟悉，试了好多次都没有成功。而且到后面我发现要关闭自动签名，只将 ProvisioningStyle 改为 Manual是不够的。我自己手动的去关闭和开启自动签名，看看project.pbxproj到底哪些参数改变了，然后采用我比较熟悉的PlistBuddy命令，写了个脚本去改变这些值，来达到关闭自动签名的效果。

最终我写出来的脚本大概是这样的：


```` 
#修改ProvisioningStyle 
/usr/libexec/PlistBuddy -c “Set :objects:574F440C1AEE5EA3003F9BB5:attributes:TargetAttributes:574F44131AEE5EA3003F9BB5:ProvisioningStyle Manual”  ${WORKSPACE}/xxxxx/xxxxx.xcodeproj/project.pbxproj

#修改Distribution签名配置
/usr/libexec/PlistBuddy -c “Set :objects:2B888A201AFB4C3F005E1E12:buildSettings:CODE_SIGN_IDENTITY[sdk=iphoneos*] ‘xxxxxxxxxxxxxx’”  ${WORKSPACE}/xxxxx/xxxxx.xcodeproj/project.pbxproj

/usr/libexec/PlistBuddy -c “Set :objects:2B888A201AFB4C3F005E1E12:buildSettings:PROVISIONING_PROFILE  'xxxxxxxxxxxx'”  ${WORKSPACE}/xxxxx/xxxxx.xcodeproj/project.pbxproj

/usr/libexec/PlistBuddy -c “Set :objects:2B888A201AFB4C3F005E1E12:buildSettings:PROVISIONING_PROFILE_SPECIFIER xxxxxxxxxxxx”  ${WORKSPACE}/xxxxx/xxxxx.xcodeproj/project.pbxproj

#修改Debug签名配置
/usr/libexec/PlistBuddy -c “Set :objects:574F443B1AEE5EA4003F9BB5:buildSettings:CODE_SIGN_IDENTITY[sdk=iphoneos*] ‘iPhone Distribution’”  ${WORKSPACE}/xxxxx/xxxxx.xcodeproj/project.pbxproj

/usr/libexec/PlistBuddy -c “Set :objects:574F443B1AEE5EA4003F9BB5:buildSettings:PROVISIONING_PROFILE  'xxxxxxxxxxxx'”  ${WORKSPACE}/xxxxx/xxxxx.xcodeproj/project.pbxproj

/usr/libexec/PlistBuddy -c “Set :objects:574F443B1AEE5EA4003F9BB5:buildSettings:PROVISIONING_PROFILE_SPECIFIER xxxxxxxxxxxx”  ${WORKSPACE}/xxxxx/xxxxx.xcodeproj/project.pbxproj

#修改Release签名配置
/usr/libexec/PlistBuddy -c “Set :objects:574F443C1AEE5EA4003F9BB5:buildSettings:CODE_SIGN_IDENTITY[sdk=iphoneos*] ‘iPhone Distribution’”  ${WORKSPACE}/xxxxx/xxxxx.xcodeproj/project.pbxproj

/usr/libexec/PlistBuddy -c “Set :objects:574F443C1AEE5EA4003F9BB5:buildSettings:PROVISIONING_PROFILE  'xxxxxxxxxxxx'”  ${WORKSPACE}/xxxxx/xxxxx.xcodeproj/project.pbxproj

/usr/libexec/PlistBuddy -c “Set :objects:574F443C1AEE5EA4003F9BB5:buildSettings:PROVISIONING_PROFILE_SPECIFIER xxxxxxxxxxxx”  ${WORKSPACE}/xxxxx/xxxxx.xcodeproj/project.pbxproj
````
归纳起来，就是先将签名方式ProvisioningStyle改为手动的，然后将Distribution，Debug, Release 相关的CODE_SIGN_IDENTITY[sdk=iphoneos*]，buildSettings:PROVISIONING_PROFILE，PROVISIONING_PROFILE_SPECIFIER的值做对应的修改。另外BundleID和develop team我没有用PlistBuddy来修改，而是通过在Jenkins的配置**Custom xcodebuild arguments** 加入参数 `DEVELOPMENT_TEAM=DLF3GQKP4Q PRODUCT_BUNDLE_IDENTIFIER=xxxxxxxxx` 来指定。这样编译器会自动修改相关值

配置完这些，ipa包是能打出来了，但是却最后失败了，报错是签名问题，还有找不到ResourceRules.plist。

最终我发现只要 把Jenkins配置中的把 Sign IPA at build time 的勾选去掉，然后打包就没有问题了。然后我也把在Jenkins的配置**Custom xcodebuild arguments** 参数中，原来有的参数 `CODE_SIGN_RESOURCE_RULES_PATH=$(SDKROOT)/ResourceRules.plist` 去掉，打出来的安装包也没有问题。当时这个参数在Xcode7是必填的，现在Xcode8却可以去掉了。

以上就是我被Xcode8折腾经验总结。

##后记：补充
这篇文章写了不久，就收到了Soto同学的反馈。根据这篇文章他也成功打包了，而且他还总结了一些更方便的方法。我补充一下，后续有同样问题的人就可以不那么折腾了。

可以通过Jenkins的配置*Custom xcodebuild arguments* 参数来修改的参数有：

* PRODUCT_BUNDLE_IDENTIFIER
* DEVELOPMENT_TEAM
* PROVISIONING_PROFILE
* PROVISIONING_PROFILE_SPECIFIER
* CODE_SIGN_IDENTITY

其中CODE_SIGN_IDENTITY[sdk=iphoneos*]貌似不是必须的，在xcodebuild的参数指定**CODE_SIGN_IDENTITY="iPhone Distribution"**就行了。

那么只有**ProvisioningStyle**要通过脚本来修改。而且 还可以通过 sed 命令来修改，不必用PlistBuddy来修改（用PlistBuddy要先找到相关UUID，很麻烦）

````
sed -i '' 's/ProvisioningStyle = Automatic;/ProvisioningStyle = Manual;/' ../#{project}/project.pbxproj
````
Soto同学就是采用这种方法成功打出了企业版包和AppStore包，成功避过了要查找UUID来定位修改相关参数值的麻烦做法。这是一种好方法。

不过这种改法不能适应所有情况，我的项目就不能这么做。我的项目依赖于另外一个工程编译出来的framework，如果在xcodebuild参数里指定PROVISIONING_PROFILE和PROVISIONING_PROFILE_SPECIFIER，是不行的，会造成编译这个framework的时候报错。这个framework工程的配置也要注意，是要把自动签名关闭，并且team指定为none。所以我还是保持我原来的做法，虽然很笨，不灵活，但是能解决问题。

如果谁跟我的情况类似，然后有更好的办法，麻烦告诉我^_^

##升级macOS Sierra 后打包签名失败问题
这个问题是Soto同学遇到的，我没遇到，因为我还没升级到最新系统。这里也说一下，给有需要的人用。

如果你打包出现签名错误，例如：

Command /usr/bin/codesign failed with exit code 1, 并且还 returned: -25308, unknown error -1=fffffffffff 之类的关键字，很有可能就是因为升级到了最新系统macOS Sierra导致的。

解决方式：

打开keychain，然后找到你打包用到的那个证书，展开证书，选中那个密钥，右击-->显示简介-->访问控制，然后选择 **允许所有应用程序访问此项目**,就可以了。

感谢Soto同学的反馈


-----
**由于我的网络经常连不上评论系统Disqus，所以有问题想问我或者想跟我交流的，请直接给我发邮件或者加我QQ411084057.谢谢**