---
description: 微操大师上线。
---

# Git秀

## 文件管理

![git 存储区](<../.gitbook/assets/image (1) (1) (1).png>)

### 1.工作区->暂存区

`git add <file>`

`git stage`

### 2.暂存区->版本库

`git commit -m "描述"`

`git commit --amend //和上一次的提交合并`

### 3.工作区->版本库

`git commit -a`

### 4.暂存区->工作区

`git checkout <file>`

* 工作区的修改会被覆盖
* 工作区新增的文件不会被删除

### 5.版本库->暂存区

`git reset <file>`

`git reset --mixed HEAD <file>`

* 暂存区中的数据会被覆盖
* 工作区中的数据不受影响

### 6.版本库->工作区

`git checkout HEAD <file>`

* 工作区和暂存区都会被覆盖

## 查看历史

`git log`：查看当前HEAD的历史

`git reflog`：查看所有的操作历史

## 当前在origin/develop分支对应的本地分支下工作，要修改即将发版的origin/release分支上的bug：

1. `checkout origin/release`：会在本地创建一个对应的分支“release”
2. 在上面修改
3. `git push HEAD:origin/release`：将其提交到远程的release分支

## 完成功能a并push到远程，然后完成功能b并push到远程，此时需要修改功能a：

1. 修改完成后`commit`
2. `git rebase -i HEAD~3`：会在vim打开当前head之前到三次commit，此时最下面的是最新的提交，
3. 将第三行调到第二行：dd剪切当前行，p将粘贴板复制在当前行的下一行
4. 将第三行开头的commit改成fix
5. wq退出

## 先完成功能a，合到远程，然后完成功能b，合到远程，此时这两个功能不需要发版，要从远程库撤回

1. revert功能b，再次提交
2. revert功能a，再次提交
3. revert “revert a”
4. revert “revert b”
5. 等到要发版在把这两个revert提交
