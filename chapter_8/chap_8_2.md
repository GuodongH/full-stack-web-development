# 使用 Effect 管理副作用

前面我们在利用 `@ngrx/schematics` 生成文件的时候，其实不光是 `reducers` 和 `actions` ，还生成了 `effects` 目录和文件。那么这个 `effects` 是什么呢？

要解释 `effects` ，我们需要回到前一章中关于 `actions` 和 `reducers` 的例子讨论：

> 当我们点击页面上的按钮时发射一个信号，接收到信号之后我们访问后端 API，如果成功则发射一个添加项目成功的 `Action` ，否则发射一个添加项目失败的 `Action` 。当应用接收到添加项目成功的 `Action` 后改变应用状态，在列表中添加这个新的项目，如果 `Action` 是添加项目失败的类型的，那么应用项目列表的状态不变。值得指出的一点是 `Add Project` 这个信号的发射源是页面，而 `Add Project Success` 和 `Add Project Fail` 的发射源是调用后台 API 的逻辑。

在这个例子中，我们来看一下 `Add Project` 这个 Action ，这个信号其实是要调用后端 API 的，而这个信号并不会和 State 有什么直接的联系，只有在后端添加数据成功后发出 `Add Project Success` 时才会对 State 产生影响。这种不对 State 产生影响的，但是在 State 之外的其他方面产生了影响的现象，我们给它起个名字叫 Effects ，译成中文就是副作用。为什么叫副作用？因为这个副作用的“副”是参照 Redux 来的， Redux 的主要作用是维护管理 State ，那么对于非 State 的其他方面的影响就是副作用了。

除了 Http 请求这种常见的 Effects ，其他较常见的还有比如对于 local storage 的读写，对 indexDB 的操作等等。任何非 State 的操作都可以看作 Effects 。

下面我们来看一个例子来说明 Effects 怎么写

```ts
import { Injectable } from '@angular/core';
import { Observable, of } from 'rxjs';
import { Action, Store } from '@ngrx/store';
import { Actions, Effect } from '@ngrx/effects';
import { map, switchMap, catchError } from 'rxjs/operators';

import { ProjectService } from '../services/project.service';

import * as fromAdmin from '../reducers';
import * as fromProject from '../actions/project.action';

@Injectable()
export class ProjectEffects {
  @Effect()
  loadUsers$: Observable<Action> = this.actions$.ofType<fromProject.Actions>(fromUser.ActionTypes.LoadUsers).pipe(
    switchMap(_ =>
      this.service.getAll().pipe(
        map((projects: Projects) => new fromProject.LoadProjectsSuccess(projects)),
        catchError(error => of(new fromProject.LoadProjectsFailure(error)))
      )
    )
  );

  constructor(private actions$: Actions, private service: ProjectService, private store: Store<fromProject.State>) {}
}

```

首先在 Effects 中的构造函数中注入 Actions ，这个 Actions 就是一个 Action 的事件流，是一个 Obseravble 。

它会监听应用发射出来的所有 Action , `ngrx` 提供了一个操作符 `ofType` ，这个操作符起到的作用就是过滤。比如上面的代码中 `this.actions$.ofType<fromProject.Actions>(fromUser.ActionTypes.LoadUsers)` 就意味着我们只关心 `LoadUsers` 也就是项目中的加载用户列表的 Action 。

既然是一个 Observable ，我们后面就采用了 `switchMap` 表示收到这个 Action 信号后要执行另一个流的操作。要后续操作的这个流就是调用服务层中的加载用户的方法 `getAll()` ，这个方法中调用了 HttpClient 访问后端定义的 API ，可能的结果有两种情况：成功或者失败。

请求成功的时候，我们发射 `LoadProjectsSuccess` 这个 Action ，而失败的时候，我们利用 `catchError` 捕获到异常，然后发射 `LoadProjectsFailure` 这个 Action 。

发射这两个 Action 的意义在哪里呢？因为这个 Effect 只负责发送加载用户列表这个 Http 请求，而这个处理已经结束了，其他的事情不是这个 Effect 要处理的，所以我们把对应的信号发射出去。

那么谁负载处理呢？谁关心谁处理，还记得我们在 Reducer 中有这两个 Action 的对应处理吗？所以 Reducer 关心就是 Reducer 来处理这两种状态。同样的，如果有其他 Effects 关心这两个状态，那它们也会处理。

说到这里，大家应该理解了， Action 是一个一直存在的信号流，而 Reducer 和 Effects 都在监听，选择自己关心的在处理。区别是，接到信号后， Reducer 只改变状态，而 Effects 只关心副作用。
