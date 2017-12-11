# CSS 入门

语法 `selector {property1: value1; property2: value2; ... propertyN: valueN }`

## 四种 Selector

### HTML selector

HTML selector 就是 `HTML` 的 tags, 比如 `p`, `div`, `td` 等，比如我们希望 `h1` 标签指定字体大小和颜色 `h1 {color:red; font-size:14px;}`

### Class selector

以 `.${className}` 来定义，如

```javascript
<style>
.center
{
    text-align:center;
}
</style>

<body>
<h1 class="center">标题居中</h1>
</body>
```

### ID selector

以 `#${id}"` 来定义，如下 Style 应用于 id 属性为 "para1" 的元素上

```javascript
<style>
#para1
{
    text-align:center;
    color:red;
}
</style>

<p id="para1">Hello World!</p>
```

### Attribute selector

直接使用 Tag 标签的 `style` 属性

`<h3 style="color:red;">红色标题</h3>`
