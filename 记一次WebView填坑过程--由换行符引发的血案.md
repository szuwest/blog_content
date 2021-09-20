Date: 2019-07-27
Title: 记一次WebView填坑过程--由换行符引发的血案
Category: 技术
Tags: Android开发 WebView JavaScript

# 记一次WebView填坑过程--由换行符引发的血案
最近使用WebView掉坑了，然后艰难爬坑经历感触很深，写出来大家借鉴一下。
### 需求
我们有个网页需要用到很多js库，这些库比较大，而且基本上是不变的。为了提高性能，将这些网页和JS库放到本地，进行加载。变的数据从服务器获取，然后跟本地的HTML组装后显示。这种需求还是挺普遍的。

### 实现方式
由web的同事调试好HTML文件，他们把所有的HTML，js,css,image和资源一起打包给我们
我们把这些资源放到assets目录下（也可以专门在assets再建一个子目录来放）

采用Android自带的WebView来加载和显示这些网页。有两个方法**loadUrl**和**loadDataWithBaseUrl**方法。
> 1.loadUrl("file:///android_asset/xxxxxx.html")这种方式适合HTML不需要改变的情况，直接加载展示，省时省力

> 2.loadDataWithBaseUrl方式需要先把HTML内容现在到内存，然后再展示。这种方式可以随意修改HTML里的内容，修改好再交由WebView展示，灵活性强。

### 遇到的坑
测试的时候，我们遇到同一个HTML文件，通过loadUrl方式加载展示，没有任何问题。但是通过loadDataWithBaseUrl加载展示，HTML的内容死活展示不出来

### 爬坑过程
* 1. 反复确认loadDataWithBaseURL的用法有没有用对，baseUrl参数没有问题。
mWebView.loadDataWithBaseURL("file:///android_asset/", htmlContent, "text/html", "utf-8", null).
网上搜资料看看别人的用法，还有查看我们工程里其他用了loadDataWithBaseURL方法的地方，都是差不多的。然后我怀疑是htmlContent有问题，是不是读取的时候出错了。
我去看从asset里读取文件的方法.

````
public static String getStringFromAssets(Context context,String fileName) {
        try {
            InputStreamReader inputReader = new InputStreamReader(context.getAssets().open(fileName));
            BufferedReader bufReader = new BufferedReader(inputReader);

            String line;
            StringBuilder stringBuilder = new StringBuilder();
            while ((line = bufReader.readLine()) != null) {
                stringBuilder.append(line);
            }
            return stringBuilder.toString();
            
        } catch (Exception e) {
            e.printStackTrace();
        }
        return "";
    }
    
````
看上去并没有什么问题呀。我搜了下这个方法在我们工程里的使用，发现还有很多使用的地方，他们使用都没问题。我不放弃，到网上再搜一下。全是同样的方法。我断点调试，并且把htmlContent打印出来，好像没有太大的问题呀，不是这里的问题？

* 2.查看我们工程里其他用了loadDataWithBaseURL方法的地方，我的用法跟他们一致。不过最终我发现我用的HTML文件跟他们用的HTML文件有不同的地方：我使用的HTML文件中
更复杂一些，在文件里引用JS资源文件，图片文件，在HTML里本身内嵌了JS代码。我猜测是内嵌JS导致的。我网上搜资料，看看是不是这样的。终于搜到一篇[文章](https://wenwen.sogou.com/z/q436628432.htm)说这个问题，但是这个帖子是很久以前的。下面是它的原话：

> 使用WebView的loadUrl(url),网页中的Javascript运行正常。但是获取url的html内容后，使用loadData或者loadDataWithBaseURL之后，Javascript就不起作用了！google了一圈，似乎这是普遍问题，不知道这里的高手有法子解决不？

貌似验证了我的想法。但是这是很久以前的帖子呀。我想Google改不会那么傻吧。后来我调试时在logcat里看到一些日志。加载网页时打印了类似下面的日志
> [INFO:CONSOLE(1)] "Uncaught SyntaxError: Unexpected identifier "

说明网页在解析JS的时候报错了。
我看了下网页中JavaScript，按道理的这些都是正确的，因为采用loadUrl方式加载没问题，直接在电脑上浏览器打开也没有问题。我把这些js都删掉，只加了一条console.log('1111')的打印语句。再次运行加载，没报错了！！说明内嵌JavaScript代码是可行的。那就是我们原本的JavaScript有问题了？？

我把那部分JS放到一个新的单独的文件里。因为我们的HTML文件本身也引用了其他的独立JS文件。一运行，正常加载显示！JS还是那个JS，换到文件里就没问题了？

所以我那时得出结论：
* 内嵌JavaScript代码在WebView的解析跟独立JS文件不太一样，可能内嵌JavaScript代码的解析要严格一些？

我开始验证这个想法，去修改内嵌JavaScript代码。
我在很多地方加了console打印，看看执行到哪里报错。然后我发现貌似去掉了注释，报错就不一样了。
> Uncaught SyntaxError: Unexpected token。
>
> Uncaught SyntaxError: Unexpected end of input

最后我把注释都去掉，把该加分号的地方都加上。就能正常运行了。
我初步得出以下结论：
> 需要HTML文档里内嵌JavaScript代码的话，要非常注意JavaScript的语法问题。WebView对这部分的JS代码语法要求非常严，经常发生识别不了而报错的情况。
现在发现①非常注意表达式后面要加分号，不能省略分号！！②不能在JS里随便加注释。要仔细检查语法，严格遵循JS标准
 
我把这些结论还加到了我们代码的注释了，以免后人采坑。

到这里问题解决了，我把HTML从asset中加载出来，再HTML字符串中的一些字符串（默认数据）替换成我们从服务器取到的数据（真实用户数据），然后调用loadDataWithBaseURL方法展示。完美。

到这里有些人会说loadUrl方法不是没问题吗，直接用它不就行了，折腾那么多干嘛。确实是如此，用loadUrl方法能实现我的需求，实际上我也试用了这个方法，没问题。
不过采用loadUrl的方式，实现起来不太一样。需要修改JavaScript代码。这些HTML是有web同时提供的，他们都是已经默认数据直接调试好了，一打开就能展示。采用loadUrl的方法，你不能预先修改HTML文件，只能在加载完HTML数据之后，在WebView的webclient回调方法**onPageFinished**里去调用JS方法。

就是说：

* 1.你将JS里的window.onload方法去掉，让他们先不展示
* 2.你新增一个js方法，这方法接收一些参数，然后在这个方法里调用渲染方法（即原来的window.onload里的方法）。
* 3.在你Java代码里，在WebView的webclient回调方法**onPageFinished**里去调用你新增的JS方法，把参数都传入进去。

看到没有，这里要修改的东西还蛮多，还要要求客户端开发人员懂JS代码。

采用loadDataWithBaseURL方式，理想的情况下，不需要改JS代码，直接告诉开发人员默认数据在哪里，把HTML加载进内存的时候，通过String.replace方法，把默认数据替换为真实数据，就可以了。

iOS端就是这么实现的，但是Android这边实现的时候就遇到了坑。。。。
我肯定想两端实现方式保持一致，所有优先采用loadDataWithBaseURL方式。当然，你也可以让web开发人员按你的要求写好，那你也不用改动那么多。。。

### 背后真相
问题解决了，版本也发了。但问题的真相真的是WebView解析内嵌JavaScript比较严格吗？从现象来看，不支持注释，不能省略表达式的分号，这明显是**换行**问题。
所以我想问题可能真的出现在从asset里读取文件地方。看代码就是没有明显错误，一行一行的读取进来，拼接好全部返回。网上查到的所有代码都是这么读的。

那一次性读取整个文件进来呢？所有我找了下资料，将代码改成一次性读取整个文件。

````
public static String getStringFromAssets(ontext context,String fileName) {
        try {
            InputStreamReader inputReader = new InputStreamReader(context.getAssets().open(fileName));
            BufferedReader bufReader = new BufferedReader(inputReader);

            StringWriter out = new StringWriter();
            copy(bufReader, out);
            return out.toString();
        } catch (Exception e) {
            e.printStackTrace();
        }
        return "";
    }

    private static void copy(Reader in, Writer out) throws IOException {
        int c = -1;
        while((c = in.read()) != -1) {
            out.write(c);
        }
    }
    
````
果然什么问题都没有！看logcat打印出来的内容也是跟HTML里的内容一模一样，之前方式打印出来的没有换行并没有太过在意！

那只能说明bufReader.readLine的方法有问题，我网上搜了下，说这个方法会自动去掉换行。原来真相是这个么？？原来WebView一直是个**背锅侠**呀，而且这个锅背了好多年😂😂。

我们来看看时间的真凶readLine方法的声明：

````
/**
     * Reads a line of text.  A line is considered to be terminated by any one
     * of a line feed ('\n'), a carriage return ('\r'), or a carriage return
     * followed immediately by a linefeed.
     *
     * @return     A String containing the contents of the line, not including
     *             any line-termination characters, or null if the end of the
     *             stream has been reached
     *
     * @exception  IOException  If an I/O error occurs
     *
     * @see java.nio.file.Files#readAllLines
     */
    public String readLine() throws IOException {
        return readLine(false);
    }
````
注意看return的声明**not includingany line-termination characters**.
人家这里说得很清楚嘛，吐血了。。。

如果你对Java很熟悉，或者对readLine()方法熟悉，你就不用折腾那么多，就不会掉坑里了。

我在想为什么一直没发现呢，这个方法用了这么久，网上也这么多人用，就没人遇到问题？我猜主要原因是大部分情况下，没有换行符也没有什么影响。我们一般都是把json或HTML文件放assets里，然后读取出来使用。json文件有没有换行根本不影响！我们工程里大部分都是这种用法。如果HTML里没有内嵌JavaScript代码也没有问题！如果真有内嵌JavaScript代码了，也可以通过采用loadUrl方法来绕过，就是我之前的做法那样。

只有正面刚，才会发现这里有个坑。readLine这个方法不好，太有迷惑性了。
还有就是不要随便相信网上拷来的代码！！！这些血的教训。

最后附上修正版的正确的从assets里读取文件的方法。

````
 /**
     * 获取asset的文件，转化成String
     *
     * @param context
     * @param fileName
     * @return
     */
    public static String parseAssetsData(Context context, String fileName) {
        StringBuilder stringBuilder = new StringBuilder();
        InputStreamReader inputStreamReader = null;
        BufferedReader bfReader = null;
        try {
            AssetManager assetManager = context.getAssets();
            inputStreamReader = new InputStreamReader(assetManager.open(fileName));
            bfReader = new BufferedReader(inputStreamReader);
            String line;
            //注意！！ readLine()返回的内容已经把换行符去掉，所以要补上
            while ((line = bfReader.readLine()) != null) {
                stringBuilder.append(line);
                stringBuilder.append("\n");
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            closeStream(inputStreamReader);
            closeStream(bfReader);
        }
        return stringBuilder.toString();
    }

    /**
     * 关闭流
     *
     * @param io
     */
    public static void closeStream(Closeable io) {
        if (io != null) {
            try {
                io.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
````

----------------
如果你觉得这篇文章有用，请打赏小钱喝杯咖啡^_^
![打赏](https://raw.githubusercontent.com/szuwest/szuwest.github.io/master/images/2018-02-21%20133111.jpg)