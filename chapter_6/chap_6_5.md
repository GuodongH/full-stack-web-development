# 安全守卫

一般来说，所有应用都一定程度上的权限要求，不可以任意的自由导航的

* 特定的视图只对某些用户授权访问
* 用户需要先鉴权，也就是登录才能访问
* 你需要先获取某些数据才能显示对应视图
* 在离开某一视图之前，你希望提醒用户保存数据或确认取消保存

在 Angular 中引入了一个守卫的概念来帮开发者处理上面的场景，守卫根据场景不同，分为以下几种类型：

* `CanActivate` -- 决定是否可以导航到某个路由
* `CanActivateChild` 决定是否可以导航到某个子路由
* `CanDeactivate` 决定是否可以离开当前路由
* `Resolve` 在路由激活之前加载数据
* `CanLoad` 决定是否可以导航到某个懒加载模块的路由

你在路由的每一层级都可以设置多个守卫， Angular 的规则是只要这些守卫有返回 false 的，那么尚未执行的守卫会取消，整个导航的动作也会取消。从优先级上说， Angular 会先检查 `CanDeactivate` 和 `CanActivateChild` ，检查的顺序是从里（最深层的子路由）往外。然后会检查 `CanActivate` ，这个检查是从外往里进行的。

一个简单的路由守卫可以是一个函数，只要它返回 `Observable<boolean>` ， `Promise<boolean>` 或者 `boolean` 。另外守卫如果要能在各个路由模块中被使用的话，需要 `provide` 出来。

```ts
@NgModule({
  ...
  providers: [
    provide: 'AlwaysActivateGuard',
    useValue: () => {
      return true;
    }
  ],
  ...
})
export class AppModule {}
```

上面就是一个最简单的守卫，它总是返回 `true` ，而且可以通过 `AlwaysActivateGuard` 进行注入，使用这个守卫那么在路由定义中像下面这样即可。

```ts
export const routes:Routes = [
  {
    path: '',
    component: SomeComponent,
    canActivate: ['AlwaysActivateGuard']
  }
];
```

当然大部分情况下，我们还是需要建立一个类来构造一个守卫的，因为守卫里面经常需要注入其他的 `service` 来判断什么样的逻辑下会返回 `true` 或 `false` 。比如下面的例子中，我们需要调用 `AuthService` 来判断用户是否已经登录。

```ts
import { Injectable } from '@angular/core';
import { CanActivate } from '@angular/router';
import { AuthService } from '../auth.service';

@Injectable({
  providedIn: 'root'
})
export class AuthGuard implements CanActivate {

  constructor(private authService: AuthService) {}

  canActivate() {
    return this.authService.isLoggedIn();
  }
}
```

## 激活守卫

## 激活子路由守卫

## 加载守卫

## 退出守卫

## 数据预获取守卫

