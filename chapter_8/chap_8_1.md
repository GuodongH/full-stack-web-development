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

如前面所说， Redux 是脱胎于 React 社区的，那么问题来了，在 Angular 中是否适用呢？ Angular 中现有的状态管理机制是什么样子呢？从我个人的经验来看， Redux 对于 Angular 生态的地位要低于它在 React 生态中的地位，这是由于 Angular 内建的依赖注入特性其实在很大程度上解决了状态的管理问题。我们可以通过向组件内注入 service 的方式让多个组件共享数据。所以单纯的状态管理并不是在 Angular 中使用 Redux 的根本原因，Redux 之所以非常流行的主要因素之一是它的工具支持简直是太完美了，对于开发者来说，如果你可以看到每时每刻状态的值的变化以及这些变化产生的来源，这大大的提升了开发人员的生产效率，调试问题不要太简单啊。

![Redux Dev Tool](/assets/2018-07-15-22-32-43.png)

## Redux 的核心概念

Redux 的核心概念非常简单，只有三个基础元素： `Action` 、 `State` 、 `Reducer` 。

### Action -- 信号

`Action` 是什么呢？可以理解成一个信号或者事件：这个信号不仅存在于在页面的交互中，同样存在于应用和后台 API 的交互中。比如我们点击一个增加项目的按钮，也就是发射了一个信号。那么我们预期的结果是项目列表页面上会多一个项目，也就是说项目的状态在应用收到这个信号后改变了。

在 Redux 中的 `Action` 是一个非常简单的 POJO ，有两个属性 `type` 和 `payload` 。 `type` 用于区分信号的类型，而 `payload` 是这个信号携带的数据，这个是可选的。用 `TypeScript` 来定义一下的话就是下面的样子：

```ts
export interface Action {
    type: string;
    payload?: any
}
```

比如上述的增加项目的信号可以定义成下面的样子， `type: 'Add Project'` 就是我们用于区分 Action 以便后面在 Reducer 中采取不同的处理。也就是说系统的信号很多，那么在处理的时候我总要知道这个信号是是干嘛的，然后根据其携带的数据去处理。这个例子中的 `payload`  是一个项目对象，添加一个项目当然要把这个项目的数据发出来，否则就算我们收到这个信号也没办法处理数据。

```ts
{
  type: 'Add Project',
  payload: {
    id: '1234',
    name: '测试项目',
    owner: 'admin'
  }
}
```

前面我们提过， `Action` 不只存在于页面之上，具体来说，比如我们的增加项目如果需要先访问后台 API ，后台 API 添加之后才在前端应用添加项目并展示。那么这个调用后台 API 的动作也应该是由一个信号触发的。那么这个触发动作可以是刚刚我们定义的那个 `Action` 吗？当然可以，但这样做下去就会发现问题， HTTP 请求并不是永远都能成功的，如果失败了，怎么办呢？其实失败的情况还真的挺多的：比如由于我们传递的参数没有满足服务端要求，比如我们访问的 API 路径错误，比如断网或访问超时了等等。

所以最好的方式是重新规划我们的 `Action` ：当我们点击页面上的按钮时发射一个信号，接收到信号之后我们访问后端 API，如果成功则发射一个添加项目成功的 `Action` ，否则发射一个添加项目失败的 `Action` 。当应用接收到添加项目成功的 `Action` 后改变应用状态，在列表中添加这个新的项目，如果 `Action` 是添加项目失败的类型的，那么应用项目列表的状态不变。值得指出的一点是 `Add Project` 这个信号的发射源是页面，而 `Add Project Success` 和 `Add Project Fail` 的发射源是调用后台 API 的逻辑。

```ts
// 由于一般后台 API 会为项目生成一个 ID，所以在前端我们并不在 payload 包含 id ，因为此时 id 尚未产生
{
  type: 'Add Project',
  payload: {
    name: '测试项目',
    owner: 'admin'
  }
}
// 服务端添加成功后，会返回这个已经在后台数据库保存的项目，这个里面就有 ID 了
{
  type: 'Add Project Success',
  payload: {
    id: '1234',
    name: '测试项目',
    owner: 'admin'
  }
}
// 如果服务端添加失败，返回一个添加项目失败的 Action ，携带错误信息
{
  type: 'Add Project Fail',
  payload: {
    code: '400',
    message: '非法请求'
  }
}
```

从上面的的 Action 来看，简单是足够简单了，但是有一个问题就是类型都是字符，这样在大项目中很容易出现拼写错误或者命名重复。在 Angular 中我们利用 TypeScript 的特性可以将其强类型化，就像下面的例子这样，这个后面我们会详细介绍。

```ts
export enum AuthActionTypes {
  Login = '[Login Page] Login',
  LoginSuccess = '[Auth API] Login Success',
  LoginFailure = '[Auth API] Login Failure'
}

export class Login implements Action {
  readonly type = AuthActionTypes.Login;

  constructor(public payload: Auth) {}
}

export class LoginSuccess implements Action {
  readonly type = AuthActionTypes.LoginSuccess;

  constructor(public payload: TokenPair) {}
}

export class LoginFailure implements Action {
  readonly type = AuthActionTypes.LoginFailure;

  constructor(public payload: AppError) {}
}

export type AuthActions =
  | Login
  | LoginSuccess
  | LoginFailure
```

在设计系统的时候，我个人的习惯是先从 Action 开始设计，先把页面上和 API 返回的 Action 都列出来，在 Action 的 type 中一般要标识一下来源，比如是页面产生的还是 API 产生的。先规划 Action 可以让我们对系统交互有一个比较完整的梳理。

### State -- 应用状态

State 这个概念说起来非常简单，你在页面上展现的数据就是 State ，不同条件下按钮的颜色、开启/关闭进行中的动画都是 State 。如果你有后端的开发经验或者移动端的 MVVM 开发经验的话，甚至可以把 State 类比为 View Model 。这个 State 和领域对象是不一样的，领域对象体现的是应用的业务逻辑关系，但 State 一般是要和具体的数据展现逻辑有关系的。一个 State 的定义其实和普通对象也别无二致，就是一个 POJO 。

```ts
export interface State {
  pending: boolean;
  error?: string;
}
```

Redux 的开发者曾经把 State 和数据库做过类比，如果整个应用的状态类比成一个数据库的话，应用中的每个界面或者功能模块对应的 State 就是数据库中的表。

但是和数据库中的表不一样的是， State 是可以层层嵌套的，比如下面的定义中可以看到在应用的根 State 中我们有一个叫 `auth` 的子 State ，而这个叫 `auth` 的 State 又包含了三个子 State: token, loginPage 和 registerPage 。

```ts
import * as fromToken from './token.reducer';
import * as fromLoginPage from './login-page.reducer';
import * as fromRegisterPage from './register-page.reducer';
import * as fromRoot from '../../reducers';

export interface AuthState {
  token: fromToken.State;
  loginPage: fromLoginPage.State;
  registerPage: fromRegisterPage.State;
}

export interface State extends fromRoot.State {
  auth: AuthState;
}
```

### Reducer -- 纯函数的状态处理

讲完了前两个概念，其实 Reducer 就再简单不过了 -- Reducer 就是接收当前的 State 和 Action 作为参数，返回**新的** State 的一个纯函数。注意这是一个新的 State，不是原来的 State ，在 Redux 中我们永远不会修改 State ，而是返回一个新的 State，这一点非常重要。这个函数一般情况下就是一个 `switch...case` 的结构，针对不同的 Action 返回不同的 State 。

从下面的例子可以看出 Reducer 一般情况下就是根据 Action 类型做 `switch` 然后返回不同的 State 。

```ts
export function reducer(state = initialState, action: authActions.AuthActions): State {
  switch (action.type) {
    case authActions.AuthActionTypes.Register: {
      return { pending: true, error: undefined };
    }
    case authActions.AuthActionTypes.ClearRegisterErrors:
    case authActions.AuthActionTypes.RegisterSuccess: {
      return { pending: false, error: undefined };
    }
    case authActions.AuthActionTypes.RegisterFailure: {
      return { pending: false, error: action.payload.title };
    }
    default:
      return state;
  }
}
```

和 State 类似的， Reducer 也是可以分成多级的，从应用的根级到每个界面或功能的级别，同样是采用 key value 字典这样的形式构造，下面的这个例子描述了一个典型的父级 reducer 长成什么样子。

```ts
export const reducers: ActionReducerMap<AdminState> = {
  audit: fromAudit.reducer,
  auditPage: fromAuditPage.reducer,
  user: fromUser.reducer,
  authority: fromAuthority.reducer,
  building: fromBuilding.reducer,
  product: fromProduct.reducer,
  room: fromRoom.reducer,
  workspace: fromWorkspace.reducer
};
```

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

配置好 `@ngrx/schematics` 之后，我们就可以通过 Angular CLI 进行 Reducer 、 Action 、 Effects 等模版的快速构建。比如利用 `store` 子命令可以为应用添加 store 的支持，其中 `--root` 说明我们要创建应用的根 store 。

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

### 构建应用的根 reducer

有了这个模版我们构建应用的 reducer 就方便多了，我们在应用根 State 和 Reducer 中添加 router ，这里我们导入了 `@ngrx/router-store` 。这个类库提供了对于系统路由的 Redux 封装支持，使用这个类库的目的是要让 store 变成系统唯一信任的路由变化源。如果不采用这个类库，你会发现路由的变化是无法体现在 store 中的。这个类库的目的不是替换 Angular Router ，而是要监听路由的变化，将其以 Reducer State 的形式存储起来。此外你也可以封装一些路由 Action 来替换系统提供的路由导航方式，这样做的目的也还是为了统一 store 的行为，让 store 成为唯一的信任源。但请注意，传统的 Router Link 和 navigate 一样可以使用， `@ngrx/router-store` 会监听系统路由的变化的。

```ts
import { ActionReducerMap, createSelector, ActionReducer, MetaReducer } from '@ngrx/store';
import { storeFreeze } from 'ngrx-store-freeze';

import { environment } from '../../environments/environment';
import { RouterStateUrl } from '../utils/router';
import { logout } from '../utils/auth';

import * as fromRouter from '@ngrx/router-store';
import * as fromAuth from '../auth/actions/auth.action';

/**
 * 定义应用的根状态结构
 */
export interface State {
  router: fromRouter.RouterReducerState<RouterStateUrl>;
}

/**
 * 定义应用的根 reducer 结构
 */
export const reducers: ActionReducerMap<State> = {
  router: fromRouter.routerReducer
};

/**
 * 对于退出登录这个特殊的 Action ，我们需要将所有的 state 清空 -- return reducer(undefined, action);
 * 当然我们还需要清除 local storage 中的信息，这个我们通过一个函数 logout() 来完成。
 */
export function storeStateGuard(reducer: ActionReducer<State>): ActionReducer<State> {
  return function(state, action) {
    if (action.type !== fromAuth.AuthActionTypes.Logout) {
      return reducer(state, action);
    }
    logout();
    return reducer(undefined, action);
  };
}

/**
 * 以日志形式输出状态和动作
 */
export function logger(reducer: ActionReducer<State>): ActionReducer<State> {
  return function(state: State, action: any): State {
    console.log('state', state);
    console.log('action', action);

    return reducer(state, action);
  };
}

/**
 * 除了应用本身的 reducer 之外, @ngrx/store 还可以加载一系列的 meta reducer 。
 * 你可以把它的作用想象成插件，下面的例子中在生产环境提供 storeStateGuard 而在开发
 * 环境提供 logger ， storeFreeze 和 storeStateGuard
 */
export const metaReducers: MetaReducer<State>[] = !environment.production
  ? [logger, storeFreeze, storeStateGuard]
  : [storeStateGuard];

```

除去 router 之外，我们还根据系统环境的不同构建了不同的 meta reducer ？咦？怎么又搞出来一个新名词？这个 meta reducer 是什么？

### Meta Reducer

Meta Reducer 其实本质上就是一个函数，一个高阶函数。那么什么又是高阶函数呢？一个接收**函数**作为参数的函数就是高阶函数。那么 Meta Reducer 就是

>>接收一个 reducer 作为参数，并返回一个新的 reducer 的函数

下面是一个非常简单的输出 state 和 action 的日志 meta reducer ，我们可以看到我们定义了一个函数，这个函数接受 reducer 作为参数，然后进行了日志输出，返回了一个新的 reduer 。

```ts
export function logger(reducer: ActionReducer<State>): ActionReducer<State> {
  // 返回一个 reducer 函数
  return function(state: State, action: Action): State {
    console.group(action.type);
    console.log('state', state);
    console.log('action', action);

    return reducer(state, action);
  };
}
```

ngrx 团队为什么要引入这样一个概念呢？在 React 中使用过 Redux 的同学应该知道 Redux 在 React 生态中有 n 多的中间件（ `middleware` ）。这些中间件的效果就是要提供一些 Redux 本身不提供的特性，比如上面例子中我们就为 Redux 添加了在控制台输出日志的功能。而 ngrx 团队感觉通过高阶函数可以达到中间件的目的，没有必要照搬 React 生态中的中间件概念。

### 构建 Feature

原来 Redux 的一大问题是在大型项目中，由于整个项目使用的是同一个根级 reducer 。虽然可以通过文件切割的形式分成各个功能模块的 reducer 文件，但是根 reducer 中仍然需要维护各个子 reducer ，导致大型项目中，团队频繁更改同一个文件，这造成了项目协作上的痛苦。

另一方面，传统的 Reducer 是一棵全局状态树，大型项目的 Reducer 树会非常庞大，而且这棵树上大部分的状态对于当前页面是没有用的，这不仅造成了资源的浪费和性能的降低，而且复杂度升高，导致维护成本上升。

`@ngrx` 在 4.x 以上版本提供了一个解决方案 -- Feature ，简单来说， Feature 构成了全局 store 的一部分，和 Module 在 Angular 中的地位类似，你也可以把它理解成 Store 的模块。这样的安排在 Angular 中实在太方便了，你可以按照 Angular 模块的划分去进行 Reducer 的设计。

比如说，我们新建一个 `Admin` 模块

```bash
ng g m Admin --flat false
#下面的是命令输出
CREATE src/app/admin/admin.module.spec.ts (267 bytes)
CREATE src/app/admin/admin.module.ts (189 bytes)
```

然后通过下面命令，我们建立一个 Admin 的 Feature ，下面的命令会在 `AdminModule` 所在的 `admin` 目录中帮我们构建好完整的 `actions` ， `reducers` 和 `effects` 目录，以及对应的模版文件。这样一个文件结构也是我们推荐的 -- 在模块中建立对应的 Redux 文件。

```bash
ng g f admin/Admin -m admin/admin.module --group
#下面的是命令输出
CREATE src/app/admin/actions/admin.actions.ts (238 bytes)
CREATE src/app/admin/reducers/admin.reducer.ts (387 bytes)
CREATE src/app/admin/reducers/admin.reducer.spec.ts (324 bytes)
CREATE src/app/admin/effects/admin.effects.ts (334 bytes)
CREATE src/app/admin/effects/admin.effects.spec.ts (583 bytes)
UPDATE src/app/admin/admin.module.ts (492 bytes)
```

此外这个命令还会更新 `admin.module.ts` ，我们可以看到，和根模块中 store 的导入方式不同，在这个模块中我们使用 `StoreModule.forFeature` 和 `EffectsModule.forFeature` 来导入 `reducer` 和 `effects`。

```ts
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';
import { StoreModule } from '@ngrx/store';
import { EffectsModule } from '@ngrx/effects';

import { AdminEffects } from './effects/admin.effects';

import * as fromAdmin from './reducers/admin.reducer';

@NgModule({
  imports: [
    CommonModule,
    StoreModule.forFeature('admin', fromAdmin.reducer),
    EffectsModule.forFeature([AdminEffects])
  ],
  declarations: []
})
export class AdminModule { }

```

有了这样的工具，我们构建 Redux 的时候就会少了很多麻烦的文件结构的维护工作，而专注于业务逻辑本身。
