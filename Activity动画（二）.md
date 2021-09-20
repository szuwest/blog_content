Date: 2014-11-12
Title: Activity动画（二）
Catogary: Android
Tags: Android 技术 动画

#Activity动画（二）
如果要统一所有的Activity的打开可关闭动画，可以有一个很简便的方法：就是设置Application的theme。在Application的theme里有一个Android:windowAnimationStyle的属性，可以定义了一个Activity的打开和关闭动画。例如下面的例子就自定义了一个theme。

----

````
<style name="ThemeLightNoTitle" mce_bogus="1" parent="@android:style/Theme.Light.NoTitleBar">
        <item name="android:windowAnimationStyle">@style/AnimationActivity</item>
    </style>
    
    <style name="AnimationActivity" mce_bogus="1" parent="@android:style/Animation.Activity">
        <item name="android:activityOpenEnterAnimation">@anim/translate_between_interface_right_in</item>
        <item name="android:activityOpenExitAnimation">@anim/translate_between_interface_left_out</item>
        <item name="android:activityCloseEnterAnimation">@anim/translate_between_interface_left_in</item>
        <item name="android:activityCloseExitAnimation">@anim/translate_between_interface_right_out</item>
        <item name="android:taskOpenEnterAnimation">@anim/translate_between_interface_right_in</item>
        <item name="android:taskOpenExitAnimation">@anim/translate_between_interface_left_out</item>
        <item name="android:taskCloseEnterAnimation">@anim/translate_between_interface_left_in</item>
        <item name="android:taskCloseExitAnimation">@anim/translate_between_interface_right_out</item>
    </style>
 ````
  
 将这个theme运用到Application的theme里，所以的Activity就会按照你定义的动画来交互了。
 
 不过，现实的需求往往是更复杂的动画或者交互，不可能所有的界面的交互动画都一致的。有些页面需要特别定制，而且可能很复杂。例如我们要实现类似于iOS的NavigationViewController的页面右滑关闭页面的交互动画，或者要实现点击一张图片缩略图，然后它渐渐放大，点击关闭它又慢慢缩回原处的动画，按照之前介绍的方法，都无法实现。


-----
在iOS自带相册APP里，点击一个缩略图，缩略图渐渐放大进入大图查看。这个交互基本成了一个相册必备的标准交互。Android上的相册很多也用了这种动画交互。如果缩略图的查看和大图查看都是在同一个Activity中完成的话，还是可以比较容易实现的。但是如果缩略的查看和大图的查看分处在不同的Activity中的话，那这就不是很好实现了。这里我准备讲讲可以怎么实现这个功能。而且这个方法可以推广出去，很多类型的Activity切换动画都可以按照这种方式实现。

现在假设查看缩略图的Activity是A，查看大图的Activity是B。
> * 一 设置B的的theme为透明的transparent。
> * 二 在A中，启动B前，把一些必要的信息打包成Bundle传给B，启动B后把默认的启动动画覆盖,overridePendingTransition(0, 0)。例如把被点击的ImageView的在屏幕中的坐标，还有该ImageView的width和height，还有该ImageView所显示的图片资源id，或者URL，都存入Bundle中，讲该Bundle放入Intent中，启动B时传给B。
> * 三 在B的onCreate方法中，把Bundle中的信息取出来，并且对你想要做动画的ImageView中，获取ViewTreeObserver实例，加入一个OnPreDrawListener，在OnPreDrawListener的onPreDraw方法中，对ImageView做放大动画，初始位置就是从Bundle中获取，最终位置根据要显示的图片的大小来定。
> * 四 关闭B Activity时，对ImageView做一个缩小的动画，初始位置就是它现在的位置，最终位置是最开始从Bundle中获取的那个位置。覆盖Activity的finish方法，同样要将Activity的默认动画覆盖掉overridePendingTransition(0, 0)。

我们看一下具体怎么实现，有什么要注意的。
> 一· 设置B的的theme为透明的transparent

	<style name="Transparent">
        <item name="android:windowNoTitle">true</item>
        <item name="android:windowIsTranslucent">true</item>
        <item name="android:windowBackground">@android:color/transparent</item>
    </style>
    
    <activity android:name="com.example.android.activityanim.PictureDetailsActivity"
                android:label="@string/subactivity_name"
                android:theme="@style/Transparent" >
        </activity>
这里主要是将B Activity的window和它的背景设置成透明的。所有的Activity都有一个默认的样式，默认样式不是透明的。设置成透明的话，在启动B Activity时它的背景是透明的，并且调用overridePendingTransition(0, 0)覆盖掉默认动画，你不会觉察到已经从A进入到B界面。

> * 二. 在A中，启动B前，把一些必要的信息打包成Bundle传给B，启动B后把默认的启动动画覆盖,overridePendingTransition(0, 0)。

	// Interesting data to pass across are the thumbnail size/location, the
            // resourceId of the source bitmap, the picture description, and the
            // orientation (to avoid returning back to an obsolete configuration if
            // the device rotates again in the meantime)
            int[] screenLocation = new int[2];
            v.getLocationOnScreen(screenLocation);
            PictureData info = mPicturesData.get(v);
            Intent subActivity = new Intent(ActivityAnimations.this,
                    PictureDetailsActivity.class);
            int orientation = getResources().getConfiguration().orientation;
            subActivity.
                    putExtra(PACKAGE + ".orientation", orientation).
                    putExtra(PACKAGE + ".resourceId", info.resourceId).
                    putExtra(PACKAGE + ".left", screenLocation[0]).
                    putExtra(PACKAGE + ".top", screenLocation[1]).
                    putExtra(PACKAGE + ".width", v.getWidth()).
                    putExtra(PACKAGE + ".height", v.getHeight()).
                    
            startActivity(subActivity);
            
            // Override transitions: we don't want the normal window animation in addition
            // to our custom one
            overridePendingTransition(0, 0);
 
 这里有个关键点，获取被点击的View的在屏幕上的位置是通过getLocationOnScreen()方法。这个位置在B中很重要。另外还有宽高也是必须信息，被点击的图片resId或者URL等其他信息。

> 二. 在B的onCreate方法中，把Bundle中的信息取出来，并且对你想要做动画的ImageView中，获取ViewTreeObserver实例，加入一个OnPreDrawListener，在OnPreDrawListener的onPreDraw方法中，对ImageView做放大动画，初始位置就是从Bundle中获取，最终位置根据要显示的图片的大小来定。

因为第一步，在用户觉察不到的情况下已进入Activity B,接下来只要在B的视图出现的时候，以一个缩略图放大的动画效果出现就行了。用户看到的效果是点击了A的一个缩略图，缩略图逐渐放大的过程，用户很容易认为这是在A中完成的，实际上这所有的动画都是在B中完成的。看看B中相关的代码:

	// Retrieve the data we need for the picture/description to display and
        // the thumbnail to animate it from
        Bundle bundle = getIntent().getExtras();
        Bitmap bitmap = BitmapUtils.getBitmap(getResources(),
                bundle.getInt(PACKAGE_NAME + ".resourceId"));
        String description = bundle.getString(PACKAGE_NAME + ".description");
        final int thumbnailTop = bundle.getInt(PACKAGE_NAME + ".top");
        final int thumbnailLeft = bundle.getInt(PACKAGE_NAME + ".left");
        final int thumbnailWidth = bundle.getInt(PACKAGE_NAME + ".width");
        final int thumbnailHeight = bundle.getInt(PACKAGE_NAME + ".height");
        mOriginalOrientation = bundle.getInt(PACKAGE_NAME + ".orientation");
        
        mBitmapDrawable = new BitmapDrawable(getResources(), bitmap);
        mImageView.setImageDrawable(mBitmapDrawable);
        
        ViewTreeObserver observer = mImageView.getViewTreeObserver();
            observer.addOnPreDrawListener(new ViewTreeObserver.OnPreDrawListener() {
                
                @Override
                public boolean onPreDraw() {
                    mImageView.getViewTreeObserver().removeOnPreDrawListener(this);

                    // Figure out where the thumbnail and full size versions are, relative
                    // to the screen and each other
                    int[] screenLocation = new int[2];
                    mImageView.getLocationOnScreen(screenLocation);
                    mLeftDelta = thumbnailLeft - screenLocation[0];
                    mTopDelta = thumbnailTop - screenLocation[1];
                    
                    // Scale factors to make the large version the same size as the thumbnail
                    mWidthScale = (float) thumbnailWidth / mImageView.getWidth();
                    mHeightScale = (float) thumbnailHeight / mImageView.getHeight();
    
                    runEnterAnimation();
                    
                    return true;
                }
            });

ViewTreeObserver是View树监听类，可以监听View的一些变化。其中ViewTreeObserver.OnPreDrawListener内部类，是当视图要被绘制时会被回调的类。我们只要在创建一个ViewTreeObserver.OnPreDrawListener实例，传给ViewTreeObserver，就可以在View被绘制时回调onPreDraw方法，我们所要做的就是在这个方法中开始动画。这里注意在动画开始前已经把要显示的图片设置上去了。
runEnterAnimation方法：

	// Set starting values for properties we're going to animate. These
        // values scale and position the full size version down to the thumbnail
        // size/location, from which we'll animate it back up
        mImageView.setPivotX(0);
        mImageView.setPivotY(0);
        mImageView.setScaleX(mWidthScale);
        mImageView.setScaleY(mHeightScale);
        mImageView.setTranslationX(mLeftDelta);
        mImageView.setTranslationY(mTopDelta);
        
        // Animate scale and translation to go from thumbnail to full size
        mImageView.animate().setDuration(duration).
                scaleX(1).scaleY(1).
                translationX(0).translationY(0).
                setInterpolator(sDecelerator));
这段代码先将ImageView从大图缩小到我们点击的那个缩略图的大小和位置，然后开始放大动画到它大图显示的样子。这样就实现了我们想要的点击缩略图放大到大图的效果。

> * 关闭B Activity时，对ImageView做一个缩小的动画，初始位置就是它现在的位置，最终位置是最开始从Bundle中获取的那个位置。覆盖Activity的finish方法，同样要将Activity的默认动画覆盖掉overridePendingTransition(0, 0)

	@Override
    public void finish() {
        super.finish();
        
        // override transitions to skip the standard window animations
        overridePendingTransition(0, 0);
    }
    
    /**
     * Overriding this method allows us to run our exit animation first, then exiting
     * the activity when it is complete.
     */
    @Override
    public void onBackPressed() {
        runExitAnimation(new Runnable() {
            public void run() {
                // *Now* go ahead and exit the activity
                finish();
            }
        });
    }
    
    private void runExitAnimation(Runnable endAction) {
    	// Animate image back to thumbnail size/location
        mImageView.animate().setDuration(duration).
                   scaleX(mWidthScale).scaleY(mHeightScale).
                   translationX(mLeftDelta).translationY(mTopDelta).
                                withEndAction(endAction);
    }
    
当用户关闭界面的时候，先执行ImageView的缩小动画，缩小到原来缩略图的位置。当动画结束后，finish掉Activity并覆盖掉默认的Activity动画。

**总结：第一步先要把Activity的theme变成透明。第覆盖掉Activity的启动或者关闭的默认动画效果。第三步如果要做进入动画，则可以实现ViewTreeObserver.OnPreDrawListener接口，在onPreDraw方法中做动画。如果是做退出动画，可以先对视图做动画，在动画结束的时候调用finish方法。**

[demo代码](https://github.com/szuwest/ActivityAnimations).该demo是dev-bytes上的demo代码。

##扩展
如果B要支持左右滑动查看大图，然后在查看某已图片时点击返回键，图片要缩小至该图缩略图的位置上，这该怎么做？
这种需求改变对A变化不大，但是对B的改变还是很大的，首先B要支持滑屏，一般来说要用到ViewPage或类似的控件。另外还有滑动时B如何获取数据，返回时如何获取当前大图对应的缩略图在A中的位置。我想实现方法应该有多种，我能想到的一种比较丑陋但是又比较简单的做法是，定义一个接口，声明了一些获取数据和缩略图位置等方法。然后A实现这个接口，传给B作为一个静态变量保存起来，这样A与B就可以通信了。这样只要B左右滑动的时候，A也保持一致的上下滑动，在B要关闭的时候，先获取到相应缩略图的问题，就能做动画了。