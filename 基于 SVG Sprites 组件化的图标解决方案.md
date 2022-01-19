# 基于 SVG Sprites 组件化的图标解决方案

## 前言

图标（Icon）是页面中常见的元素，起着简化文字、引导用户行为的作用。在页面设计上，一方面我们可使用大量的图标来节省空间；另一方面，因视觉符号的统一性，也可通过图标的使用来解决语言国际化的问题。在前端开发过程中，图标引用是非常常见的需求，随着浏览器对矢量图形的支持越来越好，SVG 慢慢成为主流和推荐的方法，应用也越来越广泛。基于SVG的图标技术在行业内已有非常广泛的应用：

* Github、Ant Design、知乎全站采用了 Inline SVG 作为其 Icon 展示方案。
* Icon Font 主站采用了 Inline SVG 的方式展示其所有的 Icon。
* 京东、腾讯视频部分 Icon 采用了基于 SVG Symbols 的 SVG Sprites 方案展示。

对于 Icon Font 相信大家并不陌生，但是仍然存在着一些不足：
1. 浏览器将其视为文字进行抗锯齿优化，有时得到的效果并没有想象中那么锐利。 尤其是在不同系统下对文字进行抗锯齿的算法不同，可能会导致显示效果有所差异。
2. Icon Font 作为一种字体，Icon 显示的大小和位置可能要受到 font-size、line-height、word-spacing 等等 CSS 属性的影响。 Icon 所在容器的 CSS 样式可能对 Icon 的位置产生影响，调整起来很不方便。
3. 使用上存在不便。首先，加载一个包含数百图标的 Icon Font，却只使用其中几个图标，非常浪费加载时间。通过iconfont生成的图标，新增图标需要重新下载替换原有图标资源，不够灵活。
4. 为了实现最大程度的浏览器支持，可能要提供至少四种（TTF、WOFF、EOT、SVG）不同类型的字体文件。
5. 填充仅限于单色，无法支持包含多种颜色的图标。

相对于基于 Icon Font 的图标解决方案，Svg Icon 除了能兼容现有图片，还支持矢量（缩放不失真，在高倍屏幕上更清晰，文件尺寸更小，能保证更快的加载时间以及更好的用户体验）、多色、可读性好等特性，有利于SEO。

本文重点针对 SVG 在前端项目中的实际应用进行介绍。

## 在项目中解决了什么问题？

前面我们提到了关于 SVG 应用前景、优势，但是在实际开发中却并没有那么友好，特别是随着项目的持续迭代，涉及到图标的变动需要开发人员手动进行维护。以下是采用 Svg Icon 进行开发时经常遇到的问题：

1. 每次新增图标均需要手动导入。
2. 遇到页面中需要展示多个图标的场景，多个请求会增加服务端负载，拖慢页面加载速度。
3. 缺乏灵活性，比如图标 hover 时高亮。
4. 可控性差，SVG 本身可能存在默认样式，由于优先级问题可能存在修改样式不生效。

那么怎样更好地提高我们的开发效率呢？这里总结了以下几点解决方案：
1. 自动导入：利用 require.context 原理以及 webpack esMoudle 实现；新增图标无需手动操作即可自动导入，更加灵活，提高可扩展性。
2. 利用 SVG 的 symbol 元素，将每个 svg 文件内容中的 path 内嵌在一个个 symbol 中，通过 id 进行标识(symbolId)，最终合成一个 svg 嵌入到 html 中，这样的好处是不会向服务端发送请求，并且可以在项目任何地方使用。 示例 `<svg><use :xlink:href="symbolId" /></svg>`。
3. 将 Svg Icon 封装为组件，提高可复用性，可维护性。
4. 通过 svgo 插件对 svg 文件进行优化压缩处理，去除不必要的冗余、干扰项。

## 代码实现
### 目录结构

![tree](https://wos.58cdn.com.cn/IjGfEdCbIlr/ishare/pic_15351198642710363.png)

**涉及知识点：**

1. SVG symbol、use
2. svg-sprite-loader 插件
3. require.context 原理
4. CSS3-currentColor
5. svgo 插件

### SVG symbol、use

首先我们需要先了解下 SVG symbol 以及 use 元素。
**[symbol](https://developer.mozilla.org/zh-CN/docs/Web/SVG/Element/symbol)** 元素用来定义一个图形模板对象(本身是不呈现的)，它可以用一个<use>元素实例化。它可以在同一文档中多次使用。只有 symbol 元素的实例（亦即，一个引用了symbol的 <use>元素）才能呈现。

**[use](https://developer.mozilla.org/zh-CN/docs/Web/SVG/Element/use)** 元素在 SVG 文档内取得目标节点，并在别的地方复制它们。它的效果等同于这些节点被深克隆到一个不可见的 DOM 中，然后将其粘贴到 use 元素的位置。
**示例：**

``` 
<svg>
<!-- symbol definition  NEVER draw -->
<symbol id="sym01" viewBox="0 0 150 110">
  <circle cx="50" cy="50" r="40" stroke-width="8" stroke="red" fill="red"/>
  <circle cx="90" cy="60" r="40" stroke-width="8" stroke="green" fill="white"/>
</symbol>
<symbol id="sym02" viewBox="0 0 150 110">
  <circle cx="50" cy="50" r="40" stroke-width="8" stroke="red" fill="red"/>
  <circle cx="90" cy="60" r="40" stroke-width="8" stroke="green" fill="white"/>
</symbol>
<!-- actual drawing by "use" element -->
  <use xlink:href="#sym01" x="0" y="0" width="100" height="50"/>
  <use xlink:href="#sym01" x="0" y="50" width="75" height="38"/>
</svg>
<svg>
  <use xlink:href="#sym02" x="0" y="100" width="50" height="25"/>
</svg>
```
到这我们可以猜想，是不是可以将所有 svg 图标放在一个个 symbol 中，通过 id 标识.
```
<svg>
  <symbol id="sym01">...</symbol>
  <symbol id="sym02">...</symbol>
  <symbol id="sym03">...</symbol>
  ...
</svg>
```
接下来介绍的 svg-sprite-loader 插件就可以帮助我们实现上述功能。

###  处理 SVG 图标

 通过使用 [svg-sprite-loader](https://www.npmjs.com/package/svg-sprite-loader) 插件将 SVG 图片拼接成 SVG Sprites，当浏览器事件 DOMContentLoaded 被触发时，SVG Sprites 将被自动渲染并注入到 document.body 中。其它地方通过 <use> 复用。


![svg6.png](https://wos.58cdn.com.cn/IjGfEdCbIlr/ishare/c26c60f9-3095-4d32-9c90-8c7c6544f187svg6.png)

``` 
 npm install svg-sprite-loader --save-dev
```

我们知道 `vue-cli` 默认会使用 `url-loader` 对 SVG 进行处理，将它打包放在 `/img` 目录下，所以这时候我们使用 `svg-sprite-loader` 会引发一些冲突。解决方式有以下两种：

1. 通过使用 `exclude` 配置 url-loader 只处理除此文件夹之外的 SVG 文件。
2. 通过 `include` 让 svg-sprite-loader 只处理指定文件夹下面的 SVG 文件。

这样就完美解决了之前冲突的问题.

``` js
config.module
    .rule('svg')
    .exclude.add(resolve('src/icons'))
    .end()

config.module
    .rule('icons')
    .test(/\.svg$/)
    .include.add(resolve('src/icons'))
    .end()
    .use('svg-sprite-loader')
    .loader('svg-sprite-loader')
    .options({
        symbolId: 'icon-[name]'
    })
    .end()
```

如果是webpack则需要在 module.rules 中添加如下规则：

``` js
{
    test: /\.(eot|svg|ttf|woff|woff2)(\?\S*)?$/,
    exclude: resolve('src/icons'),
    use: {
        loader: 'file-loader',
        options: {
            esModule: false,
            name: 'fonts/[name].[hash:8].[ext]'
        }
    }
}, {
    test: /\.svg$/,
    include: resolve('src/icons'),
    use: {
        loader: 'svg-sprite-loader',
        options: {
            symbolId: 'icon-[name]'
        }
    }
}
```
这样通过处理我们可以在项目的任意位置使用图片.
```html
<svg>
  <use xlink:href="#icon-notice"></use>
</svg>

<svg>
  <use xlink:href="#icon-add"></use>
</svg>
...
```
每次使用都得 svg 标签包着 use 还是不够简洁，是不是封装个组件在全局注册会更方便呢？

### 组件化
首先创建 svg-icon 组件。

``` vue
//src/components/SvgIcon/index.vue
<template>
  <svg class="svg-icon" aria-hidden="true">
    <use :xlink:href="IconName"></use>
  </svg>
</template>

<script>
export default {
  name: 'icon-svg'，
  props: {
    iconClass: {
      type: String，
      required: true
    }
  }，
  computed: {
    //svg文件名
    iconName() {
      return `#icon-${this.iconClass}`
    }
  }
}
</script>

<style>
.svg-icon {
  width: 1em;
  height: 1em;
  fill: currentColor;
  overflow: hidden;
}
</style>
```

### 自动导入

**引用 svg 目录下的所有 svg 文件**
当我们使用 svg 文件过多或者是有增减时，每次都 import 略显繁琐，使用 require.context 可以实现一次引用全部。
代码如下：

``` js
//src/icons/index.js
import Vue from 'vue'
//引入svg组件
import SvgIcon from '@/components/SvgIcon'

//全局注册Icon-svg
Vue.component('svg-icon'，SvgIcon)

// icons图标自动加载
const req = require.context('./svg'，true， / \.svg$ / );
req.keys().map(req);
```

**require.context的原理**
require.context 会被编译为 __webpack\__require__ 的方式，打包之后，和其他的模块引用完全一样，所以可以完成放心的在项目里使用。
**require.context 函数说明**
require.context 函数接受三个参数：

* directory {String} -读取文件的路径
* useSubdirectories {Boolean} -是否遍历文件的子目录
* regExp {RegExp} -匹配文件的正则 

> 语法: require.context(directory， useSubdirectories = false， regExp = /^.//); 

经过上面的介绍，我们很容易了解到 `require.context('./svg'， true， /\.svg$/)` 的作用是一次性引入 svg 目录下所有 svg 文件。
接下来再看看`req.keys().map(req)`,下图就是我们通过调试的所有内容：

![svg5.png](https://wos.58cdn.com.cn/IjGfEdCbIlr/ishare/ce113780-6043-4285-a5df-9487f97bc1edsvg5.png)

首先我们打印了下 req 返回了一个 webpack 上下文环境，实质上是一个函数，有3个方法分别为id， keys， resolve

1. id {String}  属性：包含 map 对象的模块 id `req.id => "./src/icons/svg sync \.svg$"`
2. keys {Function} 执行它可以返回文件的 key 组成的数组['./logo.svg'，'./doc.svg'，...]
3. resolve {Function}  接收一个 req 参数，是模块文件相对于js执行文件的路径即文件的 key ，返回模块相对于项目启动目录的路径 `req.resolve("./add.svg")=>"./src/icons/svg/add.svg"`
当上下文传入某个文件的键时 `req(req.keys()[0])` 就会得到一个标准的 es module
利用 `req.keys().map(req)` map方法就可以返回所有 svg 文件的 es module

`main.js`
```js
//在main.js 引入SVG
import './icons'
```
只要这个文件被 `main.js` 引用，就会被打到 webpack 的模块依赖中

## 实现效果
通过上一步的实现，可用看到图标的使用方式非常简单，直接使用 svg-icon 标签，将 icon-class 设置为我们的图标文件名即可显示出我们想要的图标; class-name 为可选参数， 通过自定义 class 进行一些简单的样式修改

``` html
<svg-icon icon-class="password" class-name='custom-class' />
```

通过修改自定义 class 元素 color 修改图标的 color ，使用场景：图标 hover 时高亮

![0929.gif](https://wos.58cdn.com.cn/IjGfEdCbIlr/ishare/pic_14161004145632391.gif)

**改变颜色**

svg-icon 通过设置 fill: [currentColor](https://www.html.cn/book/css/values/color/currentColor.htm)，默认会读取其父级的 color ; 
我们可以通过改变父元素的 color 或者直接改变 fill 的颜色即可。

实际项目中可能会发现修改颜色不起作用，什么原因导致的呢？
经分析定位到是当前引用 svg 文件中设置了默认的 fill 属性导致修改颜色不生效

![svg10.png](https://wos.58cdn.com.cn/IjGfEdCbIlr/ishare/2a591dab-f66d-4d85-9f9a-f9ee44f31267svg10.png)

这里提供了以下两种解决方案, 亲测可用：

1. 查看 svg 文件 path 上是否有 fill 属性，若存在删除该属性即可，可以手动删除，这里推荐使用 svgo 工具批量删除。
2. 重写 svg fill样式：path { fill: inherit !important }。

## 进一步优化（基于 svgo 实现 svg 压缩处理优化功能）

为什么需要？
设计师导出的 svg 可能包含大量的无用信息，例如编辑器元数据、注释、隐藏元素、默认值或者非最优值，以及其他一些不会影响渲染结果的可以移除或转换的内容。

![svg12.png](https://wos.58cdn.com.cn/IjGfEdCbIlr/ishare/pic_6473035460409921.png)

可以看到上图 svg 内容已经算蛮精简的了，但是还是会存在一些无用的信息，造成不必要的冗余
这时候我们就可以利用 svgo 去除 svg 中的无用标签，精简结构。

[svgo](https://github.com/svg/svgo)是一个基于 Nodejs 的 svg 文件优化工具，其通过一系列的配置项可以实现定制化的精简 svg 的需求。
 ```npm install svgo --save-dev```
更多详细的配置可以在 /src/icons/svgo.yml 中进行配置。

``` yml
# //src/icons/svgo.yml
plugins:
- removeAttrs:
    attrs:
      - 'fill'
      - 'fill-rule'
```

为了方便我们使用可以在 package.json 的 scripts 添加如下配置：

``` json
  "svgo": "svgo -f src/icons/svg --config=src/icons/svgo.yml"
```

-f FOLDER, --folder=FOLDER : 输入文件夹，优化和重写所有 *.svg 文件
--config=CONFIG : 配置文件或 JSON 字符串，以扩展或替换默认值

执行命令 `npm run svgo` ，根据我们设置的规则批量压缩优化我们的 svg 文件。

![table](https://wos.58cdn.com.cn/IjGfEdCbIlr/ishare/pic_8433081145914322.png)

以下是压缩后的内容:

``` js
<svg width="30" height="30" xmlns="http://www.w3.org/2000/svg"><path d="M27 0a3 3 0 0 1 3 3v24a3 3 0 0 1-3 3H3a3 3 0 0 1-3-3V3a3 3 0 0 1 3-3h24zM16.863 7H9.36l-.13.007c-.69.07-1.23.67-1.229 1.4v14.059l.006.135c.065.713.645 1.272 1.352 1.274h12.284l.13-.007c.692-.07 1.229-.673 1.228-1.403v-9.1l-.002-.02-.002-.017h-4.768l-.131-.006c-.692-.07-1.234-.67-1.234-1.403V7zm1.364 11.953c.243 0 .469.134.59.351a.72.72 0 0 1 0 .704.676.676 0 0 1-.59.35h-5.454l-.104-.007a.681.681 0 0 1-.487-.343.72.72 0 0 1 0-.704.678.678 0 0 1 .59-.35zm0-3.516c.376 0 .68.315.68.703a.692.692 0 0 1-.68.704h-5.454l-.093-.007a.697.697 0 0 1-.59-.697c0-.388.306-.703.683-.703zM17.851 7v5.297L23 12.3v-.002L17.851 7z"/></svg>
```

## 总结

本文开篇介绍了 svg 发展趋势、列举了 svg 在业内应用案例，并且与 Icon Font 进行了对比，然后针对 svg 在前端项目中的应用展开，介绍了在项目中遇到的问题以及优化点，并给出了具体的解决方案。在遇到类似场景时希望能对大家有所帮助。

## 扩展阅读

1. [GitHub 网站在16年就已经将图标的展示方式由 Icon Font 转成了 Inline SVG](https://github.blog/2016-02-22-delivering-octIcons-with-svg/)
2. [来自 CSS Trick: Inline SVG vs Icon Fonts](https://css-tricks.com/icon-fonts-vs-svg/)
3. [为什么要用SVG？svg与iconfont、图片多维度对比](https://www.jianshu.com/p/34167208c699)
4. [手摸手，带你优雅的使用 icon](https://juejin.im/post/6844903517564436493)
