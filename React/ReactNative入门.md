# ReactNative 入门

主要说一些和原生开发不一样的内容

## Props 和 State

这个和 `React` 一样

## 样式定义

推荐使用 `JS` 来定义样式，**命名遵循 `CSS` 的命名**，但改成用驼峰命名法，所有核心的组件支持 `style` 属性，可以接受 `JS` 对象，可以是数组，且数组后的优先级比较高

**怎样使用原生的资源？字符串、颜色值等？国际化？**

## 宽高

`RN` 中尺寸无单位，都使用 `dp`

```javascript
 <View style={{width: 50, height: 50, backgroundColor: 'powderblue'}} />
```

`flex` 属性可以使其在可利用的空间中动态地扩张或收缩，有点像 `LinearLayout` 的 `weight` 属性

## Flexbox 布局

[React Native----flex（弹性布局）](https://juejin.im/post/596399c4f265da6c40736b26?utm_source=gold_browser_extension)

`flexbox` 规则来指定某个组件的子元素的布局

- flexDirection

  如名字，控制布局方向，支持['row'|'column']

  ```javascript
  <View style={{flex: 1, flexDirection: 'row'}}>
  ```

- alignItems

  决定其子元素沿着次轴的排列方式，`flex-start`、`center`、`flex-end` 以及 `stretch`

- justifyContent

  决定其子元素沿着主轴的排列方式，可选`flex-start`、`center`、`flex-end`、`space-around` 以及 `space-between`

## 处理长列表

- FlatList

  `FlatList` 组件用于显示一个垂直的滚动列表，其中的元素之间结构近似而仅数据不同，优先渲染屏幕上可见的元素，类似原生的 `ListView`，且比 `RN` 提供的 `ListView` 有更好的性能 [React Native 为何要新设计一个 ListView （FlatList ）？](https://www.zhihu.com/question/55518679)

  > 记得为列表元素加上 `key`

  两个必要的属性 `data` 和 `renderItem`，前一个是数据源，后一个是 `item` 渲染，类似 `Adapter#getItemView`

- SectionList

  带分类的 `ListView`，必要的属性 `sections`、`renderItem` 和 `renderSectionHeader`，第一个是一个数组类型的数据源

## 网络

### 使用 Fetch

推荐使用，使用 `Promise` 模式，返回一个 `Promise` 对象

### 使用其他网络库

`React Native`中已经内置了`XMLHttpRequest` API(也就是俗称的 `ajax` )

## 图片资源

### 静态图片资源

```javascript
<Image source={require('./my-icon.png')} />
```

> `require` 中的图片名字必须是一个静态字符串，不能是变量，因为 `require` 是在编译时期执行，而非运行时期执行！

```javascript
// 错误
var icon = this.props.active ? 'my-icon-active' : 'my-icon-inactive';
<Image source={require('./' + icon + '.png')} />

// 正确
var icon = this.props.active ? require('./my-icon-active.png') : require('./my-icon-inactive.png');
<Image source={icon} />
```

图片的适配符，"xxx.ios.xxx" 和 "xxx.android.xxx" 会自动根据平台来加载资源，"xxx@2.xxx" 和 "xxx@3.xxx" 则根据屏幕密度来加载资源

### 静态非图片资源

`require` 语法也可以用来静态地加载你项目中的声音、视频或者文档文件。大多数常见文件类型都支持，包括 `.mp3`, `.wav`, `.mp4`, `.mov`, `.htm` 和 `.pdf` 等

### 混合 App 的资源

使用 `Android` 的 `drawable` 下的资源可以直接通过名字，不带路径和后缀来引用

```javascript
<Image source={{uri: 'app_icon'}} style={{width: 40, height: 40}} />
```

对放置在 `asset` 目录下，可以加上 `asset:/` 前缀，图片也需要加上后缀

```javascript
<Image source={{uri: 'asset:/app_icon.png'}} style={{width: 40, height: 40}} />
```

### 网络资源

首先需要强调的是**需要指定资源尺寸**

```javascript
// 正确
<Image source={{uri: 'https://facebook.github.io/react/img/logo_og.png'}}
       style={{width: 400, height: 400}} />
```

# 参考

- [ReactNative 入门基础](http://reactnative.cn/docs/0.45/tutorial.html#content)
