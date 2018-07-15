# 使用 Redux 管理状态

从前面的使用情况来看，我们的状态管理由于采用了 `rxjs` ，效果还是不错的。但是这种状态管理方案由于没有统一的一个标准，在大型项目的具体实施中，很可能会走了样。我们这里来介绍一下 `Redux` <https://redux.js.org/introduction> 。 `Redux` 自从在 `React.js` 社区火爆之后，其思想很大的影响了其他框架，包括 `Vue` 和 `Angular` ，现在可以认为 `Redux` 是业界比较认可的一个状态管理方案，虽然它也不是完美的。

Redux是为了解决应用状态（State）管理而提出的一种解决方案。那么什么是状态呢？简单来说对于应用开发来讲，UI上显示的数据、控件状态、登陆状态等等全部可以看作状态。

我们在开发中经常会碰到，这个界面的按钮需要在某种情况下变灰、那个界面上需要根据不同情况显示不同数量的 Tab 、这个界面的某个值的设定会影响另一个界面的某种展现等等。这些的背后就是状态，状态分为 UI 可视部分和数据部分两大类，数据的有的可以体现在界面上，比如增加一个新的 item 到列表，但也有无法直接体现在界面上的，比如某种设置，是否存储到本地 IndexDB 等。应该说应用开发中最复杂的部分就在于这些状态的管理。很多项目随着需求的迭代，代码规模逐渐扩大、团队人员水平参差不齐就会遇到各种状态管理极其混乱，导致代码的可维护性和扩展性降低。

## 何时需要使用 Redux？

看我前面说了一堆问题，好像不用 Redux 就天塌下来了。其实不是的，Redux ，就像其他任何一种设计模式一样，有它的适用范围，千万不要认为它是万能的。不分场合的使用 Redux 有的时候会增加业务上不必要的复杂度， Redux 的开发者曾经有一篇著名的帖子 - 你也许不需要 Redux <https://medium.com/@dan_abramov/you-might-not-need-redux-be46360cf367#.z9abvda1k> ，建议大家可以去读一下。在文章中，作者已经非常清晰的列出了什么时候适合采用 Redux ，何时不需要使用 Redux 。这里我简单摘要并补充一些自己的理解。

Redux 要求开发者改变编程的范式

* 使用 POJO (Plain Old Javascript Object) 描述应用状态
* 使用 POJO 描述变化
* 使用纯函数处理变化的逻辑

这几个要求是使用 Redux 需要做的妥协，在你考虑使用 Redux ，一定要认真评估，因为这几个要求还是侵入式很强的约束。一般来说，在你可以使用本地状态很清晰的完成业务的时候，你是**不需要**使用 Redux 的。当然这样说你可能还是不清楚什么时候适合使用 Redux ，这里我尝试给出一些具体的场景

* 你有一些数据，这些数据需要在多个界面展示。换句话说如果仅仅需要在组件内部使用的状态是不需要使用 Redux 的。
* 当你发现需要在组件中维护多个成员变量，而这些变量已经构成了非常复杂的条件判断，导致代码的可读性和可维护性出现了严重问题，这个时候 Redux 往往可以帮你解耦状态逻辑和组件本身，而 Redux 中的逻辑由于是纯函数，也非常易于测试。
* 当你需要将本地状态保存到 local storage 或者在服务端渲染时希望预先组装状态时，因为我们在 Redux 中使用 POJO 描述状态和变化，这个特性让我们可以较容易的实现类似的功能。

如果有兴趣的同学可以去看 Facebook 团队的一个分享 <https://youtu.be/nYkdrAPrdcw> （注意：可能需要科学上网），里面非常详尽的解释了 Facebook 团队当初遇到的一个非常难解决的 bug ，以及由这个 bug 产生的为什么要使用类似 Redux 这种 Store 解决方案的过程。

### Angular 中有哪些现有的状态管理机制？

如前面所说， Redux 是脱胎于 React 社区的，那么问题来了，在 Angular 中是否适用呢？ Angular 中现有的状态管理机制是什么样子呢？从我个人的经验来看， Redux 对于 Angular 生态的地位要低于它在 React 生态中的地位，这是由于 Angular 内建的依赖注入特性其实在很大程度上解决了状态的管理问题。我们可以通过向组件内注入 service 的方式让多个组件共享数据。所以单纯的状态管理并不是在 Angular 中使用 Redux 的根本原因，Redux 之所以非常流行的主要原因之一是它的工具支持简直是太完美了，对于开发者来说，如果你可以看到每时每刻状态的值的变化以及这些变化产生的来源，这大大的提升了开发人员的生产效率，调试问题不要太简单啊。

![Redux Dev Tool](/assets/2018-07-15-22-32-43.png)

另一个 Redux 可以大放光彩的地方是

## 如何

Action/Reducer/

## 在 Angular 中使用 Redux

Angular 团队中的一些成员开发了一个开源项目 `ngrx` <https://github.com/ngrx/platform> ，这个项目差不多算作半个官方性质的 Angular 版的 Redux 吧。对应的子项目有以下几个

* `@ngrx/store` - 使用 `RxJS` 实现的 Redux 状态管理框架
* `@ngrx/effects` - 对于 action 产生的副作用管理（副作用我们在后面章节有介绍）
* `@ngrx/router-store` - 将 Angular 路由连接到 store，让你可以像管理其他状态一样管理路由
* `@ngrx/store-devtools` - 让 ngrx 应用也能使用强大的 Redux Dev Tool （你一定会喜欢时光机器这么强大的特性）
* `@ngrx/entity` - 一个帮助类库，用来简化 reducer 的写法，简化常见的增删改查操作
* `@ngrx/schematics` - 用于使用 Angular-CLI 生成 reducer 、effects 或 action 模版文件

### 安装 ngrx

使用下面的命令安装，为了方便，我们一次性安装所有的软件包。

```bash
cnpm install --save @ngrx/store @ngrx/effects @ngrx/router-store @ngrx/entity
cnpm install --save-dev @ngrx/store-devtools @ngrx/schematics @angular-devkit/core @angular-devkit/schematics
```

或者

```bash
yarn add @ngrx/store @ngrx/effects @ngrx/router-store @ngrx/entity
yarn add @ngrx/store-devtools @ngrx/schematics @angular-devkit/core @angular-devkit/schematics --dev
```

### 配置 Angular CLI 命令

项目工程大了之后，很多同学会觉得 ngrx 的写法很繁琐，因为要建立 action 、 reducrer 、 effects 等等。 ngrx 团队也意识到了这一点，因此为开发者提供了 `@ngrx/schematics` ，让我们可以使用 Angular CLI 来快速建立这些文件。为了以后自动生成文件比较方便，可以使用下面的命令，将其 `@ngrx/schematics` 设置为 CLI 的默认

```bash
ng config cli.defaultCollection @ngrx/schematics
```

`@ngrx/schematics` 继承了 Angular CLI 的 `@schematics/angular` 定义的命令集合，但如果我们想要改变一些默认值的时候，比如如果要生成使用 `scss` 的组件文件时，我们就需要在 `angular.json` 加入下面的键值对：

```json
"schematics": {
  "@ngrx/schematics:component": {
    "styleext": "scss"
  }
}
```

配置好 `@ngrx/schematics` 之后，我们就可以通过 Angular CLI 进行 Reducer 、 Action 、 Effects 等模版的快速构建。比如利用 `store` 子命令可以为应用添加 store 的支持。

```bash
ng generate store State --root --module app.module.ts
```

当然也可以利用简略的缩写命令进行，其中 `g` 就是 `generate` 的缩写，而 `st` 就是 `store` 的缩写

```bash
ng g st State --root --module app.module.ts
```

这个命令会在 `app` 目录下新建一个 `reducers` 目录，并在其中新建一个 `index.ts` ，这个文件就是项目根级别的 reducer ，在这个文件中，我们会定义整个应用的状态类型和 reducer ，我们可以看到 CLI 为我们生成了下面的模版。

```ts
import {
  ActionReducer,
  ActionReducerMap,
  createFeatureSelector,
  createSelector,
  MetaReducer
} from '@ngrx/store';
import { environment } from '../../environments/environment';

export interface State {

}

export const reducers: ActionReducerMap<State> = {

};


export const metaReducers: MetaReducer<State>[] = !environment.production ? [] : [];

```

有了这个模版，你就可以直接定义状态和 Reducer 了，而且 CLI 聪明的更新了 `app.module.ts` 为我们导入了必要的 ngrx 模块。

```ts
import { BrowserModule } from '@angular/platform-browser';
import { NgModule } from '@angular/core';

import { AppComponent } from './app.component';
import { StoreModule } from '@ngrx/store';
import { reducers, metaReducers } from './reducers';
import { StoreDevtoolsModule } from '@ngrx/store-devtools';
import { environment } from '../environments/environment';

@NgModule({
  declarations: [AppComponent],
  imports: [
    BrowserModule,
    StoreModule.forRoot(reducers, { metaReducers }),
    !environment.production ? StoreDevtoolsModule.instrument() : []
  ],
  providers: [],
  bootstrap: [AppComponent]
})
export class AppModule {}

```

###

```ts
import { ActionReducerMap, createSelector, createFeatureSelector, ActionReducer, MetaReducer } from '@ngrx/store';

import { environment } from '../../environments/environment';
import { RouterStateUrl } from '../utils/router';

import * as fromRouter from '@ngrx/router-store';
import * as fromAuth from '../auth/actions/auth.action';

/**
 * storeFreeze prevents state from being mutated. When mutation occurs, an
 * exception will be thrown. This is useful during development mode to
 * ensure that none of the reducers accidentally mutates the state.
 */
import { storeFreeze } from 'ngrx-store-freeze';
import { logout } from '../utils/auth';

/**
 * As mentioned, we treat each reducer like a table in a database. This means
 * our top level state interface is just a map of keys to inner state types.
 */
export interface State {
  router: fromRouter.RouterReducerState<RouterStateUrl>;
}

/**
 * Our state is composed of a map of action reducer functions.
 * These reducer functions are called with each dispatched action
 * and the current or initial state and return a new immutable state.
 */
export const reducers: ActionReducerMap<State> = {
  router: fromRouter.routerReducer
};

export function storeStateGuard(reducer: ActionReducer<State>): ActionReducer<State> {
  return function(state, action) {
    if (action.type !== fromAuth.AuthActionTypes.Logout) {
      return reducer(state, action);
    }
    logout();
    return reducer(undefined, action);
  };
}

// console.log all actions
export function logger(reducer: ActionReducer<State>): ActionReducer<State> {
  return function(state: State, action: any): State {
    console.log('state', state);
    console.log('action', action);

    return reducer(state, action);
  };
}

/**
 * By default, @ngrx/store uses combineReducers with the reducer map to compose
 * the root meta-reducer. To add more meta-reducers, provide an array of meta-reducers
 * that will be composed to form the root meta-reducer.
 */
export const metaReducers: MetaReducer<State>[] = !environment.production
  ? [logger, storeFreeze, storeStateGuard]
  : [storeStateGuard];

```
