# Http 拦截

大家这几章来回在 Angular 和 Spring 之间切换，是否也感觉到 Angular 和 Spring 之间有很多共享的概念，比如依赖注入、比如注解。Http Interceptor 这个概念对于 Angular 也是存在的。前端是要发起 HTTP 请求到后端的，那么如果我们需要，比如统一在前端发起的请求中加入某些指定的 Header ，这个当然可以在每个服务中去实现。但是不是有点繁琐啊，而且也不利于代码的重构，因为这些加 Header 的地方分散在程序的各个地方。当然 Interceptor 能干的事情不止是这些，我们也可以截获返回的 Response ，根据不同的返回状态做特定的处理。

## 实现一个简单的 HttpInterceptor

Angular 提供了 `HttpInterceptor` 接口，用于拦截 `HttpRequest` 并处理。 `HttpInterceptor` 提供了 `intercept()` 方法，它通常在调用下一个拦截器之前拦截传出请求。 `intercept()` 方法有两个参数 `HttpRequest` 和 `HttpHandler` 。

下面我们就来实现一个 interceptor ，当客户端发送 API 请求到服务器的时候，如果这个 API 的路径传递有误，那么后端会返回一个 `404` ，我们这个截断器的作用就是拦截到请求后，如果返回的错误码是 404 的话，就导航到前端的 404 路由。

```ts
import { Injectable, Injector } from '@angular/core';
import { Router } from '@angular/router';
import {
  HttpInterceptor,
  HttpRequest,
  HttpHandler,
  HttpEvent
} from '@angular/common/http';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class NotFoundInterceptor implements HttpInterceptor {
  constructor(private _injector: Injector) {}

  intercept(
    req: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    return next.handle(req).pipe(
      tap(
        event => {},
        err => {
          if (err.status === 404) {
            const router = this._injector.get(Router);
            router.navigate(['/404']);
          }
        }
      )
    );
  }
}
```

很常见的一个场景是当用户浏览某个不存在的链接时，系统会提供一个 404 页面。传统的 web 开发中，这一块后端会统一处理。但在前后端分离的场景中，我们要知道 Angular 生成的是一个单页面应用 (最终只有一个 `index.html` )，所谓的路由其实只是用来装载不同的组件，浏览器地址栏的那个路由链接其实只是显示而已。所以在我们的场景中，只需要建立一个 404 的组件，在恰当的时候显示它就好了。

## 鉴权 Interceptor

在实际的开发中，通常我们的后台的 Rest API 需要基于 Token 机制的鉴权，这种鉴权在我们的项目中是在 Http Request 的头中写如鉴权信息 `Authorization: Bearer <jwtToken>` 。我们当然可以利用 Angular 的 HttpHeader 对象进行写入，但如果 API 较多，系统较复杂的话，这样做的重复性劳动就太多了一些。

这是一个比较典型的使用 Interceptor 来简化这个流程的场景，我们拦截请求，并将鉴权头设置进去，这样在业务模块中就无需考虑鉴权的问题了，大大简化了业务模块的逻辑。

```ts
import { Injectable } from '@angular/core';
import { HttpInterceptor, HttpHandler, HttpRequest } from '@angular/common/http';
import { Store } from '@ngrx/store';
import { switchMap, map, tap } from 'rxjs/operators';

import { AuthService } from '../../auth/services/auth.service';
import { isRefreshTokenExpired, isIdTokenExpired, getRefreshToken, getIdToken } from '../../utils/auth';
import { RefreshTokenSuccess, Logout } from '../../auth/actions/auth.action';

import * as fromAuth from '../../auth/reducers';

@Injectable()
export class AuthHeaderInterceptor implements HttpInterceptor {
  constructor(private store: Store<fromAuth.State>, private authService: AuthService) {}
  intercept(req: HttpRequest<any>, next: HttpHandler) {
    if (!req.url.includes('/api/') || req.url.includes('/api/auth')) {
      return next.handle(req);
    }
    if (isRefreshTokenExpired()) {
      this.store.dispatch(new Logout());
      return next.handle(req);
    }
    if (isIdTokenExpired()) {
      return this.authService.refreshTokens(getRefreshToken()).pipe(
        tap(pair => this.store.dispatch(new RefreshTokenSuccess(pair))),
        map(pair => pair.id_token),
        switchMap(token => {
          return next.handle(req.clone({ setHeaders: { Authorization: `Bearer ${token}` } }));
        })
      );
    }
    const id_token = getIdToken();
    const reqClone = req.clone({ setHeaders: { Authorization: `Bearer ${id_token}` } });
    return next.handle(reqClone);
  }
}

```

创建 Interceptor 之后，我们需要在要使用的模块中（如果是全局的 Interceptor，一般放在 AppModule 或 CoreModule 中）的 `providers` 数组中列出来。这样系统就会自动使用注册的 Interceptor 进行请求的拦截了。

```ts
// 省略
import { AuthFailureInterceptor } from './interceptors/auth-failure.interceptor';
// 省略
@NgModule({
  // 省略
  providers: [
    { provide: HTTP_INTERCEPTORS, useClass: NotFoundInterceptor, multi: true },
    { provide: HTTP_INTERCEPTORS, useClass: AuthHeaderInterceptor, multi: true },
    { provide: HTTP_INTERCEPTORS, useClass: AuthFailureInterceptor, multi: true }
  ]
})
export class CoreModule {
  constructor(
    @Optional()
    @SkipSelf()
    parentModule: CoreModule,
    ir: MatIconRegistry,
    ds: DomSanitizer
  ) {
    // 省略
  }
}

```

## 一个日志拦截器

在开发中，如果我们可以知道 Http 请求的信息、返回的响应是成功还是失败以及执行了多长时间这些信息，那对于我们在开发中调试问题是很方便的。可能有的同学说 Chrome 的开发者工具已经可以看的这些信息啊，是的，但是这个信息是在你打开开发者工具的情况下才能看到。设想一下，你可能要求你的客户或者是测试人员都一直打开开发者工具，遇到问题再把信息发给你吗？而如果我们有一个日志可以记录下发出的请求，就可以在适当时机将日志传回服务器端进行分析了。

```ts
import { Injectable } from '@angular/core';
import { HttpInterceptor, HttpRequest, HttpHandler, HttpResponse } from '@angular/common/http';
import { finalize, tap } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler) {
    const startTime = Date.now();
    let status: string;

    return next.handle(req).pipe(
        tap(
          event => {
            status = '';
            if (event instanceof HttpResponse) {
              status = '成功';
            }
          },
          error => status = '失败'
        ),
        finalize(() => {
          const elapsedTime = Date.now() - startTime;
          const message = req.method + " " + req.urlWithParams +" "+ status
          + " : " + elapsedTime + "毫秒";

          this.logDetails(message);
        })
    );
  }
  private logDetails(msg: string) {
    console.log(msg);
  }
}
```
