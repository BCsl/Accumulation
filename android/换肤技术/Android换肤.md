# Android换肤

## 面对的问题

- 1.不重启Activity的情况下换肤（尤其是ListView和RecycleView的item）
- 2.更低的入侵性（不要自定义View...）

## 两种方式

### Android应用内换肤

#### 局限性

- 有限的皮肤切换
- 耦合程度高 （？）
- 5.0以下不能定义和替换Selector！([相关解析](http://blog.chengyunfeng.com/?p=1013))

#### 实现方式

一般通过系统提供的`setTheme`方法

#### 案例

##### [Colorful](https://github.com/hehonghui/Colorful)

基于Theme的Android动态换肤库，无需重启Activity、无需自定义View

实现细节：1.定义属性，替代原生的属性 2.为这些属性定义多套的Theme 3.通过一个`ViewSetter`对象把View和自定义的属性绑定，注册到`Colorful`对象

在当前Ativity主题发生改变的时候，遍历当前`Colorful`对象注册的`ViewSetter`，设置新的值

```java

/**
 * 修改各个视图绑定的属性
 */
private void makeChange(int themeId) {
  Theme curTheme = mActivity.getTheme();
  for (ViewSetter setter : mElements) {
    setter.setValue(curTheme, themeId);
  }
}
```

获取新的Theme的该属性值

```java
protected int getColor(Theme newTheme) {
  TypedValue typedValue = new TypedValue();
  newTheme.resolveAttribute(mAttrResId, typedValue, true);
  return typedValue.data;
}
```

##### [Knight](https://github.com/zjutkz/Knight/blob/72de2147265dcea8c4ace158722174a7a9ed7ad6/app)

使用APT的思想

### 动态下载资源

#### 好处

- 随时更新
- APK体积减小
- 可以作为增值服务卖钱

## 参考

- [Android换肤技术总结](http://blog.zhaiyifan.cn/2015/09/10/Android%E6%8D%A2%E8%82%A4%E6%8A%80%E6%9C%AF%E6%80%BB%E7%BB%93/)

- [hongyangAndroid/ChangeSkin](https://github.com/hongyangAndroid/ChangeSkin#Demo%E8%BF%90%E8%A1%8C)

- [万能的APT！编译时注解的妙用](http://zjutkz.net/2016/04/07/%E4%B8%87%E8%83%BD%E7%9A%84APT%EF%BC%81%E7%BC%96%E8%AF%91%E6%97%B6%E6%B3%A8%E8%A7%A3%E7%9A%84%E5%A6%99%E7%94%A8/)
