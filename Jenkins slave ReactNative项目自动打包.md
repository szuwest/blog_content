Date: 2018-03-8
Title: Jenkins slave ReactNative项目自动打包
Category: 技术
Tags: ReactNative Jenkins iOS


# Jenkins slave ReactNative项目自动打包
这次我想在谈谈Jenkins自动打包的问题。针对react-native项目。
一般情况下，我们都是在一台电脑上安装Jenkins，然后在该电脑上打开Jenkins网页进行打包配置。如果我们有很多项目，分布在不同的电脑，而且运行的系统也不同，这时候我们需要一个master来统一管理这些电脑，然后统一在一个网页上进行打包配置。这就是Jenkins的master-slave打包模式。这种模式跟单机模式稍有不同，这里主要讲怎么配置slave。

## slave节点配置

* 首先slave上也同样要安装Jenkins和相关的打包所依赖的环境，一个都不能少。配置好之后，我们并不需要在本机上打开Jenkins网页来创建项目，配置项目等，而是要到主机上去配置。在去主机的Jenkins管理页面上新建节点之前，从机还需要做一些其他配置。
   * 打开远程登录。在系统设置-->共享，然后选中远程登录，允许所有用户访问。
   * 记住本机IP地址。在系统设置-->网络，可以看到当前网络的IP地址。为了以后方便，可以将IP地址设置为一个固定IP地址。不然当该电脑重启之后，IP地址可能会变，然后主机再按照之前配置的旧IP连接从机的时候，就连不上了。Mac电脑配置固定IP也有注意的地方。我第一次配置就是没有成功，上不了网。选择手动方式，然后填上一个没有被占用的IP地址，然后，子网掩码也要注意，如果跟服务器是跨网段访问的，需要注意255.255.255.0这种是不行的。当然按照之前默认的来一般没问题。然后最要注意的地方是要配置DNS，DNS跟路由器的地址一样。我就遇到没有配置DNS然后导致连不上网的情况。我开始以为它会自动配置的，原来还需要自己配置。   
* 主机跟从机怎么连接呢。首先在主机上的Jenkins管理页面上进入**系统管理**-->**管理节点**页面，然后新建节点。填入一个名称，之后靠这个名称来识别节点。最好能识别系统。例如叫macOS.另外一个选项，选择固定代理。
* 配置节点：这里有个远程工作目录，指的是Jenkins的配置目录。一个是/Users/你的用户名/.jenkins.启动方法我选的是Launch slave agents via ssh，然后填上从机电脑的IP地址和登录的用户名密码（Credentials）。这里如果没有添加过用户名和密码的话，需要先添加。这里还有一个叫 **Host Key Verification Strategy**的选项，我选择的是Non verifying Verification Strategy.别的好像都不行，具体的原因我也不知道。Availability选择的是尽量保持代理在线。		
Node Properties项目栏可以不用填。貌似有必要也可以填一下那个环境变量，以防一些命令找不到。

## 创建项目
配置好节点只有，可以新建项目了。怎么配置项目，网上也有非常多的文章。这里有几点要额外说明。

一是项目怎么跟从机绑定到一起呢。有个已交**Restrict where this project can be run**的选项，勾选它，然后填入之前创建的节点名称。这样这个项目的打包就是在这个节点上进行的。

其他的一些配置网上有很多，我这里不说，而且有些配置并不是必要的，每个人的需求不同。对于我的项目，我基本只配置一个 参数化构建过程，可以选择分支来构建。然后是源码管理，选择git。git的地址一般选择ssh方式。这样的方式网络连接也比较稳定，不容易出现代码过多，拉取代码太久而导致超时。

再有需要的配置就构建。Jenkins有xcode插件，可以按照插件来配置。但是我的项目是react-native项目，感觉按照一般的xcode项目来配置不是很好配置。我直接就写了一个脚本，在这脚本里做了所以的一切事情。所以在Jenkins里我只需要建立一个Execute Shell，然后在这个shell里执行我建立的脚本，一切都OK了。

注意我的脚本是放在 项目根目录/ios 目录下。而Jenkins的目录是在项目根目录下的。
Jenkins中的配置。

````
cd ios
./build_using_gym.sh

````

build\_using\_gym.sh的内容：

````
#!/bin/bash -ilex

#设置超时
export FASTLANE_XCODEBUILD_SETTINGS_TIMEOUT=120

#计时
SECONDS=0

#假设脚本放置在与iOS项目相同的路径下
#project_path="$(dirname "$(pwd)")"
project_path="$(pwd)"
#取当前时间字符串添加到文件结尾
now=$(date +"%Y_%m_%d_%H_%M_%S")
#项目名称
projectName="VprScene"

#指定项目的scheme名称
scheme="VprScene"
#指定要打包的配置名
configuration="Release"
#指定打包所使用的输出方式，目前支持app-store, package, ad-hoc, enterprise, development, 和developer-id，即xcodebuild的method参数
export_method='ad-hoc'

#指定项目地址，如果是workspace，需要修改
workspace_path="$project_path/${projectName}.xcodeproj"
#指定输出路径
output_path="/Users/xhkj/Desktop/${projectName}"
#指定输出归档文件地址
archive_path="$output_path/${projectName}_${now}.xcarchive"
#指定输出ipa地址
ipa_path="$output_path/${projectName}_${now}.ipa"
#指定输出ipa名称
ipa_name="${projectName}_${now}.ipa"
#获取执行命令时的commit message
commit_msg="$1"

#输出设定的变量值
echo "===workspace path: ${workspace_path}==="
echo "===archive path: ${archive_path}==="
echo "===ipa path: ${ipa_path}==="
echo "===export method: ${export_method}==="
echo "===commit msg: $1==="

# 解锁对login.keychain的访问，codesign会用到，需要替换为实际密码
security unlock-keychain -p "123456" $HOME/Library/Keychains/login.keychain

#react-native yarn 
#yarn install --registry http://192.168.0.9
yarn install

#cocoapods
#pod install --no-repo-update


#fasltlane build，注意如果是cocoapods工程需要修改命令
fastlane gym --project ${workspace_path} --scheme ${scheme} --clean --configuration ${configuration} --archive_path ${archive_path} --export_method ${export_method} -allowProvisioningUpdates --output_directory ${output_path} --output_name ${ipa_name}

#上传到pgyer
#蒲公英上的User Key
uKey="xxxxxxxxxxxxxxxxxxxx"
#蒲公英上的API Key
apiKey="xxxxxxxxxxxxxxxxxxxxx"
#执行上传至蒲公英的命令
echo "++++++++++++++upload+++++++++++++"
curl -F "file=@${ipa_path}" -F "uKey=${uKey}" -F "_api_key=${apiKey}" https://qiniu-storage.pgyer.com/apiv1/app/upload

#输出总用时
echo "===Finished. Total time: ${SECONDS}s==="

````

这个脚本里主要做了几件事情。

* 定义了项目名称，项目路径，打包输出路径，打包方式（ad-hoc）之类的变量定义
* 解锁keychain，需要预先填入解锁密码。因为要签名，所有要访问keychain，这里很重要，否则会导致签名失败
* 运行yarn install.这是ReactNative项目打包前必须的
* 运行cocoapods install。如果是cocoapods项目，这也是打包前必须的
* 运用fastlane来打包。fastlane的安装请自行查找资料
* 上传到蒲公英。这里要预先指定普公益的key和secret。

这里还有一个特别要注意的地方。如果你的脚本在单机上跑没有问题。但是一通过Jenkins来跑就出问题，就是 脚本第一行要加 -ilex 参数.

这个脚本的好处是，除了要解锁keychain，其他的什么开发者证书，teamID都不用填写。这些配置只需要在你的xcode工程里配置了就行。这个脚本的重用性就很高。我每个项目，都只需要拷贝这个脚本过去，修改一下工程名字，必要时修改一些命令，就可以了。

[ios打包脚本地址](https://github.com/szuwest/rn_build_ios_sh)

如果你觉得这篇文章有用，请打赏小钱喝杯咖啡^_^
![打赏](https://raw.githubusercontent.com/szuwest/szuwest.github.io/master/images/2018-02-21%20133111.jpg)