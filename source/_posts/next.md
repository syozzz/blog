---
title: Next.js 初见
author: syo
avatar: https://pic.imgdb.cn/item/5efaf31714195aa59486bf81.jpg
categories: 技术
comments: true
tags: 
 - javascript
 - next.js
 - ssr
 - react
date: 2020-8-11 13:40:00
keywords: next.js
description: 使用 Next 做 react 应用的 SSR 
photos: https://pic.imgdb.cn/item/5f1c36a214195aa594c1b1bd.jpg
---

一直以来写 *React* 应用都是基于 *create-react-app* 搭建的,   使用起来倒也顺畅，但是最终 build 出来的 bundle 包往往都非常的大，即使使用 *react-loadable* 按不同页面拆分动态引入，应用首屏渲染所要加载的依赖也是不可小觑的。这就导致首屏加载会有较长的留白时间，用户体验大打折扣。SSR（服务器端渲染 Server Side Render）便是一种解决此类问题的利器。在最近的项目里，便体验了一次使用 *Next.js* 做 *React* 应用的服务端渲染，这里做下记录，以防以后再踩坑。

### 服务端渲染定义

> **SSR服务端渲染**（server side render）*指一般情况下，一个[web页面](https://zh.wikipedia.org/w/index.php?title=Web页面&action=edit&redlink=1)的数据渲染都是由[客户端](https://zh.wikipedia.org/wiki/客户端)或者[浏览器](https://zh.wikipedia.org/wiki/浏览器)端来完成。先从[服务器](https://zh.wikipedia.org/wiki/服务器)请求，然后到页面；再通过[AJAX](https://zh.wikipedia.org/wiki/AJAX)请求到页面数据并把相应的数据填充到[模板](https://zh.wikipedia.org/wiki/模板)，形成完整的页面来呈现给用户。服务端渲染把数据的初始请求放在了服务端，服务端收到请求后，把数据填充到模板形成完整的页面，由服务端把渲染的完整的页面返回给客户端。这样减少了一次客户端到服务端的[HTTP](https://zh.wikipedia.org/wiki/HTTP)请求，加快了相应速度，一般用于首屏的性能优化。

### 服务端渲染优劣

当然，服务端渲染也是一把双刃剑，有优点，也存在缺点。

优势：利于 SEO ，首屏加载快。

劣势：服务器压力大，好废资源。

### Next.js 

> Next.js是一个基于React.js的开放源码的网络套件架构。Next.js的使用者包含腾讯新闻、抖音、币安、Netflix、以及Y Combinator公司投资的Twitch和Scale。Next.js在npm上每周下载次数有47万次。
>
> Next.js能让网络应用程序开发者使用React.js的模式来创造服务器绘制的静态网页或动态网页。因此，网络爬虫可以更有效率的自动浏览网页于达成更好的SEO。相较之下，传统React.js的单页应用大多必须由客户端的JavaScript绘制网页的完整内容。

#### 1.安装

添加依赖

```shell
yarn add next react react-dom
```

在 package.json 中添加脚本

```json
	{
  "scripts": {
    "dev": "next",
    "build": "next build",
    "start": "next start"
  }
}
```

也可以直接使用官方提供的 *create-next-app* 工具进行安装，这里不再赘述。

####2.目录结构

*Next* 默认以根目录下 pages 目录作为页面级组件的映射目录，pages 下的组件都会以文件名注册为对应的路由，并不需要再使用 *react-dom* 显示的声明路由。

*Next* 默认以根目录下 public 目录作为静态资源的映射目录，代码可以通过 / 直接对静态资源进行访问。

#### 3.自定义 Document

通常我们只需要在对应的页面级组件中写代码就行了，*Next* 会自动为我们添加 html, body 等标签。但是，如果有一些需要个性化的地方，比如定义公共的 meta 标签，设置 icon 等，便需要对 Document 进行自定义。在 pages 目录下，创建 _document.js 文件即可自定义 Document。

以引入 styled-components 为例。

```javascript
import Document, { Head, Main, NextScript } from 'next/document'
import { ServerStyleSheet } from 'styled-components'

//继承 Next 的 Document
export default class MyDocument extends Document {

    static async getInitialProps(ctx) {
        const initialProps = await Document.getInitialProps(ctx)
        //挂载 styled-components
        const sheet = new ServerStyleSheet()
        const originalRenderPage = ctx.renderPage
        ctx.renderPage = () => originalRenderPage({
            enhanceApp: App => (props) =>
                // App挂载样式
                sheet.collectStyles(<App {...props} />)
        })
        return {
            ...initialProps,
            styles: (
                <>
                    {initialProps.styles}
                    {sheet.getStyleElement()}
                </>
            )
        }
    }

    render() {
        return (
            <html>
                <Head>
                    <meta name="referrer" content="never" />
                    ...icon meta 等配置
                </Head>
                <body>
                    <Main />
                    <NextScript />
                </body>
            </html>
        )
    }
}
```

**注意：**在 _document.js 中添加除 <Main> 以外的组件，并不会被渲染。

#### 4.自定义 App

当我们还需要对 App 渲染做些自定义逻辑时，可以通过在 pages 目录下添加 _app.js 来实现。

以添加公共吸顶 Header 为例。

```javascript
import App from 'next/app'
import { appWithTranslation } from '../i18n'
import { StickyContainer, Sticky } from 'react-sticky'
import Header from '../components/header'
import Tabs from '../components/tabs'

class MyApp extends App {

    static async getInitialProps ({ Component, router, ctx }) {
        let pageProps = {}

        if (Component.getInitialProps) {
            pageProps = await Component.getInitialProps(ctx)
        }

        return {pageProps}
    }

    render() {
        const { Component, pageProps } = this.props
        return (
            <StickyContainer>
                <Header/>
                <Sticky topOffset={60}>
                    {({
                          style
                      }) => (
                        <div
                            style={{
                                ...style,
                                zIndex: 1000,
                                background: '#ffffff'
                            }}
                        >
                            <Tabs style={{...style}}/>
                        </div>
                    )}
                </Sticky>
                <Component {...pageProps} />
            </StickyContainer>
        )
    }
}

export default appWithTranslation(MyApp)
```

#### 5.整合 ant-design

- 添加依赖

  ```shell
  yarn add @next/bundle-analyzer babel-plugin-import antd @ant-design/icons
  ```

- 根目录下添加 next.config.js

  ```javascript
  const withBundleAnalyzer = require('@next/bundle-analyzer')({
      enabled: process.env.ANALYZE === 'true',
  })
  
  module.exports = withBundleAnalyzer()
  ```

- _app.js 中引入 antd 的样式

  ```javascript
  ...
  import 'antd/dist/antd.min.css'
  ...
  ```

- 根目录下新增 .babelrc 文件

  ```javascript
  {
    "presets": [
      "next/babel"
    ],
    "plugins": [
      [
        "babel-plugin-styled-components",
        {
          "ssr": true,
          "displayName": true,
          "preprocess": false
        }
      ],
      [
        "import",
        {
          "libraryName": "antd",
          "libraryDirectory": "lib",
          "style": "index.css"
        }
      ],
      [
        "import",
        {
          "libraryName": "@ant-design/icons",
          "libraryDirectory": "lib/icons",
          "camel2DashComponentName": false
        },
        "@ant-design/icons"
      ]
    ]
  }
  ```

### 参考

- [维基百科]([https://zh.wikipedia.org/wiki/%E6%9C%8D%E5%8A%A1%E5%99%A8%E7%AB%AF%E6%B8%B2%E6%9F%93](https://zh.wikipedia.org/wiki/服务器端渲染))
- [Next.js 文档]([https://nextjs.frontendx.cn/docs/#%E5%AE%89%E8%A3%85](https://nextjs.frontendx.cn/docs/#安装))