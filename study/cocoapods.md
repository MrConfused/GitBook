---
description: 一种应用级别的依赖管理器。
---

# CocoaPods

> [冬瓜：CocoaPods历险记](https://mp.weixin.qq.com/mp/appmsgalbum?\_\_biz=MzA5MTM1NTc2Ng==\&action=getalbum\&album\_id=1477103239887142918\&scene=173\&from\_msgid=2458324020\&from\_itemidx=1\&count=3\&nolastread=1#wechat\_redirect)

## 一、版本管理工具及 Ruby 工具链环境

### 1 背景知识

#### 1.1 Version Control System (VCS)

`版本控制系统（VCS）`是敏捷开发的重要一环

`Source Code Manager` (SCM) 源码管理就属于 VCS

在SCM中有`CocoaPods`和`Git`等工具。`CocoaPods` 是针对各种语言所提供的 `Package Manger (PM)`。

`Git` 或 `SVN` 是针对项目的**单个文件**的进行版本控制，而 PM 则是以每个独立的 **Package** 作为最小的管理单元。

被 PM 接管的依赖库的文件，通常会在 `Git` 的 `.ignore` 文件中选择忽略它们。

#### 1.2 Git Submodule

**`Git Submodules` 可以算是 PM 的“青春版”，它将单独的 git 仓库以子目录的形式嵌入在工作目录中**。

它不具备 PM 工具所特有的**语义化版本**管理、无法处理依赖共享与冲突等。只能保存每个依赖仓库的文件状态。

Git submodule 是依赖 `.gitmodules` 文件来记录子模块的。

`.gitmodules` 仅记录了 path 和 url 以及模块名称的基本信息， 但是我们还需要记录每个 Submodule Repo 的 commit 信息，而这 commit 信息是记录在 `.git/modules` 目录下。同时被添加到 `.gitmodules` 中的 path 也会被 git 直接 ignore 掉。

#### 1.3 Package Manger

PM 基本都具备了语义化的版本检查能力，依赖递归查找，依赖冲突解决，以及针对具体依赖的构建能力和二进制包等。

| **Key File** | **Git submodule** | **CocoaPods** | **SPM**          | **npm**           |
| ------------ | ----------------- | ------------- | ---------------- | ----------------- |
| **描述文件**     | .gitmodules       | Podfile       | Package.swift    | Package.json      |
| **锁存文件**     | .git/modules      | Podfile.lock  | Package.resolved | package-lock.json |

* **描述文件**：声明了项目中存在哪些依赖，版本限制；
* **锁存文件（Lock 文件）**：记录了依赖包最后一次更新时的全版本列表。

#### 1.4 CocoaPods

**`CocoaPods` 是开发 iOS/macOS 应用程序的一个第三方库的依赖管理工具。**

利用 `CocoaPods`，可以定义自己的依赖关系（简称 `Pods`）

![](<../.gitbook/assets/image (4).png>)

**Podfile**

`Podfile` 是一个文件，以 DSL（其实直接用了 Ruby 的语法）来描述依赖关系，用于定义项目所需要使用的第三方库。

**Podfile.lock**

它记录了需要被安装的 Pod 的每个已安装的版本。

如果你想知道已安装的 `Pod` 是哪个版本，可以查看这个文件。

推荐将 `Podfile.lock` 文件加入到版本控制中，这有助于整个团队的一致性。

**Manifest.lock**

这是每次运行 `pod install` 命令时创建的 `Podfile.lock` 文件的副本。

如果遇到错误 **沙盒文件与 `Podfile.lock` 文件不同步 (The sandbox is not in sync with the `Podfile.lock`)**，这是因为 `Manifest.lock` 文件和 `Podfile.lock` 文件不一致所引起。

由于 `Pods` 所在的目录并不总在版本控制之下，这样可以保证开发者运行 App 之前都能更新他们的 `Pods`，否则 App 可能会 crash，或者在一些不太明显的地方编译失败。

**Master Specs Repo**

`CocoaPods` 通过官方的 Spec 仓库来管理这些注册的依赖库。

但是随着不断新增的依赖库导致 Spec 的更新和维护成为了使用者的包袱。这个问题在 1.7.2 版本中已经解决了，`CocoaPods` 提供了 **Mater Repo CDN** ，可以直接 CDN 到对应的 Pod 地址而无需在通过本地的 Spec 仓库了。在 1.8 版本中，官方默认的 Spec 仓库已替换为 CDN，其地址为 [**https://cdn.cocoapods.org**](https://cdn.cocoapods.org)。

### 2 Ruby 生态及工具链

`CocoaPods` 是通过 Ruby 语言实现的。它本身就是一个 `Gem` 包。

**`RVM` 和 `rbenv` 都是管理多个 Ruby 环境的工具，它们都能提供不同版本的 Ruby 环境管理和切换。**

#### 2.1 RubyGems

**RubyGems 是 Ruby 的一个包管理工具，管理着称为 Gem的用 Ruby 工具或依赖。**

当我们使用 `gem install xxx` 时，会通过 `rubygems.org` 来查询对应的 Gem Package。

而 iOS 日常中的很多工具都是 Gem 提供的，例：`Bundler`，`fastlane`，`jazzy`，`CocoaPods` 等。

在默认情况下 Gems 总是下载 library 的最新版本，这无法确保所安装的 library 版本符合我们预期。因此我们还缺一个工具。

#### 2.2 Bundler

Gemfile 的 DSL 写法和 Podfile 如出一辙。

**Bundler 是管理 Gem 依赖的工具，可以隔离不同项目中 Gem 的版本和依赖环境，本身也是一个 Gem。**

Bundler 通过读取项目中的依赖描述文件 `Gemfile` ，来确定各个 Gems 的版本号或者范围，来提供了稳定的应用环境。

当我们使用 `bundle install` 它会生成 `Gemfile.lock` 将当前 librarys 使用的具体版本号写入其中。之后，他人再通过 `bundle install` 来安装 libaray 时则会读取 `Gemfile.lock` 中的 librarys、版本信息等。

**Gemfile**

**Bundler 依据项目中的 `Gemfile` 文件来管理 Gem，（而 `CocoaPods` 通过 Podfile 来管理 Pod）。**

那什么情况会用到 Gemfile 呢 ？

`CocoaPods` 每年都会有一些重大版本的升级，前面聊到过 `CocoaPods` 在 `install` 过程中会对项目的 `.xcodeproj` 文件进行修改，不同版本其有所不同，这些在变更都可能导致大量 `conflicts`，处理不好，项目就不能正常运行了。

如果项目是基于 `fastlane` 来进行持续集成的相关工作以及 App 的打包工作等，也需要其版本管理等功能。

### 3 如何安装一套可管控的 Ruby 工具链？

**我们可以使用 `homebrew` + `rbenv` + `RubyGems` + `Bundler` 这一整套工具链来控制一个工程中 Ruby 工具的版本依赖。**

![操作安装流程](../.gitbook/assets/image.png)

#### 3.1 使用 `homebrew` 安装 `rbenv`

```
brew install rbenv
```

#### 3.2 使用 `rbenv` 管理 Ruby 版本

```
rbenv install 2.6.0
```

安装成功后，让其生效：

```
rbenv global 2.6.0
```

rbenv global 2.6.0 # 默认使用 2.6.0

rbenv shell 2.6.0 _# 当前的 shell 使用 2.6.0, 会设置一个 `RBENV_VERSION` 环境变量_

rbenv local j2.6.0 _# 当前目录使用2.6.0, 会生成一个 `.rbenv-version` 文件_

> 输入上述命令后，可能会有报错。`rbenv` 提示我们在 `.zshrc` 中增加一行 `eval "$(rbenv init -)"` 语句来对 `rbenv` 环境进行初始化。如果报错，我们增加并重启终端即可。



### 4 如何使用 Bundler 管理工程中的 Gem 环境

在项目中增加一个 `Gemfile` 描述，从而锁定当前项目中的 `Gem` 依赖环境。

#### 4.1 在 iOS 工程中初始化 `Bundler` 环境

其实就是自动创建一个 `Gemfile` 文件：

```
bundle init
```

#### 4.2 在 `Gemfile` 中声明使用的 `CocoaPods` 版本并安装

编辑 `Gemfile` 文件，之后执行一下 `bundle install` ：

```
bundle install
```

#### 4.3 使用当前环境下的 `CocoaPods` 版本操作 iOS 工程

```
bundle exec pod install
```

使用当前环境的pod版本来安装项目依赖（前提是你还需要写一个 `Podfile` ）。



## 二、CocoaPods 核心组件

### 1 CocoaPods项目的依赖

{% code title="Gemfile" %}
```ruby
SKIP_UNRELEASED_VERSIONS = false
# ...

source 'https://rubygems.org'
gemspec
gem 'json', :git => 'https://github.com/segiddins/json.git', :branch => 'seg-1.7.7-ruby-2.2'

group :development do
  cp_gem 'claide',                'CLAide'
  cp_gem 'cocoapods-core',        'Core', '1-9-stable'
  cp_gem 'cocoapods-deintegrate', 'cocoapods-deintegrate'
  cp_gem 'cocoapods-downloader',  'cocoapods-downloader'
  cp_gem 'cocoapods-plugins',     'cocoapods-plugins'
  cp_gem 'cocoapods-search',      'cocoapods-search'
  cp_gem 'cocoapods-stats',       'cocoapods-stats'
  cp_gem 'cocoapods-trunk',       'cocoapods-trunk'
  cp_gem 'cocoapods-try',         'cocoapods-try'
  cp_gem 'molinillo',             'Molinillo'
  cp_gem 'nanaimo',               'Nanaimo'
  cp_gem 'xcodeproj',             'Xcodeproj'
   
  gem 'cocoapods-dependencies', '~> 1.0.beta.1'
  # ...
  # Integration tests
  gem 'diffy'
  gem 'clintegracon'
  # Code Quality
  gem 'inch_by_inch'
  gem 'rubocop'
  gem 'danger'
end

group :debugging do
  gem 'cocoapods_debug'

  gem 'rb-fsevent'
  gem 'kicker'
  gem 'awesome_print'
  gem 'ruby-prof', :platforms => [:ruby]
end

# 用于实现组件的热插拔
def cp_gem(name, repo_name, branch = 'master', path: false)
  return gem name if SKIP_UNRELEASED_VERSIONS
  opts = if path
           { :path => "../#{repo_name}" }
         else
           url = "https://github.com/CocoaPods/#{repo_name}.git"
           { :git => url, :branch => branch }
         end
  gem name, opts
end
```
{% endcode %}

![CocoaPods项目的依赖](<../.gitbook/assets/image (6) (1).png>)

#### 1.1 [CLAide](https://github.com/CocoaPods/CLAide)

是个[命令行解释器](https://www.wikiwand.com/en/Command-line\_argument\_parsing)，负责解析pod命令。

也可以封装常用的脚本，将其打包成常用的命令行小工具。

#### 1.2 [cocoapods-core](https://github.com/CocoaPods/Core)

用于cocoaPods中模版的解析（Podfile、.podspec、.lock）

#### 1.3 [cocoapods-downloader](https://github.com/CocoaPods/cocoapods-downloader)

用来下载源码的小工具，支持各种类型的版本管理工具，包括 HTTP / SVN / Git / Mercurial。可以处理 `tags`，`commits`，`revisions`，`branches` ， `zips` 及其他版本管理用到的操作。

#### 1.4 [Molinillo](https://github.com/CocoaPods/Molinillo/blob/master/ARCHITECTURE.md)

用来处理CocoaPods、Bundler、RubyGems的依赖。是一个具有前向检查的回溯算法。

#### 1.5 [Xcodeproj](https://github.com/CocoaPods/Xcodeproj)

可通过 Ruby 来操作 Xcode 项目的创建和编辑等。可友好的支持 Xcode 项目的脚本管理和 libraries 构建，以及 Xcode 工作空间 `.xcworkspace`和配置文件 `.xcconfig` 的管理。

#### 1.6 cocoapods-plugins

插件管理工具，其中有 `pod plugin` 全套命令，支持对于 CocoaPods 插件的`list、search、create`功能。

### 2 CocoaPods的工作流程

#### 2.1 命令入口（`pod`命令）

`pod`命令在`/bin`目录下。

用于唤起 CocoaPods，加载CocoaPods 的安装目录 `cocoapods/bin`下的 `/pod` 文件。

#### 2.2 pod install

![pod install流程](<../.gitbook/assets/image (7).png>)

*   #### Install 环境准备（prepare）

    处理版本一致性，对pod目录建立子目录结构，检测并执行 PluginManager 中的 pre-install 方法。

    如果检测出当前目录是 Pods，直接 raise 终止。
*   #### 解决依赖冲突（resolve\_dependencies）

    通过 `Podfile`、`Podfile.lock` 以及沙盒中的 `manifest` 生成 _Analyzer_ 对象。_Analyzer_ 内部会使用 `Molinillo::DependencyGraph` 图算法解析得到一张依赖关系表。

    analyze 的过程中有一个 _pre\_download_ 的阶段，**不属于依赖下载**过程，而是在当前的**依赖分析**阶段。主要是解决当我们在通过 Git 地址引入的 Pod 仓库的情况下，系统无法从默认的 Source 拿到对应的 Spec，需要直接访问我们的 Git 地址下载仓库的 zip 包，并取出对应的 `podspec` 文件，从而进行对比分析。
*   #### 下载依赖文件（download\_dependencies）

    创建沙盒目录的文件访问器，解析沙盒中的各种文件调，然后调用对应 Pod 的 `install!` 方法进行资源下载。
*   #### 验证 targets (validate\_targets)

    验证之前流程中的产物 (pod 所生成的 Targets) 的合法性。

    * 验证是否有重名的 `framework`
    * 验证动态库中是否有静态链接库 (`.a` 或者 `.framework`) 依赖，[issue](https://github.com/CocoaPods/CocoaPods/issues/3289)
    * 确保 Swift Pod 的 Swift 版本正确配置且互相兼容的。
    * 检测 Swift 库的依赖的 Objective-C 库是否支持了 module。
      * Swift 库在解析后会生成对应的 `modulemap` 和 `umbrella.h` 文件，这是 LLVM Module 的标配。
      * Objective-C 也是支持 [LLVM Module](http://clang.llvm.org/docs/Modules.html)。当我们以 Dynamic Framework 的方式引入 Objective-C 库时，Xcode 支持配置并生成 header，**但是当以静态库 .a方式引入时， 需要自己编写对应的 `umbrella.h` 和 `modulemap`**，如果Objective-C 库启用了 `modular_headers`，则 CocoaPods 会为我们生成对应 `modulemap` 和 `umbrella.h` 来支持 LLVM Module。 参考：[CocoaPods 1.5.0 — Swift Static Libraries](http://blog.cocoapods.org/CocoaPods-1.5.0)。
      * ![](<../.gitbook/assets/image (2).png>)
*   #### 生成工程 (Integrate)

    将之前版本仲裁后的所有组件通过 Project 文件的形式组织起来，并且会对 Project 中做一些用户指定的配置。

    时间开销也是比较大，可以优化。
*   #### 写入依赖 (write\_lockfiles)

    将依赖更新写入 `Podfile.lock` 和 `Manifest.lock`
*   #### 结束回调（perform\_post\_install\_action）

    为所有插件提供 post-installation 操作以及 hook。

![pod install过程中各组件大作用](<../.gitbook/assets/image (5).png>)



