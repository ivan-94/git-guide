# 集中式工作流

集中式工作流， 或者称为`SVN工作流`, 如果你是从 SVN 迁移过来的，那么你已经熟知集中式工作流了。和 SVN 一样， 集中式工作流以一个`中央仓库`, 所有成员的本地版本库的修改都提交到这个分支。这种工作流的特点有:

* 使用`中央仓库`作为本地版本库同步的唯一入口。
* 只有一个`master`分支

![集中式工作流配图](images/centralized1.png)

尽管和 SVN 类似，但和 SVN 相比，Git 还有以下优势：

* 每个本地项目都是一个完整的版本库， 有一定的独立性， 不需要依赖于中央仓库, 我们可以自己选择合适的时间点进行同步。而 SVN 可能仅仅是一个目录的副本, 中央仓库挂了，将无法提交
* Git 有更强大的分支和合并模型

## 1. 适用场景

* 如果刚从 SVN 迁移过来，使用集中式工作流，让团队无需去适应一个全新的开发流程；或者可以作为`过渡方案`
* 工作流简单自然。学习成本低，适合小型项目

## 2. 示例

### 2.1 创建中央仓库

我们可以从 `gitlab` 中创建一个项目, 或者直接通过 ssh 登录到服务器进行创建：

```shell
$ ssh user@myhost
# 创建一个裸仓库

# 裸仓库是指没有工作目录的仓库，也就是说只有一个`.git`文件夹。
# 裸仓库一般在服务端使用, 因为我们不会在服务端进行实际的开发
$ git init --bare /path/to/myrepo.git
```

![裸仓库](images/bare-repo.png)

创建完仓库之后, 团队成员就可以将其克隆到本地进行开发：

```shell
$ git clone ssh://user@myhost/path/to/myrepo.git
```

![克隆](images/clone.png)

### 2.2 开发和提交

现在就可以在工作区中添加和编辑文件了. 每一个提交都需要经过暂存和提交

```shell
# 新建一个文件
$ echo "hello world" > "hello.txt"

# 添加到索引
$ git add "hello.txt"

# 提交到版本库
$ git commit -m "初始化项目， 新建hello.txt"
```

### 2.3 推送和拉取

到目前为止，变更还在本地版本库，我们需要通过 `push` 命令，将变更推送到远程版本库:

![推送配图](images/push.png)

```shell
# 查看远程版本库
# 在git clone一个远程版本库时，git已经将远程版本库设置好了，默认命名为origin
$ git remote
origin


# 目前远程分支为空
$ git remote show origin
* remote origin
  Fetch URL: ssh://user@myhost/path/to/myrepo.git
  Push  URL: ssh://user@myhost/path/to/myrepo.git
  HEAD branch: (unknown)
  Local branch configured for 'git pull':
    master merges with remote master

# 推送到远程版本库的master分支
$ git push origin master
```

目前的分支状态， 可以看到 master 和远程分支都指向了同一个提交对象：

![after push](images/push1.png)

现在，其他成员也要将你的提交拉取下来, 拉取使用`pull`命令

![拉取配图](images/pull.png)

```shell
(Jim)$ git pull origin master
```

### 2.4 冲突解决

假设你和另外一个同事 Jim 正在同时开发这个项目. 你修改了多次 hello.txt, 提交了多次代码到远程库，而 Jim 在创建另一个文件开发完新功能准备提交. 现在的提交图为：

![提交图](images/commit1.png)

```shell
# Jim很有成就感地准备提交到远程版本库
(Jim)$ git push origin master
error: failed to push some refs to '/path/to/myrepo.git'
hint: Updates were rejected because the tip of your current branch is behind
hint: its remote counterpart. Merge the remote changes (e.g. 'git pull')
hint: before pushing again.
hint: See the 'Note about fast-forwards' in 'git push --help' for details
```

什么鬼。远程版本库拒绝了，原因根据上图可以得知，因为远程版本库比 Jim 的本地版本库的提交记录要新。以前我们使用 SVN，在提交之前我们都习惯"`svn update`"一下， 保证版本库处于最新的状态，并在本地解决好冲突再提交。
git 也是如此。Git 推荐在提交前使用`git pull`拉取最新的版本库。

> 你也可以将远程版本库和你的版本库视作两个独立分支， 那么`git pull`和`git push`的过程就是分支合并的过程. 实际上也是如此. 关于分支合并的细节，见[分支操作](operation.md)一节

```shell
# 拉取最新远程分支, 合并之后会生成一条新的“合并提交”记录
(Jim)$ git pull
(Jim)$ git status
On branch master
Your branch and 'origin/master' have diverged,
and have 2 and 2 different commits each, respectively.
  (use "git pull" to merge the remote branch into yours)

All conflicts fixed but you are still merging.
  (use "git commit" to conclude merge)

Changes to be committed:

	modified:   hello.txt

(Jim)$ git commit -m "合并远程分支"
# 如果有冲突，修复了冲突之后就可以重新提交了
(Jim)$ git push
```

现在的提交图为：

![合并后提交图](images/commit2.png)

### 2.5 冲突解决 2

一般情况下, 分支合并都会产生一个`合并提交记录`,  如果你要获得线性的历史记录的话, 可以使用`rebase`
策略来处理合并两个分支.

`rebase`中文  翻译过来就是`变基`, 更通俗的将就是 "改变分支的基础". 根据上图,我们可以知道
Jim 的本地分支是基于`初始化项目, 新建hello.txt`这个提交为基础的. 通过 git 的`rebase`命令,  是可以篡改分支的基础的:

```shell
# 打翻从来, 使用rebase命令进行合并
(Jim)$ git pull --rebase
(Jim)$ git push
```


我们使用 rebase 命令, 让 Jim 的`本地分支`的`基础`基于`远程版本库最新提交`,  那么现在的提交图, 看起来就是这样:

![合并后提交图](images/commit3.png)

使用 rebase 的优点就是不会产生多余的提交记录; 但是会丢失分支的轨迹, 也就是你在 GUI 上看到的分支只有一条分支, 不会有普通合并那样的分支分叉和合并记录.

> rebase 模式解决冲突: <br/>
> 如果在 rebase 的过程中出现冲突, git 会暂停下来, 让你解决冲突.
> 解决完冲突之后, 再添加到索引:
>
> ```shell
> # 解决完冲突
> $ git add 冲突的文件
> $ git rebase --continue
> ```
>
> 如果要中断 rebase, 可以执行:
>
> ```shell
> # 中断rebase
> $ git rebase --abort
> ```
