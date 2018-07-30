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
* `CanLoad` 决定是否可以导航到某个懒加载模块的路由
* `Resolve` 在路由激活之前加载数据

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
import { CanActivate, ActivatedRouteSnapshot, RouterStateSnapshot } from '@angular/router';
import { AuthService } from '../auth.service';

@Injectable({
  providedIn: 'root'
})
export class AuthGuard implements CanActivate {

  constructor(private authService: AuthService) {}

  canActivate(
    route: ActivatedRouteSnapshot,
    state: RouterStateSnapshot
  ): boolean | Observable<boolean> | Promise<boolean> {
    return this.authService.isLoggedIn();
  }
}
```

## 激活守卫

没有权限的应用是很少见的，一般来说，应用都需要根据用户角色限制其访问某些区域，或者根据其帐户是否激活而让用户看到或看不到某些内容。 `CanActivate` 守卫就是一个管理这些导航规则的利器，在 Angular 中实现 `CanActivate` 只需要实现 `canActivate` 这个方法，这个方法返回的有几种可能 `Observable<boolean> | Promise<boolean> | boolean` 这个写法就是返回 `Observable<boolean>` 或者 `Promise<boolean>` 或者 `boolean` 。

```ts
export interface CanActivate {
    canActivate(route: ActivatedRouteSnapshot, state: RouterStateSnapshot): Observable<boolean> | Promise<boolean> | boolean;
}
```

我们在上面的例子中的 `AuthGuard` 其实就是用来判断是否用户已经登录，阻止未登录的用户访问该模块。

## 激活子路由守卫

我们刚才使用 `CanActivate` 阻止了未授权的访问，但如果该路由存在子路由的话，有的时候，你可能需要更细化的权限控制来允许或禁止某些角色访问子路由。此时就是 `CanActivateChild` 派上用场的时候了， `CanActivateChild` 守卫和  `CanActivate` 守卫非常相似，区别是 `CanActivateChild` 是在子路由激活前调用的。同样的，如果要实现一个子路由守卫的话，只需实现 `CanActivateChild` 接口。

```ts
export interface CanActivateChild {
    canActivateChild(childRoute: ActivatedRouteSnapshot, state: RouterStateSnapshot): Observable<boolean> | Promise<boolean> | boolean;
}
```

在下面的例子中， `canActivateChild: [ProjectOwnerGuard]` 用于只能显示用户参与的项目。

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
    canActivate: [AdminGuard],
    canActivateChild: [ProjectOwnerGuard],
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
```

## 加载守卫

加载守卫和激活守卫很多同学会分不清楚，加载守卫其实是用于懒加载模块 -- 阻止用户异步加载整个模块。从效果上看，如果使用 `CanActivate` 而不使用 `CanLoad` ，我们虽然无法访问该模块，但在浏览器中却是可以看到这个模块的源码的

![没有加载守卫的效果](/assets/2018-07-30-07-06-13.png)

而如果使用了 `CanLoad` 的话，我们是无法看到模块源码的，也就是说整个模块都不会加载。

![加载守卫的效果](/assets/2018-07-30-07-11-49.png)

## 退出守卫

`CanDeactivate` 守卫经常使用的一个场景就是用户如果要离开该路由时，提醒用户保存或放弃更改。下面的例子中，我们使用了一个小技巧来将 ``CanDeactivate` 守卫变成一个可以在整个应用共享的形式，使用一个 `CanComponentDeactivate` 让每个需要使用退出提醒功能的组件实现这个接口，从而将退出的具体逻辑交给组件去判断。

```ts
import { Injectable } from '@angular/core';
import { CanDeactivate } from '@angular/router';
import { Observable } from 'rxjs';

export interface CanComponentDeactivate {
  canDeactivate: () => Observable<boolean> | Promise<boolean> | boolean;
}

@Injectable()
export class CanDeactivateGuard implements CanDeactivate<CanComponentDeactivate> {

  canDeactivate(component: CanComponentDeactivate) {
    return component.canDeactivate ? component.canDeactivate() : true;
  }

}
```

## 数据预获取守卫

这个守卫和其他的有明显不同，这个 `Resolve` 接口返回的不是一个布尔型，而是数据本身，这个守卫的目的就是在导航到路由之前，预先得到某些数据。

```ts
interface Resolve<T> {
  resolve(route: ActivatedRouteSnapshot, state: RouterStateSnapshot): Observable<T> | Promise<T> | T
}
```

下面的例子中，我们根据路由参数的 `projectId` 取得项目数据，如果没有数据则导航到项目列表页面。

```ts
@Injectable()
export class ProjectDetailResolver implements Resolve<Project> {
  constructor(private projectService: ProjectService, private router: Router) { }

  resolve(route: ActivatedRouteSnapshot, state: RouterStateSnapshot): Observable<Project> {
    let id = route.paramMap.get('projectid');
    return this.projectService.getProject(id).map(project => {
      if (project) {
        return project;
      } else {
        this.router.navigate(['/projects']);
        return null;
      }
    });
  }
}
```
