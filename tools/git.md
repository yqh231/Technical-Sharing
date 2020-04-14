+++

## 提交信息写错或者commit后临时又要添加文件的操作?
那么可以这样：
  先提交forgotten_file

    $ git  add forgoten_file

  完了再

    $ git commit --amend

这样就可以把忘记的补上了，同时还可以修改说明

`git commit --amend`命令需要的是提交的文件快照和之前一样，

所以上面的添加文件要在`git commit`之前。

## 总有些文件会一不小心add进暂存区内，取消暂存区的文件?
在git status状态会有相应的信息提示，是git reset HEAD <file>

## 已经commit了，需要取消跟踪？
总有不小心跟踪了不该跟踪的文件的时候，这个时候需要从cached删除文件
`git rm --cached file`就派上用场了，如果是整个目录，那么这时候
可以参考rm命令的参数。。 -r 都派上用场了你没看错。。
`git rm -r --cached files/`极其好用。。整个文件夹都被干掉了。

## 公司变土豪了，买github私有库了？
转换仓库，从gitlab到github

  `git remote add upstream ssh://git@gitlab.xx.:workspace/project/lau-jay.git`
  `git remote set-url origin git@github.com:xxx/lau-jay.git`

note: .git前的lad-jay是Repositories，别误解

## 想做点commit点省的误删，但是那么多commit又很烦？
    git rebase -i 32db1a7e1....

先`git log`找到你不想要保留的之前那个的hash，然后使用上面这条命令。
就能进入一个编辑状态，然后再依据编辑状态的提示处理就ok了。很简单的。

## 经常忘记你对当前的代码干了啥git操作?
使用版本管理工具经常发生的一件事是，你对代码干了啥可以清楚的知道，但是
维护过程中你使用了什么git操作，不知道了。这时候有几个做法，一个是history命令
可以看出你之前的操作，但是如果时间久一点，那么命令一多你就难找了。
`git reflog`
可以列出你之前的所有git操作，但是这个也是有时限的，好像是一个月，不过已经够了。

## 代码出bug了。不知道谁干的？
有时候会有冲突说，这bug是xxx写的，然后xxx不服。有版本管理很简单，
能精确到某一行代码的作者是谁。
`git blame file`
就能知道谁干的了。

## 查看log想倒过来？
看代码的时候我喜欢一个commit看过去，但是有些commit多了log会很长于是log的参数
`--reverse`就派上用场了:
`git log --reverse`
这命令会把commit按时间线从第一个开始递增展示

##  只想拉某一个分支？
`git clone -b <branch> <remote_repo>`

## 分支重命名？
重命名分支
`git branch -m <oldname> <newname>`
重命名当前分支
`git branch -m <newname>`

## 命令行删除远程分支？
查看远程分支
`git branch -r `
删除远程分支
`git push origin :want_delete_branch_name`


## 拆分子模块，并修改依赖？
`git submodule add 仓库地址 路径`

git clone 的工程带有submodule时,因为并不会自动去拉取子模块
需要执行：
`git submodule update --init --recursive`


## .gitignore的写法？
* 所有空行或者以注释符号`#`开头的都会被Git忽略
* 可以使用标准的glob模式匹配(shell所使用的简化略的正则表达式)
* 匹配模式最后跟反斜杠(`/`)说明要忽略的是目录
* 要忽略指定模式以外的文件或目录，可以在模式前加上惊叹号取反

## fork了但是人家更新了不知道如何同步fork库?
经常会记不住同步的方式，特地记录下其实很简单
`git remote add upstream URL`
`git fetch upstream`
`git merge upstraem/upstream`

## 特殊情况下CI不触发了，如何再次触发？

`git commit --allow-empty -m "Trigger notification"`

commit是允许没任何变更的