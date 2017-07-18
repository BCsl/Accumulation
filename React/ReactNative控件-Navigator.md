# ReactNative控件 - Navigator

从0.44版本开始 `react-native` 包下的 `Navigator` 已经被剥离了，如果需要继续使用，如下方式来引用

安装 `npm i -S react-native-deprecated-custom-components`

导入 `import { Navigator } from 'react-native-deprecated-custom-components'`

目前比较主流的应该是 [`react-navigation`](https://reactnavigation.org/docs/navigators/) 这个库，所以后续也只使用这个库好了

内置的几种导航栏，可以嵌套使用

## 渲染屏幕

应用屏幕是通过 `React` 组件来渲染的，而 `Navigator` 也是一个 `React` 组件

## 简单使用

### [`StackNavigator`](https://reactnavigation.org/docs/navigators/stack)

API `StackNavigator(RouteConfigs, StackNavigatorConfig)`

- `RouteConfigs`

路线名：路线配置的映射对象，路线配置信息格式如下

```javascript
StackNavigator({
  // For each screen that you can navigate to, create a new entry like this:
  Profile: {
    // `ProfileScreen` is a React component that will be the main content of the screen.
    screen: ProfileScreen,
    // Optional: When deep linking or using react-navigation in a web app, this path is used:
    // The action and route params are extracted from the path.
    path: 'people/:name',
    // Optional: Override the `navigationOptions` for the screen
    navigationOptions: ({navigation}) => ({
      title: `${navigation.state.params.name}'s Profile'`,
    }),
  },{
    initialRouteName: 'Home', // 默认显示界面
    initialRouteParams : {...},  //默认的路由参数，可以通过 this.props.navigation.state.params 访问
    //设置默认的 navigationOptions，可以被 RouteConfigs 上的或者组件内 static navigationOptions 覆盖
    navigationOptions: {
        title : 'title', //作为headerTitle ，tabBarLabel的后备选项，这个选项是通用的
        header : ... ,//自定义 header
        headerPressColorAndroid : '#ffffff' ,//Android 5.0 的 ripple 颜色
        gesturesEnabled : true,  //右滑结束
    },
    //可视化参数
    mode: ['card'|'modal'],  // 页面切换模式, 'modal' 只对 ios 有效
    headerMode: ['screen'|'float'|'none'], // 导航栏的显示模式, screen: 有淡入淡出, none: 隐藏导航栏
    transitionConfig : ()=> {....} //返回一个重写转场动画的对象
    onTransitionStart: ()=>{ console.log('onTransitionStart'); },
    onTransitionEnd: ()=>{ console.log('onTransitionEnd'); }
  }
});
```

`screen` 指定了要跳转的组件，记得 `import`，`path` 和 `navigationOptions` 可选，其中 `navigationOptions` 中可以配置的信息比较多，如标题、自定义 `Header` 的信息，看官方文档描述

### [`TabNavigator`](https://reactnavigation.org/docs/navigators/tab)

带 `Tab` 分类，类似 `TabLayout`

API `TabNavigator(RouteConfigs, TabNavigatorConfig)`

`RouteConfigs` 部分和之前一样，`TabNavigatorConfig` 可以定义展示位置、样式、标题、是否懒加载等

```javascript
// 注册tabs
const Tabs = TabNavigator({
    Music: {
        screen: Music,
    },
    Video: {
        screen: Video,
        {
          title:"" ,  //tabBarLabel 的后备选项
          tabBarVisible:"", //某个 tab 页面的 tabbar 是否显示
          tabBarIcon: , //icon 元素或者返回 icon 元素方法
          tabBarLabel:'', //Label 元素或者返回 Label 元素方法 或字符串
        }
    }
    , {
      tabBarComponent:[TabBarTop|TabBarBottom], //两种样式对应 ios 和 android
      tabBarPosition: ['bottom'|'top'], //位置
      swipeEnabled: false, // 是否可以左右滑动切换tab
      animationEnabled: false, // 切换页面时是否有动画效果
      lazy : false, //按需加载
      backBehavior: ['none'|'initialRoute'], //按 back 键是否跳转到初始 tab
      initialRouteName : 'Home' , //初始 tab
      order : ['Video','Music'],//Array 对象，tab 的排列顺序
      paths : //重写 `RouteConfigs` 中的 path
      tabBarOptions: {  //Android 和 IOS 有不同的选项
          activeTintColor: '#ff8500', // 文字和图片选中颜色
          inactiveTintColor: '#999', // 文字和图片未选中颜色
          showIcon: true, // android 默认不显示 icon
          showLabel: true, //是否显示 tab 标题
          pressColor: '#xxxxxx',  //ripple 效果颜色，5.0以上
          scrollEnabled: true,  //false，tabs 默认占满屏幕
          tabStyle: {
            width: 100,    
          },
          iconStyle : { ... }
          indicatorStyle: {
              height: 0  // 如TabBar下面显示有一条线，可以设高度为0后隐藏
          },
          style: {
              backgroundColor: '#fff', // TabBar 背景色
              height: 44,
          },
          labelStyle: {
            fontSize: 12,
          },
      },
    }
});
```

### [`DrawerNavigator`](https://reactnavigation.org/docs/navigators/drawer)

抽屉样式

API `DrawerNavigator(RouteConfigs, DrawerNavigatorConfig)`

`RouteConfigs` 和上面基本一样，`DrawerNavigatorConfig` 可以定义展示位置、样式等

```javascript
const SimpleDrawer  = DrawerNavigator(
  {
    music:{
    screen:MusicTab,
    navigationOptions: {
      title: 'MusicTitle',
    },
    },
    video:{
    screen:VideoTab,
    navigationOptions: {
      title: 'VideoTitle',
      drawerLabel : 'VideoLabel',
      drawerIcon : ({ tintColor }) => (
      <Image
        source={require('./notif-icon.png')}
        style={[{tintColor: tintColor}]}
      />
    ),
    },
    },
  },{
    drawerWidth : 100 ,
    drawerPosition : ['right'|'left'],
    initialRouteName : 'Home' , //初始页面
    backBehavior: ['none'|'initialRoute'], //按 back 键是否跳转到初始页面
    order : ['Video','Music'],  //Array 对象，页面的排列顺序
    contentComponent: props => <ScrollView><DrawerItems {...props} /></ScrollView>, //自定义抽屉组件
    contentOptions :{
        items:,
        activeItemKey : '#e91e63',
        activeTintColor : '#e91e63',
        activeBackgroundColor : '#e91e63',
        onItemPress : (route) => { console.log("item press");},
        style: ...,
        labelStyle: ...,
    }  
  }
);
```

## Navigation 参数

使用 `Navigator` 的每一个界面都会得到一个 `navigation` 属性，可以通过 `this.props.navigation` 来访问，包含着以下一些属性

- `navigate`，用来导航到另外一个界面

  可以接受三个参数 `navigate(routeName, params, action)` ，分别是组件注册到 `Navigator` 的名字，传递到下一个界面的参数对象，`action` 如果该界面是一个 `navigator` 的话，将运行这个 `action`

  例子

  ```javascript
  class HomeScreen extends React.Component {
  render() {
    const {navigate} = this.props.navigation; //取出  navigation.navigate
    return (
      <View>
        <Text>This is the home screen of the app</Text>
        <Button
          onPress={() => navigate('Profile', {name: 'Brent'})}
          title="Go to Brent's profile"
        />
      </View>
     )
   }
  }
  ```

- `state`，当前屏幕的路由信息和状态

  通过 `this.props.navigation.state` 形式获得，返回的对象格式如下

  ```javascript
  {
  //注册到 router 中的名字
  routeName: 'profile',
  //a unique identifier used to sort routes
  key: 'main0',
  //可选的界面间传递的参数，由  navigate 决定的
  params: { hello: 'world' }
  }
  ```

- `setParams`，用来修改路由中的参数

  修改 `state` 中的 `params`

- `goBack`，用来帮助进行页面返回

  结束当前页面，返回上一个页面，可以接收参数

- `dispatch`，分发 `action` 到路由，支持以下类型

  - `Navigate`

    导航到页面的 `action`

    ```javascript

    import { NavigationActions } from 'react-navigation'
    const navigationAction = NavigationActions.navigate({
    routeName: 'Profile',
    params: {},
    action: NavigationActions.navigate({ routeName: 'SubProfileRoute'})})
    this.props.navigation.dispatch(navigationAction)
    ```

  - `Reset`

  - `Back`

  - `Set Params`

  - `Init`

**需要注意的是**：如果界面组件是一个 `Navigator`，如 `TabNavigator`，可以只有 `state` 和 `dispatch` 两个属性，[详细描述](https://reactnavigation.org/docs/navigators/navigation-prop)

## Navigation Actions

`navigation.dispatch()` 方法接收一个 `Action` 对象，有以下 5 种 `Action` 对象

- Navigate

这个 `Action` 用来执行跳转

```javascript

import { NavigationActions } from 'react-navigation'  //需要导入 NavigationActions 这个组件

const navigateAction = NavigationActions.navigate({
  routeName: 'Profile', //
  params: {},
  action: NavigationActions.navigate({ routeName: 'SubProfileRoute'}) //如果导航到的是 navigators,则继续执行新的 action
})

this.props.navigation.dispatch(navigateAction)
```

- Reset

  `Reset` 操作会清除原来的路由记录，添加上新设置的路由信息

- Back

  回退到上一个界面

- SetParams

  更新参数，该参数必须是已经存在于 `router`的 `param` 中

- Init

  用于初始化状态，如果状态未定义

## navigationOptions 的配置

之前介绍各种导航的时候的时候，已经提到怎么在路由/导航器中配置 `navigationOptions` 这个参数，但还可以在指定的界面上来配置

在具体的界面上有两种方式来配置 `navigationOptions`

- 1.静态配置

```javascript
class MyScreen extends React.Component {
  static navigationOptions = {
    title: 'Great',
  };
  ...
}
```

- 2.动态配置

选项可以是一个接受 `props` 参数的函数，并返回 `navigationOptions` 对象，该对象将覆盖路由/导航器中定义的 `navigationOptions`

`props` 对象中有以下属性

- `navigation`
- `screenProps`
- `navigationOptions`

```javascript

class ProfileScreen extends React.Component {
  static navigationOptions = ({ navigation, screenProps }) => ({
    title: navigation.state.params.name + "'s Profile!",
    headerRight: <Button color={screenProps.tintColor} {...} />,
  });
  ...
}
```

## 自定义导航栏

# 更多

- [React Navigation](https://reactnavigation.org/docs/intro/)

- [react-native新导航组件react-navigation详解](http://www.jianshu.com/p/7d435e199c96)

- [React Native未来导航者：react-navigation 使用详解](http://blog.csdn.net/u013718120/article/details/72357698)
