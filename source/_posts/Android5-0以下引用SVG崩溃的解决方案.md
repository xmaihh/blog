---
title: Android5.0以下引用SVG崩溃的解决方案
date: 2022-01-19 10:44:34
categories: Android Framework
tags: [Android]
toc: true
description: 地道に一歩一歩前進することは大事です。
---

# 1、xml里面引用

在根布局加上：`xmlns:app=http://schemas.android.com/apk/res-auto`

然后：

`app:srcCompat="@drawable/ic_backspace" />`

# 2、在代码里面设置

```java
ImageView img.setDrawableR.drawable.ic_backspace);

ImageView img.setImageResource(R.drawable.ic_backspace);
```

# 3、自定义属性里面包含svg图片

需要使用`TintedTypedArray`来解析

# 4、不能使用android:background来引用svg图片

如果要设置background可以使用`方式2`通过代码来设置

# 5、Glide不支持直接引用svg图片

# 6、Android 5.0以下，如果不继承AppCompatActivity，获取svg图片方法

```
VectorEnabledTintResources resources =

new VectorEnabledTintResources(getApplicationContext(), getResources());

Drawable drawable = resources.getDrawable(R.drawable.ic_3s);
```

# 7、selector里可以用svg图片

# 8、渐变色的崩溃

崩溃范围：Android API 24以下，使用drawableLeft等。

原因：SVG的渐变色只能用aapt标签包含，需要放在`drawable-v24`包中，如果依然放在drawable包中，没有问题。但高版本gradle使用aapt2进行资源打包，在低于24的手机上会因为找不到资源崩溃。但之前使用aapt进行资源打包的项目没有问题，因为此时生成了对应的png图。

特别说明：设置`background`、`srcCompat`不会有问题，但需要注意是否显示正常。

布局无法使用 `background` 和 `srcCompat` 代替时，那么可以使用以下代码动态获取 `drawable` 设置：

`AppCompatResources.getDrawable(context, resId)`
