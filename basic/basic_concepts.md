# Git 基本概念

理解 Git 的基本原理. 有利于我们快速上手，更有效地利用这个工具，了解 Git 的设计哲学

## 1. 基本概念

在我们调用`git init`后项目的根目录就会出现一个`.git`目录，这个目录包含了所有版本库的信息。

```shell
hello
  .git
    ├─hooks                     -- 定义钩子
    │   ├─pre-commit
    │   ├─pre-commit.sample
    │   ├─post-commit
    │   ├─post-commit.sample
    │   └─...
    ├─info
    │  └─exclude
    ├─objects                   -- 对象库
    │  ├─....
    │  ├─info
    │  └─pack                   -- 对象打包压缩文件
    │─refs                      -- 对象引用
    │   ├─heads
    │   ├─remotes
    │   └─tags
    ├─HEAD
    ├─config
    ├─desciption
    ├─index                     -- 索引
    └─...
```

Git 主要有两种数据结构：`对象库(Object Store)`和`索引(Index)`。

在 Git 中，每个文件、目录、提交以及标签都使用“对象”来进行存储。

### 1.1 文件对象

文件使用块(blob)进行表示， 块是一个二进制的序列. Git 的一个关键的概念就是`“内容驱动的”`，而不是像 SVN 基于文件和目录的布局，
Git 追踪的是内容而不是文件。块中不会存储任何关于文件的元信息， 这些元信息在`目录对象`中描述。

所以, 如果两个文件的内容完全一致，Git 只会使用一个块进行存储。这样可以节省存储空间. 例如：

```shell
# 创建两个内容一模一样的文件
$ echo "hello world" > foo.txt
$ echo "hello world" > bar.txt

# 在没有添加到索引(暂存区)或者版本库之前，Git是不会创建这些对象, 关于工作区/索引版本库
# 的概念， 在下文中描述. 现在将文件添加到索引(索引可以理解为提交到版本库之前的暂存区)
$ git add foo.txt bar.txt

# 列出索引区中创建的文件对象
# ls-files命令类似于`ls`命令
$ git ls-files --stage
100644     80993781b54ed1b81e47a31e6427940c1a9deafb 0       bar.txt
100644     80993781b54ed1b81e47a31e6427940c1a9deafb 0       foo.txt
# 文件权限  SHA1 散列                                        文件名

# 可以看出文件块已经添加到git对象库中;
# git使用SHA1的第一个字节作为目录，来避免把很多文件放在同一个目录中(一些文件系统会变卡)
# git 使用zlib压缩存储这些对象
$ tree .git/objects
├─80
│ └─993781b54ed1b81e47a31e6427940c1a9deafb
├─info
└─pack
```

从上面的执行结果可以看出，`foo.txt`和`bar.txt`的散列值是一样的。对于 sha1 编码来说，只要输入的内容一致， 输出的内容也是一致的。所以也说明这两个文件是共享同一个块的。

> 关于 SHA1 散列值:<br/>
>
> 上述的 SHA1 散列值可以认为是`对象ID`，通常表示为一个 40 位的 16 进制字符串. Git 会对`对象的内容`应用 SHA1 得到. <br/>
> 对于`文件`来说应用的内容是`块`，即文件的二进制表示;
> 对于`目录`来说， 应用的内容是`目录信息`; <br/>
> 对于`提交`来说，应用的内容是提交信息、目录对象、作者、提交时间、父提交等元数据; <br/>
> 对于`标签`来说, 应用的内容是标签标注和关联的提交.<br/>
> 总的来说这个散列值可以唯一地标志一个`对象`. SHA1 值在 git 的命令用的非常广泛, 但是如果要输入完整的
> SHA1 值是非常麻烦的事情，git 支持输入 SHA1 值的前缀。如下面的 `cat-file` 命令所示

我们可以使用`cat-file`命令来查看对象的内容。`cat-file`类似于`cat`命令:

```shell
# 查看对象类型；输入SHA1值的前缀，只要不冲突即可
$ git cat-file -t 8099378
blob

# 查看对象的内容
$ git cat-file -p 8099378
hello world
```

每一个文件的版本都会使用一个块进行表示. 每当文件变动，Git 就会为这个文件(完整文件)计算一个新的 SHA1 值. 并将完整文件的内容保存为块。看到这里， 你可能像我一样会有疑问，为什么 git 不保存差异? 而是保存完整的文件副本，这不是浪费空间吗？

```shell
# 追加一行到foo.txt
$ echo "hello gzb" >> foo.txt
$ git add foo.txt
# 我们看到foo.txt的SHA1已经变化了, bar.txt依旧引用80993
$ git ls-files --stage
100644 80993781b54ed1b81e47a31e6427940c1a9deafb 0       bar.txt
100644 aa1284b52c5fa5beca04b102d342261d01db289d 0       foo.txt
$ git cat-file -s aa1284
46                        # foo.txt 文件大小
$ git cat-file -s 889937
28                        # bar.txt 文件大小
```

实际上，Git 会这么做, 只不过不是马上做。上面存储对象格式称为`松散格式(loose)`. 像编程语言的垃圾回收机制一样， Git
会时不时地将多个对象`打包(pack)`成为一个"`包文件(packfile)`", 以节省空间和提高效率。

Git 首先会定位内容非常相似的`块`， 为他们之一存储整个内容， 之后再计算相似文件之间的差异，并只存储差异。例如上面的 foo.txt 和
bar.txt 文件，git 可能会存储整个 foo.txt 的内容，然后再单独存储两者之间的差异.

相对于 SVN 这类基于文件的 VCS，Git 内容驱动的方式，更能有效地节省空间，因为块是全局的。而 SVN 只能为同一目录的同名文件计算差异.

> 当版本库中保存了太多`松散对象`, 或者你向远程服务器推送， 以及手动调用`git gc`命令时，git 会进行打包。比如
>
> ```shell
> $ git push origin master
> Counting objects: 20, done.
> Delta compression using up to 4 threads.
> Compressing objects: 100% (20/20), done.
> Writing objects: 100% (20/20), 1.66 KiB | 340.00 KiB/s, done.
> Total 20 (delta 17), reused 0 (delta 0)
> To ssh://code.ejiahe.com:25522/WEB/sms2.0-client.git
>    95c66b1..9c9d863  master -> master
> ```

```shell
# 垃圾回收; 打包
$ git gc

# 我们发现之前的两个对象文件都不见的。因为他们被压缩打包进了包文件
# .pack 文件包含了所有从文件系统移除的对象
# .idx 包含了这些对象在.pack包文件中的偏移信息. 可以用于快速定位一个指定的对象
$ tree .git/objects
├─info
└─pack
  ├─pack-41941a802863135106a20d7f3663ed393ed33b3a.idx
  └─pack-41941a802863135106a20d7f3663ed393ed33b3a.pack

# 可以通过verify-pack 命令来查看索引文件. 这里举一个其他项目的例子
$ git verify-pack -v .\.git\objects\pack\pack-xxx.idx
22e70521f125689c969e1e94a9ca3fcb61b579ed blob   20 33 3546
41ec12446a664f43df51df2d953c5c04b34b265e blob   872 511 3579
1038dc0ce035045626b2efa60b16883a3507ad5b blob   80 95 4090 1 41ec12446a664f43df51df2d953c5c04b34b265e

# 1038dc为差异内容，其完整内容是41ec12.
# 在git完整内容一般是最新文件内容。 因为最新文件内容访问的频率较高
```

### 1.2 目录对象

上面大概讲述了一个文件在 git 中的存储方式。文件的元信息就存储在一个称为"`目录树(tree)`"的对象中。估计你已经想到这个对象会包含什么内容：

* 目录包含的块, 以及块的文件信息
* 子目录树

> 注意这里没有包含，目录的路径信息和目录名。这个道理和文件块对象一样，git 是`内容驱动的`. 关于目录的路径和目录名，有父目录对象描述。<br/>
> 这样，像文件块一样，目录在项目的任意一个位置，目录的内容不变（SHA1 不变）.
> 另外如果两个不同路径或名字的目录包含的文件和目录是一样的, 这两个目录共享同一个目录对象

那么项目的目录树可以表示为：<br/>
![结构图](TODO)

跟文件对象一样，Git 会为目录树计算一个 SHA1 值， 这个 SHA1 值可以用来高效地比对。 比如比较 foo 目录修改前和修改后的 SHA1 值，如果 SHA1 值相等，就说明 foo 目录内容没有变化，没有必要向下递归比对了。

上面， 我们将`foo.txt`和`bar.txt`文件添加到索引(暂存区)中, Git 还没有为他们创建目录对象。Git 会在`用户提交`或者手动调用底层的`write-tree`命令之后创建一个新的目录对象。

> 这个目录树是从当前索引中捕获的. 而不是工作区

```shell
# 立即创建一颗目录树. 这个命令返回这棵树的SHA1值
$ git wirte-tree
30c06f321a0a9838685e85ad87c330d067b59515

# 同样可以使用cat-file命令来查看目录树的内容
$ git cat-file -t 30c06
tree
$ git cat-file -p 30c06
100644 blob 80993781b54ed1b81e47a31e6427940c1a9deafb    bar.txt
100644 blob 707324cde0d21955bb8e7bfdaaa10007dd8d54fe    foo.txt



# 现在将foo.txt和bar.txt移到一个新的目录
$ mv *.txt dir/

# 添加到索引
$ git add .

# 可以看到foo.txt, bar.txt的块对象没有变化
$ git ls-files --stage
100644 80993781b54ed1b81e47a31e6427940c1a9deafb 0       dir/bar.txt
100644 707324cde0d21955bb8e7bfdaaa10007dd8d54fe 0       dir/foo.txt

# 再创建目录树对象: 因为目录结构发生了变化，所以重新创建一个新的对象
$ git wirte-tree
0c1564cd525d98a691c29779569348b3bdc0a8ce

# 在这里你可以看到dir目录对象的SHA1和上面的根目录的SHA1是一样的！
# 因为dir和之前的根目录内容是一样的，所以git复用了这个目录对象
$ git cat-file -p 0c1564
040000 tree 30c06f321a0a9838685e85ad87c330d067b59515    dir
```

### 1.3 提交

![提交图](TODO)

## 一个文件的提交过程
