Date: 2014-11-03
Title: Android Activity动画
Category: Android
Tags: 动画

#Android 动画(一)
##3.0之前的动画
Android动画一直是Android的痛点。Android的动画系统跟iOS系统的动画系统比起来真心差很多，特别是2.3之前。在2.3之前，Android的动画分两种，帧动画和逐渐动画（Tween）.Tween动画包括alpha动画，translate动画，scale动画，rotate动画。这几种动画可以随意组合，产生更复杂的动画。但是我们想要做iOS的frame那样简单的动画，却是没有直接的方法。

------
##3.0之后的动画
在Android3.0版本，终于加入了更多的动画支持，API也更加友好。加入我property动画和value动画，大大增加了动画的灵活性和可定制性。可以通过ObjectAnimator很方便的写出动画代码。像iOS的frame动画，可以通过value动画来实现。还有因为property特性，可以实现一些特别的动画，并且不局限与view，理论上任何object都可以实现property动画。
####nineOldAnimation
这个是向下兼容的Android动画库，使得在3.0之前的系统也可以实现property和value动画。不过这个库早已不维护了，我之前使用发现了一些bug，不过这影响不大。如果要兼容2.3和一下系统，这个是很不错的库。
####Android4.4和5.0
Android4.0之后陆陆续续加了一些动画支持，例如4.1加了对activity的动画支持，4.4加了一些过度动画支持，而最新的5.0加了一个vectorDrawable，可以实现一些高级的向量动画。由于这些API需要较高版本支持，我们开发还不能完全用到，这个我也在研究当中。

接下来我要讨论的时Activity动画，这个在我们开发中用得比较多，并且做得好的话，可以为应用增添不少分。

------
#Activity动画
在Android2.0之后，Activity中加入了一个重要的API  `overridePendingTransition`。这个方法在`startActivity`方法之后调用的话，可以定义新打开的activity进入动画。例如你想实现像iOS的navigationController的从右往左的push动画，或者ModelViewController的从下往上弹出页面的动画，就可以通过这种方法。


```
Intent intent = new Intent();                                   intent.setAction(BuyDiamondsActivity.DIAMONDLISTACTION);
intent.setClass(PhotoAlbumActivity.this, BuyDiamondsActivity.class);
startActivity(intent);
overridePendingTransition(R.anim.push_left_in, R.anim.push_left_out);
```


push_left_in.xml    
                             
```
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android">
	<translate android:fromXDelta="100%p" android:toXDelta="0" android:duration="300"/>
	<alpha android:fromAlpha="1.0" android:toAlpha="1.0" android:duration="300" />
</set>
```


push_left_out.xml

```
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android">
	<translate android:fromXDelta="0" android:toXDelta="-40%p" android:duration="300"/>
	<alpha android:fromAlpha="1.0" android:toAlpha="0.6" android:duration="300" />
</set>
```

同样，我们也可以在Activity的finish方法之后，调用`overridePendingTransition`来自定义Activity退出动画。这里还有个小tips,如果你要实现A跳转到B，你又没办法改变或定制A如何跳转到B，你可以在B的onCreate方法里，调用父类的onCreate方法之前调用`overridePendingTransition`也是有效的。

`overridePendingTransition`方法有个缺陷，它只支持XML文件声明的动画，在Android4.1之前（API16）我暂时还没有发现有别的方法可以用代码定义Activity跳转动画。

##ActivityOptions
估计为了弥补`overridePendingTransition`方法的不足，Google在Android4.1中加入一个新类ActivityOptions和一个新startActivity(Intent intent, Bundle options)方法。新的startActivity方法可以传入一个Bundle参数，这个Bundle可以包含了一些Activity的动画。这个Bundle数据从何而来，怎么创建，需要包含那些信息，这个你可以不用太过关心，ActivityOptions就是为产生这个Bundle而生的。来看看几个例子：


```

// Custom animations allow us to do things like slide the next activity in as we
// slide this activity out
translateButton.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        // Using the AnimatedSubActivity also allows us to animate exiting that
        // activity - see that activity for details
        Intent subActivity = new Intent(WindowAnimations.this,
                AnimatedSubActivity.class);
 		 // The enter/exit animations for the two activities are specified by xml resources
        Bundle translateBundle =
                ActivityOptions.makeCustomAnimation(WindowAnimations.this,
                R.anim.slide_in_left, R.anim.slide_out_left).toBundle();
        startActivity(subActivity, translateBundle);
    }
});
        
```
通过ActivityOptions.makeCustomAnimation的静态方法可以产生ActivityOptions对象，ActivityOptions有个toBundle的实例方法方便的将你定义的XML动画转化为Bundle数据。

```
// Starting in Jellybean, you can also provide an animation that scales up the new
// activity from a given bitmap, cross-fading between the starting and ending
// representations. Here, we scale up from a thumbnail image of the final sub-activity
thumbnail.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        BitmapDrawable drawable = (BitmapDrawable) thumbnail.getDrawable();
        Bitmap bm = drawable.getBitmap();
        Intent subActivity = new Intent(WindowAnimations.this, AnimatedSubActivity.class);
        Bundle scaleBundle = ActivityOptions.makeThumbnailScaleUpAnimation(
                thumbnail, bm, 0, 0).toBundle();
        startActivity(subActivity, scaleBundle);
    }
});
```
ActivityOptions.makeThumbnailScaleUpAnimation提供了更高级的功能：将一个缩略图方法的Activity动画。这个有时候挺有用的。ActivityOptions还提供了其他的一些方法，感兴趣的话可以参考它的API。
更多请参考[ActivityOptions API](http://developer.android.com/reference/android/app/ActivityOptions.html) 和 [Google官方教程:Defining Custom Animations](https://developer.android.com/training/material/animations.html)

下一节我会讲更高级更复杂的一些自定义Activity动画。