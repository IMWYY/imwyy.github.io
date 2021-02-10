---
title: splash页启动优化全解析
date: 2018-01-04
tags: Android
---

## 问题

一般没有特殊处理，android启动的时候，会出现白屏或者黑屏的状态，体验很差。究其原因，白屏是app在冷启动的时候，初始化，系统自动用默认的背景色来填充屏幕。这个默认的背景色和你定义的app主题有关。比如如果你的主题继承自`Theme.AppCompat.Light.NoActionBar`，那么启动的时候就是白色。

本章就来解决app启动慢的问题。

## 解决

我所了解到的，一般解决这个问题的方法有两种：

1. 将这个背景色改为透明色。虽然看上去没有白屏的状况，但用户点击app后，原先白屏的那段时间变成了桌面背景，让人感觉启动慢是手机系统的锅，跟app没啥。有点甩锅的意思。

	具体做法是在`style`中添加主题

	```xml
	<style name="AppTheme.SplashTheme" parent="Theme.AppCompat.Light.NoActionBar">
        <item name="android:windowFullscreen">true</item>
        <item name="android:windowIsTranslucent">true</item>
    </style>

	```

	注意这里`windowIsTranslucent`属性就是设置背景色透明的关键代码。然后在`AndroidManifest`中将启动的那个`activity`的`theme`设置为这个主题

	```xml
<activity
            android:name=".ui.base.SplashActivity_"
            android:theme="@style/AppTheme.SplashTheme">

            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
</activity>
	```

2. 第一种方法说白了只是给用户一个假象，“是你手机系统的问题，不是我app的问题”。既然能把背景改成透明的，那为什么不能直接改成我的splash页的背景图片呢，这样白屏的那段时间，先用这个背景图片，当splash页加载好的时候，无缝衔接，但用户看到的就是，”不错这个很快，点了没有任何停顿，立马启动。“。

	具体方法，只要将刚才的那个`AppTheme.SplashTheme`的属性改一改。

	```xml

	```
<style name="AppTheme.SplashTheme" parent="@android:style/Theme.Light.NoTitleBar.Fullscreen">
        <item name="android:windowBackground">@drawable/splash</item>
        <item name="android:windowFullscreen">true</item>
        <item name="android:windowIsTranslucent">true</item>
</style>
	```
	这里的`windowBackground`就是设置背景图片的关键代码，那段冷启动的时间，屏幕就被设置成这张图片。接下来使用一样，在`AndroidManifest`注册使用即可。强力推荐。

## 再优化

搞定了白屏问题，我们再来看看一般app使用splash页启动的方法，可能最一般的做法就是将启动页做成一个Activity，启动完跳转到MainActivity。

`SplashActivity` -> `MainActivity`

但是这样是不是有些浪费，`SplashActivity`运行期间，app应当可以做一些初始化，加载数据的工作，减轻`MainActivity`的负担。所以`SplashActivity`就不需要了，splash可以使用全屏的`dialog`代替，用完直接销毁。如此，app启动更快了。加载完splash， `MainActivity` 的数据页加载好了。

这里提供一个封装好的实现：SplashDialog

```java

public class SplashScreen {

    private Dialog splashDialog;
    private Activity activity;

    public SplashScreen(Activity activity) {
        this.activity = activity;
    }

    /**
     * 显示splash图片
     *
     * @param millis        停留时间 毫秒
     *
     */
    public void show(final int millis) {
        Runnable runnable = new Runnable() {
            public void run() {
                DisplayMetrics metrics = new DisplayMetrics();

                final ImageView root = new ImageView(activity);
                root.setMinimumHeight(metrics.heightPixels);
                root.setMinimumWidth(metrics.widthPixels);
                root.setLayoutParams(new LinearLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT,
                        ViewGroup.LayoutParams.MATCH_PARENT, 0.0F));
                root.setScaleType(ImageView.ScaleType.FIT_XY);

				   //glide加载图片
                GlideApp.with(activity)
                        .load(URL_SPLASH_IMAGE)
                        .placeholder(R.drawable.splash)
                        .error(R.drawable.splash)
                        .diskCacheStrategy(DiskCacheStrategy.NONE)
                        .into(root);

                splashDialog = new Dialog(activity, android.R.style.Theme_NoTitleBar_Fullscreen);

                Window window = splashDialog.getWindow();
                window.setWindowAnimations(R.style.dialog_anim_fade_out);

                splashDialog.setContentView(root);
                splashDialog.setCancelable(false);
                splashDialog.show();

                final Handler handler = new Handler();
                handler.postDelayed(new Runnable() {
                    public void run() {
                        removeSplash();
                    }
                }, millis);
            }
        };
        activity.runOnUiThread(runnable);
    }

    private void removeSplash() {
        if (splashDialog != null && splashDialog.isShowing()) {
            splashDialog.dismiss();
            splashDialog = null;
        }
    }
}
```

这里我使用了glide来加载图片，你也可以更改为自己的加载工具。

在 `MainActivity`使用，`oncreate`之后直接调用，`new SplashDialog(this).show(3000);`
启动页变得只需要一行代码，而且更快，更方便。ps：别忘了给`MainActivity` 加上之前的主题。

## 踩坑

上面提到了第二种方法，设置背景图片。这里遇到了一个坑，如果有类似问题可以参考。

我的 `MainActivity`设置了浸入式的状态栏，但是没有设置透明底部导航栏。导致设置背景图片`windowBackground`的高度是加上了底部导航栏，也就是说导航栏挡住了一部分的背景图片，但是`SplashDialog`加载的图片是忽略底部导航栏的，这样这两张图就会有个错位，启动的时候，会出现图片的位置移动了一下。

解决方法是，在前面的`AppTheme.SplashTheme`中，继承自`@android:style/Theme.Light.NoTitleBar.Fullscreen`，即老版本的theme，不要继承自`Theme.AppCompat.Light.NoActionBar`。原因猜想是，老版本不支持占用导航栏的空间，自然也就不会有被导航栏挡住的情况。




​