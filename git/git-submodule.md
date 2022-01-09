# Git Submodule

> [https://git-scm.com/book/zh/v2/Git-%E5%B7%A5%E5%85%B7-%E5%AD%90%E6%A8%A1%E5%9D%97](https://git-scm.com/book/zh/v2/Git-%E5%B7%A5%E5%85%B7-%E5%AD%90%E6%A8%A1%E5%9D%97)

适用场景：对于两个独立的项目，想在一个项目中使用另一个。

作用：子模块允许将一个Git仓库作为另一个Git仓库的子目录，可以将另一个仓库克隆到自己的项目中，同时还能保持独立的提交。

### 使用

添加子模块：

```
git submodule add https://XXXXX <PATH>
```

会创建 `.gitmodules` 文件：

```ini
[submodule "Name"]
	path = "Path"
	url = https://github.com/XXX
```



`git diff`时在主项目查看子模块的更改详情：

```
git diff --submodule
```

如果不想每次都输入`--submodule`，可以将 `diff.submodule` 设置为 “log” 来将其作为默认行为：`git config -f .gitmudules diff.submodule log`



`git status`时显示子模块的更改摘要：配置选项 `status.submodulesummary`：

```
git config -f .gitmudules status.submodulesummary 1
```

### 克隆含有子模块的项目

直接克隆项目，会创建子模块的目录，但是不会下载任何子模块文件。

需要运行：

`git submodule init` ：初始化本地配置文件

`git submodule update` ：下载子模块数据，并checkout合适的提交



有两种简化上述步骤的操作：

1. 克隆主项目时加上`--recurse-submodules`参数：`git clone --recurse-submodules https://XXX`
2. 直接克隆主项目，然后运行：`git submodule update --init` 。如果子模块还包含子模块，还可以加上`--recurse-submodules`参数。

### 在含有子模块的项目工作

拉取上游修改：

```
git submodule update --remote <SUBMODULE-NAME>
```

默认会拉取master分支，想拉取其他分支，可以在`.gitmodules` 文件中设置：

```
git config -f .gitmodules submodule.NAME.branch BRANCH-NAME
```



如果主项目提交了刚拉取的新子模块，需要加上`--init` 选项；

如果子模块有嵌套的子模块，需要加上 `--recursive` 选项。



让 Git 总是以 `--recurse-submodules` 拉取：

```
git config -f .gitmodules submodule.recurse true
```



子模块的URL发生改变：

```bash
# 将新的 URL 复制到本地配置中
git submodule sync --recursive
# 从新 URL 更新子模块
git submodule update --init --recursive
```

### 子模块拉取上游改动

运行 `git submodule update` 时， 子仓库会留在一个称作“游离的 HEAD”的状态。这意味着没有本地工作分支跟踪改动。也就意味着即便你将更改提交到了子模块，这些更改也很可能会在下次运行 `git submodule update` 时丢失。

解决：

```
git submodule update --remote --merge
```

或

```
git submodule update --remote --rebase
```

### push子模块的改动

可以让 Git 在推送到主项目前检查所有子模块是否已推送：

```
git push --recurse-submodules=check
```



可以让 Git 在推送到主项目前，进入子模块并执行推送操作：

```
git push --recurse-submodules=on-demand
```



配置推送默认行为：

```bash
git config -f .gitmodules push.recurseSubmodules check/on-demand
```

### merge子模块的改动

如果你和其他人同时改动了一个子模块，会引起冲突。

如果一个提交是另一个的直接祖先（一个快进式合并），那么 Git 会简单地选择之后的提交来合并；否则，Git 甚至不会尝试去进行一次简单的合并。&#x20;



解决冲突：

1. 进入子模块目录，解决冲突
2. 回到主项目，更新子模块索引
3. 提交

### 技巧

#### `foreach`：同时处理所有子模块：

```
git submodule foreach 'git stash'
git submodule foreach 'git checkout -b featureA'
git diff; git submodule foreach 'git diff' # 非常有用！！
```



#### 别名：

适用语命令很长但又不能设置选项作为它们的默认选项。

```
git config alias.sdiff '!'"git diff && git submodule foreach 'git diff'"
git config alias.spush 'push --recurse-submodules=on-demand'
git config alias.supdate 'submodule update --remote --merge'
```



### 问题

#### 主项目中的不同分支对应的子模块的commit不一样时：

`git checkout --recurse-submodules` 来切换分支，能保证不同的分支下，子模块处于正确的commit节点。



#### 将子目录转换为子模块时，需要特别注意：

* 如果删除子目录，然后运行 `submodule add XXX`，会报错`'XXX' already exists in the index`
* 必须要先取消暂存子目录。 然后才可以添加子模块：`git rm -r XXX`



* 如果切换回的分支中，那些文件还在子目录而非子模块中，会报错：`error: The following untracked working tree files would be overwritten by checkout:`
* 可以通过 `git checkout -f master` 来强制切换，但是要小心，如果其中还有未保存的修改，这个命令会把它们覆盖掉。
* 当你切换回来之后，因为某些原因你得到了一个空的子目录，并且 `git submodule update` 也无法修复它。 你需要进入到子模块目录中运行 `git checkout .` 来找回所有的文件。 你也可以通过 `submodule foreach` 脚本来为多个子模块运行它。

⚠️注意：子模块会将它们的所有 Git 数据保存在顶级项目的 `.git` 目录中。所以摧毁一个子模块目录并不会丢失任何提交或分支。
