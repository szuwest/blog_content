Date: 2021-02-03
Title: Flutter APP加入信鸽推送通知方案
Category: 技术
Tags: Flutter 推送通知 信鸽

# Flutter APP加入信鸽推送通知方案
最近我们团队又采用Flutter开发一款新APP。而且最近由于信鸽（现在又叫腾讯云移动推送）修改按量收费，我们决定在新APP里加入信鸽SDK来做推送通知功能。

我们之前主要主APP里已经使用了信鸽SDK来做推送通知功能，但这次是第一次在Flutter开发的APP上加入推送通知功能。当我们完全加入该功能之后，我觉得可以将这个技术方案分享一下。

其实加入信鸽SDK的方式跟原生APP是一样的，只是在处理推送通知页面跳转方式不一样而已。

先说说信鸽SDK的优势和注意事项。信鸽SDK以前是免费的，在去年才改成收费的，并入了腾讯云。所以他们才改名叫腾讯云移动推送。信鸽SDK还算比较稳定，毕竟是大厂开发的。收费后新增了很多统计功能，有一个重要的数据是通知开启率，还有就是通知点击率，还是很有用的数据。改成收费的的新版SDK有个问题，会跟我们的采用的iOS版的growingIO统计SDK冲突，导致growingIO活跃用户丢失。最后跟growingIO开发人员反馈后，修改growingIO SDK采用的统计方式才正常。

安卓版的信鸽SDK集成了几大厂商的通道，比较容易集成到自己项目里，这也是他的优势。然后为了推送通知能尽量在两端保持一致，这里要专门提一下推送配置。

## 推送配置
我觉得这一点对于推送通知很重要，尽量将两端的配置保持一致，服务器或者运营编辑在推送一条消息的时候，可以较简单的完成，不容易出错。在安卓端，我们采用客户端自定义方式，即自定义intent-filter里的data字段里的scheme，host，path字段。这里scheme，host一般是固定的，每个页面的path不同。而iOS里采用开启附加参数方式。附加参数里有个必须字段就是path字段。这里两端的path字段就是一致，而且还可以根据页面情况增加其他参数。

AndroidManifest.xml配置

````
<activity android:name=".MainActivity">
	<intent-filter>
		<action android:name="android.intent.action.VIEW" />
       <category android:name="android.intent.category.DEFAULT" />
       <category android:name="android.intent.category.BROWSABLE" />
		 <data
             android:host="myhost"
             android:path="/main"
             android:scheme="myscheme" />
	</intent-filter>
</activity>

````

信鸽里的推送配置（采用客户端自定义方式）,还可以往页面传递参数param1

````
myscheme://myhost/main?param1=xx
````

信鸽里的iOS配置，开启高级配置里的**附加参数**

````
path=main
param1=xxx

````

服务器API推送也是按照类似的方式配置参数，跟服务器开发人员约定好了就行了。

采用这种方式，可以专门写个文档，将每个要推送的页面对应的path,需要的额外参数都定义好，服务器开发人员或者运营编辑都可以查阅这个文档来创建推送。非常方便。

那APP收到推送后将如何处理呢？其实也很简单。

安卓端点击通知将会拉起相应配置的activity，在activity的onCreate或者onNewIntent方法里获取到intent，intent.getData获取到Uri,就可以提前里面的参宿parma1了。

iOS端用户点击通知后会打开APP，信鸽已经处理了是冷启动和后台再次唤醒进入APP的情况，都会统一回调到一个接口，在回调接口里获取到一个字典数据。在早期免费版信鸽里，配置发附加参数就在这个字典里。改成收费版后，他们把附加参数转成json字符串放入了一个叫custom的字段里存储。将custom里的josn字符串取出来，再解析成字典就，就可以获取里面的path和param1参数了。然后再根据path来跳转到相应的页面，不再累述。

## Flutter如何处理页面跳转

Flutter跟原生页面不同，但是Flutter页面天生支持path跳转，因为它是基于路由的。
Flutter APP接入信鸽跟原生APP是一样的，问题就在怎么将path传递到Flutter。这很简单，通过EventChannel。

Flutter APP的iOS端几乎和原生一样，变了的是，将path和param1参数通过event channel传递到Flutter层。

但是安卓端就稍微变了下。安卓原生APP有很多activity，每个activity都配置一个独立的path。到了Flutter APP只需一个MainActivity就可以了。

安卓的配置就变成了

````
myscheme://myhost/main?path=page2&param1=xx
````
这里myscheme://myhost/main就是不需要变的了（除非你还有的activity存在），要跳转的页面通过path参数来决定。native端MainActivity收到通知后需要将path等参宿传递到Flutter层

native端需要注意点：
这里需要特别注意当native端收到推送通知回调时，Flutter的event channel可能没有准备好，需要先将数据用一个变量存起来，等event channel准备好之后，再传递到Flutter，然后将数据清除。

这里特别说明一下，两端都是给Flutter层传递一个map数据：

````
{path:page2,param1:xxx}
````

### Flutter层处理
Flutter层处理也有一定技巧。在main函数执行的之后，就要开始注册监听event channel。
当event channel收到数据通知的时候，拿到数据map。然后用map里的path来做跳转。

````
//接收native传递来的参数obj
  void _onReceivePush(Object obj) {
    WGLog.log('_onReceivePush: $obj');
    if (obj != null && obj is Map) {
      Map<String, dynamic> pushData = Map<String, dynamic>.from(obj);
      String path = pushData['path'];
      if (path != null && path.length > 0) {
        if (!path.startsWith('/')) {
          path = '/' + path;
        }
        if (UserManager.instance.isLogin()) {
          _handlePush(path, pushData);
        } else {//未登录的情况，先缓存数据登录后再使用
          gPushData = obj;
          WGToast.showToast("请先登录");
        }
      } else {
        WGLog.log("path $path error");
      }
    }
  }
  
  void _handlePush(String path, Map<String, dynamic> args) {
    if (path == "/home" || path == "/main") {
      //首页
    } else {
      Navigator.pushNamed(context, path, arguments: args);
    }
  }
````

最重要的是 **Navigator.pushNamed(context, path, arguments: args)**这句代码。这args就是从native层传过来的map，里面可能包含要跳转页面的参数。

因为Flutter是基于路由的，所以路由注册很重要，还有相关页面如何获取或者处理传递过来的参数呢？
在MaterialApp的构造函数里有几个很重要的参数，routes，onGenerateRoute，onUnknownRoute。我们主要设置这几个参数来管理整个Flutter页面的路由。

````
class MyApp extends StatelessWidget {
  // This widget is the root of your application.
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'xxxx',
       //注册页面
      routes: AppRouter.routes,
      onGenerateRoute: AppRouter.generateRoute,
      onUnknownRoute: AppRouter.unknownRoute,
      home: MainPage(title: 'xxx'),
    );
  }
}
````
routes表示注册的路由表。当Navigator push一个命名路由的时候，会先到这个路由表里找对应path的路由，如果表里没有，就会进入onGenerateRoute，你可以在这里即时生成路由页面来处理，也可以不处理。如果onGenerateRoute没有处理，就会进入onUnknownRoute定义的路由页面。

我是这样配置的：
不需要额外参数的页面，直接注册在routes路由表里，需要传递额外参数才能打开的页面，放到onGenerateRoute动态生成（因为这里可以直接获取到外部传入的args），最后准备一个UnknownPage.而且这个UnknownPage很重要。设想你的APP已经发布了1.0，2.0，3.0版本。现在向全部用户发送推送消息，这个消息对应的页面是2.0版本才加入的，那么还在用1.0版本的用户收到这个通知后会怎么样。他会进入这个UnknownPage，你可以在这个页面提示他的APP版本太低，给个升级按钮，点击后可以跳转到商店去升级APP。

我把路由处理逻辑都放在一个这门的类来处理AppRouter。

````
class AppRouter {

  ///注册路由。routes比generateRoute的优先级高，先在routes找，找不到再到generateRoute找，最后找不到的话就进入unknownRoute
  ///
  static final Map<String, WidgetBuilder> routes = {

    '/home': (BuildContext context) => HomePage(),
    '/bindSuc': (BuildContext context) => BindSucPage(),
    '/orderList': (BuildContext context) => OrderListPage(),// 我的订单列表
 };

  //要注意页面参数的配置
  static final RouteFactory generateRoute = (settings) {
    WGLog.log("settings ${settings.toString()} ");
    Map<String, dynamic> params = settings.arguments as Map<String, dynamic>;
    if (settings.name == '/project') {
      String pid = params['id'];
      if (pid != null) {
        var project = HomeProject(id: pid);
        return MaterialPageRoute(
            builder: (ctx) {
              return ProjectDetailPage(project: project,);
            }
        );
      }
    }
    else if (settings.name == '/applyDetail') {
      String _id = params['id'];
      if (_id != null) {
        var model = ApplyInfoModel(apply_id: _id);
        return MaterialPageRoute(
            builder: (ctx) {
              return ApplyOrderDetailPage(model: model,);
            }
        );
      }
    }
	//....
 
    return null;
  };

  /// 未知路由页面。一般是APP版本太低，推送过来的页面未定义
  static final RouteFactory unknownRoute = (settings) {
    return MaterialPageRoute(
        builder: (ctx) {
          return UnknownPage();
        }
    );
  };
}
````

我认为generateRoute里的处理非常奇妙。设想有一个页面是一定要传一个id才可以正常浏览的。一般我们都会把这个页面的构造函数定义一个参数，由外部初始化的时候传入。如果将这个页面也支持注册到路由表routes里，那么你要获取传递过去的参数就有点麻烦了。你可能需要在该页面里面通过 **var args=ModalRoute.of(context).settings.arguments**方式来获取。这样不直观。

所以我认为这样处理Flutter的跳转是比较完美的方式。

----------------
如果你觉得这篇文章有用，请打赏小钱喝杯咖啡^_^
![打赏](https://raw.githubusercontent.com/szuwest/szuwest.github.io/master/images/2018-02-21%20133111.jpg)
