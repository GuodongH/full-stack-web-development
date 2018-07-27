# Angular 路由

我们前面或多或少的接触了 Angular 的路由，那么究竟什么是路由？路由就是用户可以从一个视图“导航”到另一个视图的机制。但是，请注意，这里的导航和浏览器的导航不是一个概念，因为 Angular 中的路由是基于单页应用的，所以不存在真正的链接页面。所以这个链接的形式其实往往更多意味着以 URL 这种形式定义视图的跳转逻辑。

浏览器一般有以下典型行为：

* 在地址栏中输入 URL ，浏览器会导航到指定页面
* 点击页面链接，浏览器会导航到对应页面
* 点击浏览器的前进或后退按钮，浏览器会导航前进或后退到历史页面

Angular 的路由也充分借鉴了这种行为：

* 可以通过 URL 的解析导航到一个前端视图（注意不是页面）
* 可以传递可选参数给目标视图组件，从而视图组件可以根据参数动态显示某些逻辑。
* 也可以在页面上绑定路由链接，用户点击时就会导航到指定的视图
* 当然也支持编程动态跳转，也就是我们除了可以在点击链接的时候跳转，也可以在点击按钮，选择下拉框等等情况下，使用代码来去进行视图跳转。这些跳转的行为会记录在浏览器历史，所以前进和后退按钮也是好用的

## 基准锚链接

大多数的路由应用需要在 `<head>` 下添加一个 `<base>` 元素,这个元素指定了一个基准链接。比如我们如果把 Angular 编译后的文件，也就是 `dist` 中的内容（不包含 `dist` 目录本身），拷贝到 web 服务器的发布根目录中（这一点，每个 web 服务器都不太一样）。也就是说我们的访问链接应该是 `http://localhost` 这样的类型，那么此时我们的基准链接是 `/` ，设定这个基准链接，我们既可以直接修改 `src/index.html` 为

```html
<!-- 省略其他代码 -->
<head>
  <base href="/">
  <!-- 省略其他代码 -->
</head>
<!-- 省略其他代码 -->
```

也可以通过 CLI 命令的形式自动更新，这种 CLI 的形式更适合于开发环境和生产环境不一样的情况。比如开发环境下我们的 base href 是 `/` ，而生产环境下我们是放在 web 服务器的二级目录，比如 `myUrl` 这个目录下。此时应用 `--base-href` 可以在不改动 `src/index.html` 的情况下直接改写编译成功后的文件，也就是 `dist/index.html` 。

```bash
ng build --base-href /myUrl/
```

Angular 会在任何应用中使用的图片、 css 和 javascript 脚本的相对路径**之前**添加 `base href` 的值。换句话说在应用中，如果要引用路径的话，不需要考虑项目的目录层级结构，只需要写相对于基准路径的相对路径即可。比如我们在 `assets` 中有一个图片 `hello.jpg` 那么我们如果在模版中需要引用这个图片的话，只需要写成 `assets/hello.jpg` 即可，无论这个模版文件处于项目工程的哪一层目录。

## Router 模块的简介

在任何时候，如果你需要在模版中使用 `routerLink` 指令，或者在代码中使用导航，均需要导入 `RouterModule`

```ts
import { RouterModule } from '@angular/router';
```

如果是一个路由模块，还需要在模块的导入数组中区分是根路由还是子路由模块，分别使用 `forRoot` 或 `forChild` 进行路由的构造。

```ts
// 根路由模块
imports: [ RouterModule.forRoot(routes) ],
// 子路由模块
imports: [ RouterModule.forChild(routes) ],
```

### 路由表

路由表这个名词听起来很高大上，但其实就是一个路径的配置数组。下面是一个典型的路由表的例子：

```ts
const routes: Routes = [
  {
    path: 'projects',
    loadChildren: '../project#ProjectModule',
    pathMatch: 'prefix',
  },
  {
    path: 'admin',
    loadChildren: '../admin#AdminModule',
    pathMatch: 'prefix',
  },
  {
    path: '',
    redirectTo: '/auth/login',
    pathMatch: 'full'
  },
  {
    path: '**',
    component: PageNotFoundComponent
  }
];
```

首先 `path: ''` 路径为空指的就是根路径，比如如果本地开发 web 服务器的根路径是 `http://localhost:4200` 的话，那么这个 `path: ''` 就是说当你在浏览器地址栏输入以上地址时， Angular Router 会进行路径的对比，发现匹配的条目，就由匹配到的规则处理。

这个匹配的规则有不同的策略

* `pathMatch: 'full'` -- 当且仅当路径完全匹配时才会应用这个规则。比如上面的例子中的 `path: ''` 这个路由定义中我们就使用了 `full` 这种匹配策略，所以它只会匹配 <http://localhost:4200> 这种路径（端口号不同服务器配置会有不同），但是不会匹配 <http://localhost:4200/xxx> 这种路径
* `pathMatch: 'prefix'` -- 当路径的是以 `path` 指定的值开头的时候就会匹配。比如 `projects` 的定义就是应用了 `prefix` 策略，那么它既可以匹配 <http://localhost:4200/projects> 也可以匹配 <http://localhost:4200/projects/123456> 这种路径。

### 积极加载和懒加载

上面的例子中，我们可以看到几种类型的路由定义，类似 `path: '**'` 这个定义中，我们直接指定了一个组件 `component: PageNotFoundComponent` ，这个属于最常见的一种定义，在一个模块中，如果我们希望某个路由匹配时就显示某个组件的话，这种定义就派上用场了。

`path: ''` 对应的是一个重定向，在此情况下，我们将匹配到的路由重定向到另一个路径。这里我们重定向到了 `/auth/login` ，但是很奇怪的是我们并没有再定义 `/auth/login` ，那么系统怎么处理这个路径呢？

这就需要我们引出一个路径模块的概念。在大型项目中，我们的系统经常分成很多模块，这些模块的路由如果都在根路由中定义的话，就会导致路由表过于复杂。另外这样一个架构导致任何一个模块的路由更新都会去更新根路由文件，显然这不是一个良好的设计，维护会因此变得十分困难。

一个 Angular 团队推荐的模式是为每一个功能模块建立该模块自己的路由模块。比如 Feature Module 位于 `app/admin` 目录下，那么该模块的路由模块可以叫做 `admin-routing.module.ts`，同样位于 `app/admin` 目录下。 Angular CLI 甚至提供了一个选项，可以让你在创建一个模块的同时创建它的路由模块。

```ts
ng g m auth --routing
```

我们在这个 `auth-routing.module.ts` 中定义 `auth` 模块的路由

```ts
import { NgModule } from '@angular/core';
import { Routes, RouterModule } from '@angular/router';

import { LoginComponent } from './containers/login/login.component';
import { RegisterComponent } from './containers/register/register.component';
import { ForgotPasswordComponent } from './containers/forgot-password/forgot-password.component';

const routes: Routes = [
  {
    path: 'auth',
    redirectTo: 'auth/login',
    pathMatch: 'full'
  },
  {
    path: 'auth/login',
    component: LoginComponent
  },
  {
    path: 'auth/register',
    component: RegisterComponent
  },
  {
    path: 'auth/forgot',
    component: ForgotPasswordComponent
  }
];

@NgModule({
  imports: [RouterModule.forChild(routes)],
  exports: [RouterModule]
})
export class AuthRoutingModule {}

```

然后需要在 `AuthModule` 中导入这个路由模块，需要指出的是，这个模块不是懒加载模块，所以仍然需要在根模块或者核心模块中导入。

```ts
import { NgModule } from '@angular/core';

import { AuthRoutingModule } from './auth-routing.module';
import { SharedModule } from '../shared/shared.module';

@NgModule({
  imports: [
    SharedModule,
    AuthRoutingModule
  ],
  declarations: [
    ...
  ]
})
export class AuthModule {}

```

回到我们前面提到的重定向特性，系统加载路由时，除了 `AppRoutingModule` 中的路由定义数组，也会加载根模块或核心模块中导入的其他功能模块的路由模块，如果存在的话。还记得我们是在这些模块文件中导入的它们各自的路由模块吗？这就是为什么只要在根模块或核心模块中导入，系统路由就能加载它们的原因。由于我们在核心模块中导入了 `AuthModule`，系统路由定义中就包含了 `AuthRoutingModule` 中的路由定义，所以重定向到 `/auth/login` ，就会显示 `AuthRoutingModule` 中的定义的这个路径对应的组件 -- `LoginComponent` 。

`projects` 和 `admin` 则对应着懒加载模块，使用 `loadChildren` 指定在路由匹配时加载哪个模块。这个模块的路径定义是以 `app` 为基准路径的懒加载模块文件的相对路径。比如 `project` 模块文件位于 `app/project/index.ts` ，而这个例子中我们的 `AppRoutingModule` 位于 `app/core/app-routing.module.ts` 。所以在 `AppRoutingModule` 的这个路由定义数组中，我们就需要使用 `../project#ProjectModule` 找到对应的模块。这里更一般的可以写成 `../project/index#ProjectModule`，但是一个目录下的 `index.ts` 可以作为这个目录的索引文件，所以无需显式指定就可以直接使用目录作为路径作为这个文件的路径。也就是说 `../project` 会直接寻找 `../project/index.ts` ，这样就少写了一层路径。而 `#` 用来区隔模块路径和模块名称， `ProjectModule` 是实际上的这个模块的名称。

接下来我们可以看一下这个懒加载模块的路由定义：

```ts
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';
import { AdminGuard } from './guards/admin.guard';
import { AuthGuard } from '../auth/guards/auth.guard';
import { ProjectListComponent } from './containers/project-list/project-list.component';
import { ProjectDetailComponent } from './containers/product-detail/project-detail.component';
import { TaskGroupListComponent } from './containers/task-group-list/task-group-list.component';
import { TaskGroupDetailComponent } from './containers/task-group-detail/task-group-detail.component';
import { TaskListComponent } from './containers/task-list/task-list.component';

const routes: Routes = [
  {
    path: '',
    canActivate: [AdminGuard], canLoad: [AdminGuard],
    children: [
      {
        path: '',
        component: ProjectListComponent
      },
      {
        path: ':projectId',
        component: ProjectDetailComponent,
        children: [
          {
            path: 'groups',
            children: [
              {
                path: '',
                component: TaskGroupListComponent,
              },
              {
                path: ':groupId',
                component: TaskGroupDetailComponent,
                children: [
                  {
                    path: 'tasks',
                    component: TaskListComponent,
                  }
                ]
              }
            ]
          }
        ]
      }
    ]
  }
];

@NgModule({
  imports: [RouterModule.forChild(routes)],
  exports: [RouterModule]
})
export class ProjectRoutingModule {}

```

这个路由定义中和积极加载模块不一样的是，我们的模块根路径是空 `path: ''` ，但在积极加载模块中，比如 `AuthRoutingModule` ，如果我们设定根路径为空就会导致 `AppRoutingModule` 中的 `path: ''` 的定义无效。因为两者的 `path` 完全一致，而 `AppRoutingModule` 加载在后，这样先匹配的会生效。

但是为什么在懒加载模块中，我们可以有 `path: ''` 这种空字符的定义呢？因为懒加载模块之前是没有加载到系统路由的，而我们激活懒加载的方式是访问某个特定的路由，比如 <http://localhost:4200/projects> ，这个定义是在系统路由中完成的。遇到这个 URL ，系统会加载对应的懒加载模块，这里就是 `ProjectModule` ，但是这个模块自身的路由怎么显示是由 `ProjectRoutingModule` 定义的。这里的空字符的路径不会和 `AppRoutingModule` 中的冲突的原因在于，我们在根路由中定义了一个前缀去激活这个模块，所以这里定义的空字符路径会有一个前缀，就是 `projects` 。

### 路径变量

利用路由传递变量有两种方式，其一是通过路径变量，那么什么叫做路径变量呢？比如 <http://localhost:4200/tasks/1234> 这样一个 URL 中的 `1234` 一般是这个 `Task` 的 `id` ，那么这个 `1234` 也就是一个变量的值。我们可以把类似这种 URL 表示成 <http://localhost:4200/task/:id> ，其中这个 `:id` 就代表这个路径变量，变量名是 `id` ，当然不一定起名叫 `id` ，也可以叫 `taskId` 等等你喜欢的名字。

那么在 Angular 中取得这个变量的值就可以通过注入 `ActivatedRoute` ，通过它的 `paramMap` 得到这个变量

```ts
constructor(private route: ActivatedRoute) {}
ngOnInit() {
  const id$ = this.route.paramMap.pipe(
      filter(params => params.has('id')),
      map(params => params.get('id'))
    );
}
```

需要指出的是，路径变量可以有多个，比如 <http://localhost:4200/projects/123/tasks/456> 。如果我们的定义是 <http://localhost:4200/projects/:projectId/tasks/:taskId> 那么 `projectId` 和 `taskId` 就是两个路径变量。一般来说，表达的意义和 REST API 很像，就是 `id` 为 `123` 的项目下的 `id` 为 `456` 的任务。那么这个例子的路由可以定义成下面的样子：

```ts
{
  path: ':projectId',
  component: ProjectDetailComponent,
  children: [
    {
      path: 'tasks/:taskId',
      component: TaskDetailComponent
    }
  ]
}
```

### 查询参数

另一种参数传递方式就是查询参数，比如类似 <http://localhost:4200/tasks?active=false&projectId=1234> 这样的 URL 中以 `?param1=value1&param2=value2` 的方式传递的方式就是查询参数。同样的可以通过 `ActivatedRoute` 的 `queryParamMap` 得到参数。

```ts
constructor(private route: ActivatedRoute) {}
ngOnInit() {
  const id$ = this.route.queryParamMap.pipe(
      filter(params => params.has('param1')),
      map(params => params.get('param1'))
    );
}
```

和路径变量不一样的是，查询参数无需预先定义在路由定义对象中，在 `routerLink` 指令或 `navigate` 时直接指定即可。

大多数情况下这两种传递参数的形式就已经够用了，但是还是有时候，我们需要传递一些更复杂的参数，这些参数不是单纯的值，而是对象，那么这种情况我们怎么处理呢？这就需要引出路由数据的概念了。

### 路由数据

在路由定义对象中，我们可以通过 `data` 添加需要该路由传递的数据， `data` 是一个字典类型的对象，要传递值的话，就给出一个 `key` ，如下面例子中就是 `breadcrumb` ，这个 `key` 对应的值可以是任意类型。

```ts
{
  path: 'users',
  component: UserHomeComponent,
  canActivate: [AuthGuard, AdminGuard],
  canLoad: [AuthGuard, AdminGuard],
  data: {
    breadcrumb: '用户管理'
  }
},
```

这样定义好之后就可以使用 `ActivatedRoute` 的 `data` 得到数据。

```ts
this.route.data.pipe(
  map(data => data['breadcrumb'])
  )
```

### 匹配策略

系统在匹配到路由表中的第一个符合项时就会导航到对应的 URL ，所以一定要注意顺序，不仅仅是在单个路由数组中的顺序，也要考虑存在多个路由模块时对于路由模块的导入顺序。一般来说，越模糊的（比如使用通配符的）要放在越下面，比如 `path: '**',` 这种一定要放在最下面。上面的例子中 `/auth/login` 是一个预加载模块，这个模块在根模块或者核心模块中导入时就需要在 `AppRouting` 。如果是懒加载模块，比如例子中的 `projects` 和 `admin` 则在数组中需要放在模糊的匹配之前。

## 获取父路由的参数

有的时候我们在子路由中需要获取父路由的参数，在 Angular 中，这是很容易办到的，因为路由是有父子结构的。下面的例子中，如果我们的父路由是 `:floor` ，子路由是 `:floor/:room` ，如果在子路由的组件中我们可以通过下面的方法获得父路由的参数 `floor`

```ts
const routefloor$ = this.route.parent.paramMap.pipe(
      filter(params => params.has('floor')),
      map(params => Number.parseInt(params.get('floor')))
    );
```

类似的 `this.route.parent.parent` 就是往上两层的路由了。

## 获得前一个路由

有的时候，我们还会想要获取前一个路由，这个在 Angular 中同样比较简单，只不过我们需要使用 `rxjs` 的一个操作符 `pairwise` 。这个 pairwise 操作符是发射之前和当前的值组成的数组，比如 `[0,1], [1,2], [2,3], [3,4], [4,5]` 就是一个典型的 `pairwise` 的输出。利用 `Router` 的 `events` 属性可以方便的得到前一个路由。

```ts
constructor(private router: Router) {
  this.router.events.pipe(
    filter(e => e instanceof RoutesRecognized),
    pairwise()
  )
  .subscribe((event: any[]) => {
      console.log(event[0].urlAfterRedirects);
    });
}
```
