Title: 关于Pelican上Disqus无法加载的问题
Date: 2014-12-01
Catogary: 技术
Tag: Pelican, Disqus

最近发现我的博客Disqus加载不出来，以前是好好的。后来根据它的提示去找问题，找了半天也找不出问题所在。然后网上search，发现了一篇跟我类似的问题文章[这里](http://whilgeek.github.io/posts/2014/07/we-were-unable-to-load-disqus/)。我的配置文件里确实有 `RELATIVE_URLS = True`,把它设置为False也没用。检查了N遍shortname，到Disqus设置里各种试各种检查，都还是没有解决。

没有办法，我只好在Disqus上重新创建一个新的site，然后回来配置相应的改了。run起来了，可以了。神了。

不过经过这一次的折腾，我还学到不少别的东西。前段不太懂的人，真是悲剧。以后在这方面好好补补
