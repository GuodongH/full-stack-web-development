# 服务端渲染

一般的 Angular 应用是运行在浏览器中的，页面在 DOM 中渲染。 Angular Universal 通过服务端渲染（ SSR - Server Side Rendering ）在服务器上生成静态的应用页面。

那么问题来了，说好的前后端分离呢？怎么又搞回老路了呢？我们首先来解释一下服务端渲染的好处有哪些

* 优化 SEO ，对搜索引擎爬虫友好
  * 因为 Angular 本身的控件都是使用 Javascript 操作 DOM 动态生成的，很多搜索引擎爬虫是无法直接解析出页面内容的。
  * 但是在服务端渲染后，页面的渲染在服务端处理后以静态 HTML 输出，这样爬虫就会抓取到内容了。这对于网站的推广是非常有用的，如果你希望在搜索引擎中能够搜到你的网站，那么服务端渲染技术就是比较好的选择了。
* 提高性能
  * 对于 IE 或者不支持 JavaScript 或执行JavaScript 性能很低的环境来说，类似 Angular/React/Vue 的以 Javascript 驱动的页面的用户体验会有影响。对于这些情况，服务端渲染由于输出的是静态 HTML，从而消除了 JavaScript 支持程度的影响。
* 快速显示首页，也就是用户看到的第一个界面
  * 快速显示第一页对于网站来说非常重要。调查显示如果页面加载时间超过 2 秒，很多用户就会放弃访问了。
  * 使用服务端渲染可以让页面加载的更快，因为一个纯粹的静态 HTML 页面加载是比下载 Javascript ，然后再使用 Javascript 动态渲染页面要快很多。

## Angular Universal 的工作机理

Angular Universal 通过一个 `@angular/platform-server` 的软件包来支持 DOM 的服务器实现、 XMLHttpRequest 以及不依赖于浏览器的其他功能。

因此我们需要使用 `platform-server` 模块编译客户端应用，并在 Web 服务器上运行生成的应用。服务器将客户端的页面请求传递给 `platform-server` 中提供的 `renderModuleFactory` 函数。

`renderModuleFactory` 函数接收几个输入参数：

* 模板 HTML（通常是 `index.html` ）
* 包含组件的 Angular 模块
* 显示组件的路由 URL

也就是说和客户端路由不同，客户端路由只是浏览器地址栏的显示，而在服务端渲染时，这个路由请求都会生成相应视图的 HTML。

`renderModuleFactory` 在模板的 `<app>` 标签内嵌入生成的视图 HTML，为客户端创建完成的HTML页面。

最后，服务器将最终生成的页面返回给浏览器。

## 安装依赖

Angular CLI 提供了较为方便的步骤，可以帮我们将现有的应用添加服务端渲染的特性。但在开始之前，我们需要安装以下依赖。

* `@angular/platform-server` -- Angular Universal 服务端组件
* `@nguniversal/module-map-ngfactory-loader` -- 用于在服务端渲染中处理懒加载
* `@nguniversal/express-engine` -- 一个基于 Node.js Express 框架的处理引擎
* `ts-loader` -- 使用 Webpack 进行构建时用于 Typescript 文件处理的扩展
* `webpack-cli` -- 一个 Webpack 命令行软件包

```bash
yarn add @angular/platform-server @nguniversal/module-map-ngfactory-loader express
yarn add -D ts-loader webpack-cli
```

## 添加服务端渲染模块

首先我们需要修改 AppModule ，导入 `BrowserModule` 的方式改动一下，需要通过 `withServerTransition()` 方法指定 `appId` 。

```ts
import { BrowserModule } from '@angular/platform-browser';
import { NgModule } from '@angular/core';
import { CoreModule } from './core';
import { SharedModule } from './shared';
import { LoginModule } from './login';
import { AppComponent } from './core/containers/app';

@NgModule({
  imports: [
    BrowserModule.withServerTransition({ appId: 'taskmgr' }),
    SharedModule,
    LoginModule,
    CoreModule
  ],
  bootstrap: [AppComponent]
})
export class AppModule {}

```

然后我们创建一个单独的 `AppServerModule` ，文件位于 `src/app/app.server.module.ts`

```ts
import { NgModule } from '@angular/core';
import { ServerModule } from '@angular/platform-server';
import { ModuleMapLoaderModule } from '@nguniversal/module-map-ngfactory-loader';
import { AppModule } from './app.module';
import { AppComponent } from './core/containers/app';

/**
 * 用于服务端渲染
 */
@NgModule({
  imports: [
    AppModule,
    ServerModule,
    ModuleMapLoaderModule // 处理懒加载
  ],
  bootstrap: [AppComponent]
})
export class AppServerModule {}

```

类似的，我们需要创建一个 `src/main.server.ts` 为服务端渲染添加一个入口

```ts
import { environment } from './environments/environment';
import { enableProdMode } from '@angular/core';

if (environment.production) {
 enableProdMode();
}

export {AppServerModule} from './app/app.server.module';

```

由于 Node.js 服务端的 Javascript 导入方式和客户端不太一样，我们需要编译打包成 `commonjs` 的形式，所以需要创建一个单独的 `src/tsconfig.server.json`

```json
{
  "extends": "../tsconfig.json",
  "compilerOptions": {
    "outDir": "../out-tsc/app",
    // 注意这里不是 es2015 而是 commonjs
    "module": "commonjs",
    "baseUrl": "./",
    "types": []
  },
  "exclude": [
    "test.ts",
    "**/*.spec.ts"
  ],
  // 添加编译的入口模块
  "angularCompilerOptions": {
    "entryModule": "app/app.server.module#AppServerModule"
  }
}

```

接下来需要在 `angular.json` 中添加一个 `server` 段落

```json
"architect": {
  "build": { ... }
  "server": {
    "builder": "@angular-devkit/build-angular:server",
    "options": {
      "outputPath": "dist/server",
      "main": "src/main.server.ts",
      "tsConfig": "src/tsconfig.server.json"
    },
    "configurations": {
      "production": {
        "fileReplacements": [
          {
            "replace": "src/environments/environment.ts",
            "with": "src/environments/environment.prod.ts"
          }
        ]
      }
    }
  }
}
```

## 使用 Node.js Express 构建服务器

目前 Angular Universal 的官方支持有两个框架： Node.js 和 .Net 。对于 Java 并没有官方支持，但我个人认为影响并不大，因为在微服务架构下，我们完全可以使用 Node.js 作为 Angular 的服务端，而 Java 作为 API 提供方，这样的结合丝毫没有违和感。

接下来我们就开始构建这个服务端，代码比较简单

```ts
import 'zone.js/dist/zone-node';
import 'reflect-metadata';

import { renderModuleFactory } from '@angular/platform-server';
import { enableProdMode } from '@angular/core';

import * as express from 'express';
import { join } from 'path';
import { readFileSync } from 'fs';

// 处理 Lazy Loading 需要导入的
import { provideModuleMap } from '@nguniversal/module-map-ngfactory-loader';

// 激活生产模式可以提供更快的渲染速度
enableProdMode();

// 构建 Express 服务器
const app = express();

const PORT = process.env.PORT || 4000;
const DIST_FOLDER = join(process.cwd(), 'dist');

// index.html 作为模版
const template = readFileSync(
  join(DIST_FOLDER, 'browser', 'index.html')
).toString();

const {
  AppServerModuleNgFactory,
  LAZY_MODULE_MAP
} = require('./dist/server/main');

app.engine('html', (_, options, callback) => {
  renderModuleFactory(AppServerModuleNgFactory, {
    // index.html
    document: template,
    url: options.req.url,
    // 以 DI 形式提供懒加载处理，在服务端渲染中，我们需要立即渲染
    extraProviders: [provideModuleMap(LAZY_MODULE_MAP)]
  }).then(html => {
    callback(null, html);
  });
});

app.set('view engine', 'html');
app.set('views', join(DIST_FOLDER, 'browser'));

// 对于浏览器访问静态文件提供支持
app.get('*.*', express.static(join(DIST_FOLDER, 'browser')));

// 路由访问支持
app.get('*', (req, res) => {
  res.render(join(DIST_FOLDER, 'browser', 'index.html'), { req });
});

// 启动服务，监听端口
app.listen(PORT, () => {
  console.log(`Node server listening on http://localhost:${PORT}`);
});

```

现在我们在前端项目根目录建立一个 `webpack.server.config.js` 的文件。这个文件的作用是编译 `server.ts` 和它的依赖文件，然后生成 `dist/server.js` 。

```js
const path = require('path');
const webpack = require('webpack');

module.exports = {
  mode: 'none',
  entry: {
    server: './server.ts',
  },
  target: 'node',
  resolve: { extensions: ['.ts', '.js'] },
  optimization: {
    minimize: false
  },
  output: {
    // 输出到 dist 文件夹
    path: path.join(__dirname, 'dist'),
    filename: '[name].js'
  },
  module: {
    rules: [
      { test: /\.ts$/, loader: 'ts-loader' },
      {
        test: /(\\|\/)@angular(\\|\/)core(\\|\/).+\.js$/,
        parser: { system: true },
      },
    ]
  },
  plugins: [
    new webpack.ContextReplacementPlugin(

      /(.+)?angular(\\|\/)core(.+)?/,
      path.join(__dirname, 'src'), // src 的位置
      {} // 路由表
    ),
    new webpack.ContextReplacementPlugin(
      /(.+)?express(\\|\/)(.+)?/,
      path.join(__dirname, 'src'),
      {}
    )
  ]
}
```

接下来，我们给 `package.json` 中添加几个脚本命令

```json
"scripts": {
  "ng": "ng",
  "start": "ng serve --port=8000",
  "prod": "ng serve --prod --port=8000",
  "build:ssr": "npm run build:client-and-server-bundles && npm run webpack:server",
  "serve:ssr": "node dist/server.js",
  "build:client-and-server-bundles": "ng build --prod && ng run taskmgr:server",
  "webpack:server": "webpack --config webpack.server.config.js --progress --colors",
  "start:server": "npm run build:ssr && npm run serve:ssr",
  "build": "ng build --prod",
  "test": "ng test",
  "lint": "ng lint",
  "e2e": "ng e2e"
}
```

现在我们使用 `npm run start:server` 就可以了。此时打开浏览器查看源文件的话，我们会看的 `<app-root>` 中是有完整的 HTML 的，而在没有使用服务端渲染的时候，这个里面是没有内容的。

![](/assets/2018-08-24-16-53-04.png)
