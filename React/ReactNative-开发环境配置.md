# ReactNative 开发环境搭建

按照[搭建开发环境](http://reactnative.cn/docs/0.45/getting-started.html#content)来搭建，并没有什么需要注意的

[`WebStorm` 代码提示](https://github.com/virtoolswebplayer/ReactNative-LiveTemplate)

## 新建项目

虽然用命名行新建项目的时候大部分配置都已经设置好，但还是需要了解下 `index.xxx.js` 入口文件的配置逻辑

- 1.`import` 导入相关的包

  例如这样的

  ```javascript
  import React, { Component } from 'react';
  import {
  AppRegistry,
  StyleSheet,
  Text,
  View
  } from 'react-native';
  ```

- 2.创建 `ReactNative` 组件

  ```javascript

  export default class AwesomeProject extends Component {
  render() {
    return (
      <View style={styles.container}>
      //...
      </View>
    );
  }
  }
  ```

- 3.定义样式

  需要用到 `StyleSheet` 这个类

  ```javascript
  const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: '#F5FCFF',
  },
  });
  ```

- 4.注册入口组件

  ```javascript
  // 注意，这里用引号括起来的'AwesomeProject'必须和你 init 创建的项目名一致
  AppRegistry.registerComponent('AwesomeProject', () => AwesomeProject);
  ```
