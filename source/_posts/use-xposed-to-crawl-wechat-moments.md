---
title: 用Xposed框架抓取微信朋友圈数据
date: 2016-01-22 23:38:02
tags: [android, hack]
---

因微信朋友圈为私有协议，从抓包上分析朋友圈数据几乎不可能，目前也尚未找到开源的抓取朋友圈的脚本。博主于是尝试通过使用安卓下的Xposed框架实现从微信安卓版上抓取朋友圈数据。
本文针对微信版本6.3.8。
[GitHub仓库](https://github.com/Chion82/WeChatMomentExport)

主要思路
------
从UI获取文本信息是最为简单的方法，于是应该优先逆向UI代码部分。

逆向微信apk
----------
首先解包微信apk，用dex2jar反编译classes.dex，然后用JD-GUI查看jar源码。
当然，能看到的源码都是经过高度混淆的。但是，继承自安卓重要组件（如Activity、Service等）的类名无法被混淆，于是还是能从中看到点东西。

1. 首先定位到微信APP package。我们知道这个是`com.tencent.mm`。
2. 在`com.tencent.mm`中，我们找到一个`ui`包，有点意思。
3. 展开`com.tencent.mm.ui`，发现多个未被混淆的类，其中发现`MMBaseActivity`直接继承自`Activity`，`MMFragmentActivity`继承自`ActionBarActivity`，`MMActivity`继承自`MMFragmentActivity`：
```java
public class MMFragmentActivity
  extends ActionBarActivity
  implements SwipeBackLayout.a, b.a {
    ...
}
public abstract class MMActivity
  extends MMFragmentActivity {
    ...
}
public class MMBaseActivity
  extends Activity {
    ...
}
```
现在需要找出朋友圈的Activity，为此要用Xposed hook`MMActivity`。

创建一个Xposed模块
----------------
参考[\[TUTORIAL\]Xposed module devlopment](http://forum.xda-developers.com/showthread.php?t=2709324)，创建一个Xposed项目。
简单Xposed模块的基本思想是：hook某个APP中的某个方法，从而达到读写数据的目的。
小编尝试hook`com.tencent.mm.ui.MMActivity.setContentView`这个方法，并打印出这个Activity下的全部TextView内容。那么首先需要遍历这个Activity下的所有TextView，遍历ViewGroup的方法参考了SO的以下代码：
```java
private void getAllTextViews(final View v) {
   if (v instanceof ViewGroup) {
       ViewGroup vg = (ViewGroup) v;
       for (int i = 0; i < vg.getChildCount(); i++) {
           View child = vg.getChildAt(i);
           getAllTextViews(child);
       }
   } else if (v instanceof TextView ) {
       dealWithTextView((TextView)v); //dealWithTextView(TextView tv)方法：打印TextView中的显示文本
   }
}
```
Hook`MMActivity.setContentView`的关键代码如下：
```java
findAndHookMethod("com.tencent.mm.ui.MMActivity", lpparam.classLoader, "setContentView", View.class, new XC_MethodHook() {
  ...
});
```
在findAndHookMethod方法中，第一个参数为完整类名，第三个参数为需要hook的方法名，其后若干个参数分别对应该方法的各形参类型。在这里，`Activity.setContentView(View view)`方法只有一个类型为`View`的形参，因此传入一个`View.class`。
现在，期望的结果是运行时可以从Log中读取到每个Activity中的所有的TextView的显示内容。
**但是，因为View中的数据并不一定在`setContentView()`时就加载完毕，因此小编的实验结果是，log中啥都没有。**

意外的收获
--------
当切换到朋友圈页面时，Xposed模块报了一个异常，异常源从`com.tencent.mm.plugin.sns.ui.SnsTimeLineUI`这个类捕捉到。从类名上看，这个很有可能是朋友圈首页的UI类。展开这个类，发现更多有趣的东西：
这个类下有个子类`a`(被混淆过的类名)，该子类下有个名为`gyO`的`ListView`类的实例。我们知道，`ListView`是显示列表类的UI组件，有可能就是用来展示朋友圈的列表。

顺藤摸瓜
-------
那么，我们先要获得一个`SnsTimeLineUI.a.gyO`的实例。但是在这之前，要先获得一个`com.tencent.mm.plugin.sns.ui.SnsTimeLineUI.a`的实例。继续搜索，发现`com.tencent.mm.plugin.sns.ui.SnsTimeLineUI`有一个名为`gLZ`的`SnsTimeLineUI.a`实例，那么我们先取得这个实例。

经过测试，`com.tencent.mm.plugin.sns.ui.SnsTimeLineUI.a(boolean, boolean, String, boolean)`这个方法在每次初始化微信界面的时候都会被调用。因此我们将hook这个方法，并从中取得`gLZ`。
```java
findAndHookMethod("com.tencent.mm.plugin.sns.ui.SnsTimeLineUI", lpparam.classLoader, "a", boolean.class, boolean.class, String.class, boolean.class, new XC_MethodHook() {
    @Override
    protected void afterHookedMethod(MethodHookParam param) throws Throwable {
        XposedBridge.log("Hooked. ");
        Object currentObject = param.thisObject;
        for (Field field : currentObject.getClass().getDeclaredFields()) { //遍历类成员
            field.setAccessible(true);
            Object value = field.get(currentObject);
            if (field.getName().equals("gLZ")) {
                XposedBridge.log("Child A found.");
                childA = value;
                //这里获得了gLZ
                ...
            }
        }
    }
});
```

现在取得了`SnsTimeLineUI.a`的一个实例`gLZ`，需要取得这个类下的`ListView`类型的`gyO`属性。
```java
private void dealWithA() throws Throwable{
    if (childA == null) {
        return;
    }
    for (Field field : childA.getClass().getDeclaredFields()) { //遍历属性
        field.setAccessible(true);
        Object value = field.get(childA);
        if (field.getName().equals("gyO")) {  //取得了gyO
            ViewGroup vg = (ListView)value;
            for (int i = 0; i < vg.getChildCount(); i++) {  //遍历这个ListView的每一个子View
                ...
                View child = vg.getChildAt(i);
                getAllTextViews(child); //这里调用上文的getAllTextViews()方法，每一个子View里的所有TextView的文本
                ...
            }
        }
    }
}
```
现在已经可以将朋友圈页面中的全部文字信息打印出来了。我们需要根据TextView的子类名判断这些文字是朋友圈内容、好友昵称、点赞或评论等。
```java
private void dealWithTextView(TextView v) {
        String className = v.getClass().getName();
        String text = ((TextView)v).getText().toString().trim().replaceAll("\n", " ");
        if (!v.isShown())
            return;
        if (text.equals(""))
            return;
        if (className.equals("com.tencent.mm.plugin.sns.ui.AsyncTextView")) {
            //好友昵称
            ...
        }
        else if (className.equals("com.tencent.mm.plugin.sns.ui.SnsTextView")) {
            //朋友圈文字内容
            ...
        }
        else if (className.equals("com.tencent.mm.plugin.sns.ui.MaskTextView")) {
            if (!text.contains(":")) {
                //点赞
                ...
            } else {
                //评论
                ...
            }
        }
    }
```
自此，我们已经从微信APP里取得了朋友圈数据。当然，这部分抓取代码需要定时执行。因为从`ListView`中抓到的数据只有当前显示在屏幕上的可见部分，为此需要每隔很短一段时间再次执行，让用户在下滑加载的过程中抓取更多数据。
剩下的就是数据分类处理和格式化输出到文件，受本文篇幅所限不再赘述，详细实现可参考作者GitHub上的源码。
