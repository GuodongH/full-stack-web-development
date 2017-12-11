# 环境搭建

对于开发者来讲，一个友好的、强大的开发环境可以起到事半功倍的作用。这个章节中我们会把本书需要安装的开发环境和推荐的 IDE 配置。

## 基础开发环境安装

### Ubuntu Linux

在 `Linux` 环境下，推荐使用 `SDKMAN` 来安装 `Spring`、`Java` 以及 `Grails` 等依赖类库。安装 `SDKMAN` 非常简单，打开一个 `terminal` \(命令行终端窗口\)，然后输入以下命令，请一定注意需要科学上网才能确保安装成功：

```
curl -s "https://get.sdkman.io" | bash
```

注意在此过程中，安装脚本可能会提示你缺少某些依赖的软件包，这时我们需要按屏幕提示进行软件依赖的安装，比如下面的这个提示，说的是我们的系统没有安装 zip 和 unzip

```
Looking for a previous installation of SDKMAN...
Looking for unzip...
Looking for zip...
Not found.
======================================================================================================
 Please install zip on your system using your favourite package manager.

 Restart after installing zip.
======================================================================================================
```

那么我们需要使用下面命令进行安装

```
apg-get install zip unzip
```

安装好依赖后，请再次执行上面的 `SDKMAN` 安装脚本并按屏幕提示完成安装。如果你可以看到类似下面的输出，那么安装过程就结束了

```
All done!


Please open a new terminal, or run the following in the existing one:

    source "/home/ubuntu/.sdkman/bin/sdkman-init.sh"

Then issue the following command:

    sdk help

Enjoy!!!
```

安装好之后，在 `terminal` 中再输入以下命令

```
source "$HOME/.sdkman/bin/sdkman-init.sh"
```

这样就安装好了，你可以通过以下命令来验证安装是否成功：

```
sdk version
```

使用 `sdk` 这个命令就可以很方便安装依赖类库



### macOS

`macOS` 对于开发来讲是非常友好的，首先我们需要安装 `brew` ，这是 `macOS` 上的一个包管理工具，类似于 `Ubuntu` 中的 `apt-get` 。在 `terminal` 中键入下面的命令即可完成安装。

```
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

有了 `brew` 之后，再安装其他的软件就简单多了，使用 `brew install` 命令就可以安装你希望的软件了。

### Windows

`Windows` 作为使用最普及的操作系统，很多开发者却是在 `Windows` 上过度的依赖图形化的 IDE，对于环境配置可能并不熟悉。其实在 `Windows` 搭建一个好用的环境也不是很难，但第一件事是要有一个顺手一些的 `terminal` ，不吹不黑，`Windows` 自带的 `cmd` 就不提了，`powershell` 比 `cmd` 好用一些，但也还是比 `*nix` 下的 `terminal` 差着一大截。所以这里强力推荐 `cmder` \( [http://cmder.net/](http://cmder.net/) \)，不但提供 `bash` 的体验到 `Windows`上面，而且集成了 `git`，对于开发者来说是必备神器啊。

类似的，我们还需要一个类似 `brew` 这样的包管理工具，在 `Windows` 平台上，很多类似的这种工具，但其中大部分我感觉还是使用起来有些问题，这里推荐 Scoop \( [http://scoop.sh/](http://scoop.sh/) \)，由于在设计上参考了 `brew` ，体验是非常不错的。

## IDE 的选择

由于本书中，我们会分别涉及到前端和后端的工程，一般来说，前端的工程我个人更喜欢轻量级编辑器，而对于后端来说，一个完善的 IDE 还是必要的。

### Visual Studio Code

因此轻量级编辑器，我推荐 `Visual Studio Code` \( [https://code.visualstudio.com/](https://code.visualstudio.com/) ，注意，访问不了时需要科学上网\)， 简称 `vsc` ，但大家可不要被名字欺骗了，这个东东和 `Visual Studio` 那个大家伙其实没有什么关系，除了都是微软团队做的之外。

### Intellij IDEA

这个老牌 `IDE` 始终是我心中的最佳大型 `IDE` ，即便在 `Eclipse` 最火的年代，我也认为 `IDEA` 比 `Eclipse` 好用太多。

## 字体的选择




