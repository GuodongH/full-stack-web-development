# 使用 Effect 管理副作用

前面我们在利用 `@ngrx/schematics` 生成文件的时候，其实不光是 `reducers` 和 `actions` ，还生成了 `effects` 目录和文件。那么这个 `effects` 是什么呢？

要解释 `effects` ，我们需要回到前一章中关于 `actions` 和 `reducers` 的例子讨论：

> 当我们点击页面上的按钮时发射一个信号，接收到信号之后我们访问后端 API，如果成功则发射一个添加项目成功的 `Action` ，否则发射一个添加项目失败的 `Action` 。当应用接收到添加项目成功的 `Action` 后改变应用状态，在列表中添加这个新的项目，如果 `Action` 是添加项目失败的类型的，那么应用项目列表的状态不变。值得指出的一点是 `Add Project` 这个信号的发射源是页面，而 `Add Project Success` 和 `Add Project Fail` 的发射源是调用后台 API 的逻辑。

在这个例子中，我们来看一下 `Add Project` 这个 Action ，这个信号其实是要调用后端 API 的，而这个信号并不会和 State 有什么直接的联系，只有在后端添加数据成功后发出 `Add Project Success` 时才会对 State 产生影响。这种不对 State 产生影响的，但是在 State 之外的其他方面产生了影响的现象，我们给它起个名字叫 Effects ，译成中文就是副作用。为什么叫副作用？因为这个副作用的“副”是参照 Redux 来的， Redux 的主要作用是维护管理 State ，那么对于非 State 的其他方面的影响就是副作用了。

除了 Http 请求这种常见的 Effects ，其他较常见的还有比如对于 local storage 的读写，对 indexDB 的操作等等。任何非 State 的操作都可以看作 Effects 。
