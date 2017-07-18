# 开发环境配置

## 创建项目

如下方式来创建一个 `my-app` 的新项目，新建的项目已经添加了 `React` 和 `ReactDOM` 两个用于开发 `React` 应用的引用

```java
npm install -g create-react-app
create-react-app my-app
```

新项目结构

```
my-app
├── README.md
├── node_modules
├── package.json
├── .gitignore
├── public
│   └── favicon.ico
│   └── index.html  //必须存在，模板
│   └── manifest.json
└── src
    └── App.css
    └── App.js
    └── App.test.js
    └── index.css
    └── index.js  //js 入口，必须存在
    └── logo.svg
    └── registerServiceWorker.js
```

启动项目

```java
cd my-app
npm start
```

## 开发环境

[React Developer Tools 用于检视组件的参数和状态](https://chrome.google.com/webstore/detail/react-developer-tools/fmkadmapgofadopljbjfkapdkoienihi?hl=en)

[代码检查](https://github.com/facebookincubator/create-react-app/blob/master/packages/react-scripts/template/README.md#displaying-lint-output-in-the-editor)

### Atom

[环境配置](http://www.jingyingba.com/Home/Playing/showPage/product_id/67.html)

[JSX 语法高亮](http://babeljs.io/docs/editors)

### WebStorm
