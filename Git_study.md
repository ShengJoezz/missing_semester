---
author: joe
content: git的学习
tags: 
- cs_tips
---

# 1.为什么要用Git

实际上在代码编写过程中，我们需要对旧版本进行控制，包括新版本不同功能的研发可能是并行的，我们需要对开发过程中的文件进行版本的管理。

现代的版本控制系统可以帮助您轻松地（甚至自动地）回答以下问题：

- 当前模块是谁编写的？
- 这个文件的这一行是什么时候被编辑的？是谁作出的修改？修改原因是什么呢？
- 最近的1000个版本中，何时/为什么导致了单元测试失败？

尽管版本控制系统有很多， 其事实上的标准则是 Git。

# 2.Git的数据类型

## 快照

Git 将顶级目录(root)中的文件和文件夹作为集合，并通过一系列快照(snapshot)来管理其历史记录。

文件被称作Blob对象（数据对象），也就是一组数据。
目录则被称之为“树”(tree)，它将名字与 Blob 对象或树对象进行映射（使得目录中可以包含其他目录）。
快照(snapshot)则是被追踪的最顶层的树。(也就是说，snapshot是在不同的状态的root，实际也就是所有文件)

例如，一个树看起来可能是这样的：

```
<root> (tree)
|
+- foo (tree)
|  |
|  + bar.txt (blob, contents = "hello world")
|
+- baz.txt (blob, contents = "git is wonderful")
```

这个顶层的树包含了两个元素，一个名为 "foo" 的树（它本身包含了一个blob对象 "bar.txt"），以及一个 blob 对象 "baz.txt"。

## Git的数据结构

那么我们很容易就想到一个思路，即我把不同时期的快照依次排列，这样我就可以挑选我想要的历史代码了，事实上，线性历史记录是一种最简单的模型，它包含了一组按照时间顺序线性排列的快照。不过处于种种原因，Git 并没有采用这样的模型，比如说，我对于同一个老文件可能同时会有两个修改需求，这个并行同时产生了两个相同的文件，因此我们无法用简单的线性时间关系来排列这些文件了。

在 Git 中，历史记录是一个由快照组成的有向无环图。

有向无环图，实际上就是有一定方向(这是obvious的，因为文件会有时间顺序，但是不会产生循环，因为时间是单向的)。或者说这代表 Git 中的每个快照都有一系列的“父辈”，也就是其之前的一系列快照。注意，快照具有多个“父辈”而非一个，因为某个快照可能由多个父辈而来。例如，经过合并后的两条分支。

在 Git 中，这些快照被称为“提交”(commit)。通过可视化的方式来表示这些历史提交记录时，看起来差不多是这样的：

```
o <-- o <-- o <-- o
            ^  
             \
              --- o <-- o
```

上面是一个 ASCII 码构成的简图，其中的 `o` 表示一次提交（快照）。

箭头指向了当前提交的父辈（**这是一种“在...之前”，而不是“在...之后”的关系**，这代表一种追溯的含义），即左边的是最早的，右边是最晚的。在第三次提交之后，历史记录分岔成了两条独立的分支。这可能因为此时需要同时开发两个不同的特性，它们之间是相互独立的。开发完成后，这些分支可能会被合并并创建一个新的提交，这个新的提交会同时包含这些特性。新的提交会创建一个新的历史记录.


Git 中的提交是不可改变的。但这并不代表错误不能被修改，只不过这种“修改”实际上是创建了一个全新的提交记录。而引用（参见下文）则被更新为指向这些新的提交。

## 数据模型及其伪代码表示

以伪代码的形式来学习 Git 的数据模型，可能更加清晰：

```
// 文件就是一组数据,这里array<byte>是代表一系列的字节组成的数组，也就是二进制文件，这是一个常见的作法
type blob = array<byte>

// 一个包含文件和目录的目录，map是一种数据结构，用于存储键值对，这里即为某个字符串"string"对应"tree or bolb"
type tree = map<string, tree | blob>

// 每个提交都包含一个父辈，元数据和顶层树,因此提交中我们包含四个内容，其中author和message我们称之为metadata(元数据)，而snapshot就是`root`对应的tree,另外还要把父辈的提交也提交上去。
type commit = struct {
    parent: array<commit>
    author: string
    message: string
    snapshot: tree
}
```

这是一种简洁的历史模型。


## 对象和内存寻址

Git 中的对象可以是 blob、树或提交：

```
type object = blob | tree | commit
```

由于每个object都会相对很大，因此我们采用`hash函数`的方式，对其进行简化。

Git 在储存数据时，所有的对象都会基于它们的 [SHA-1 哈希](https://en.wikipedia.org/wiki/SHA-1) 进行寻址。

```
objects = map<string, object>
//store函数利用hash函数对object进行简化，生成某个40位的数字，这个数字就是object的key值
def store(object):
    id = sha1(object)
    objects[id] = object
//如果想要取用这个object，只需要用这个object的id。并在字典查询即可
def load(id):
    return objects[id]
```

Blobs、树和提交都一样，它们都是对象。当它们引用其他对象时，它们并不会重复存储，只会存储一次，后续的使用，利用它们的哈希值作为引用。

例如，[上面](#snapshots)例子中的树（可以通过 `git cat-file -p 698281bc680d1995c5f4caaf3359721a5a58d48d` 来进行可视化），看上去是这样的：

```
100644 blob 4448adbf7ecd394f42ae135bbeed9676e894af85    baz.txt
040000 tree c68d233a33c5c06e0340e4c224f0afca87c8ce87    foo
```

树本身会包含一些指向其他内容的指针，例如 `baz.txt` (blob) 和 `foo`
(树)。如果我们用 `git cat-file -p 4448adbf7ecd394f42ae135bbeed9676e894af85`，即通过哈希值查看 baz.txt 的内容，会得到以下信息：

```
git is wonderful
```

## 引用

现在，所有的快照都可以通过它们的 SHA-1 哈希值来标记了。但这也太不方便了，谁也记不住一串 40 位的十六进制字符。

针对这一问题，Git 的解决方法是给这些哈希值赋予人类可读的名字，也就是引用（references）。引用是指向提交的指针。与对象不同的是，它是可变的（引用可以被更新，指向新的提交）。例如，`master` 引用通常会指向主分支的最新一次提交。

```
references = map<string, string>

def update_reference(name, id):
    references[name] = id

def read_reference(name):
    return references[name]

def load_reference(name_or_id):
    if name_or_id in references:
        return load(references[name_or_id])
    else:
        return load(name_or_id)
```

这样，Git 就可以使用诸如 "master" 这样人类可读的名称来表示历史记录中某个特定的提交，而不需要在使用一长串十六进制字符了。

有一个细节需要我们注意， 通常情况下，我们会想要知道“我们当前所在位置”，并将其标记下来。这样当我们创建新的快照的时候，我们就可以知道它的相对位置（如何设置它的“父辈”）。在 Git 中，我们当前的位置有一个特殊的索引，它就是 "HEAD"。

# 3.Git的实战使用

首先我们创建一个新的文件夹，存放我们的代码，即:![[git_study1.png]]
因此我们创建了一个demo文件夹，同时我们创建一个`hello.txt`文件来当作我们的代码文件，即利用![[git_study2.png]]
这个时候如果我们使用`ls`，会发现并没有`.git`文件夹，我们做一个`git init`的操作，再`ls`仍旧发现不了`.git`文件夹，是因为`.git`是一个隐藏文件夹，我们应当使用复合指令，在`powershell`中即使用`ls -Hidden`，那么这个时候我们就可以发现`.git`文件了。
![[git_study3.png|975]]
那么这里我们可以利用`git help xxx`，会自动弹出`git`中关于`xxx`函数的的帮助网页。![[git_study4.png]]
![[git_study5.png|825]]
此外，我们可以利用`git status`来查询此刻`git`的状态，即![[git_study6.png|800]]
我们看第二句，实际上就是告诉我们现在还没有提交，第三行是在提醒我们需要利用`git add <filename>`来把文件放入到暂存区，这样才能顺利commit，这里我们插入一下暂存区的概念。

 “暂存区（staging area）”的机制允许我们指定下次快照中要包括那些改动。比如说我们在开发过程中会自动生成很多配置文件，这些并不是代码的核心内容，我们只关心核心代码，因此我们可以选择局部的文件来进行提交。

![[git_study7.png|725]]
这个时候我们已经`git add hello.txt`了，我们这时候再去使用`status`来观察`git`状态，会发现现在状态已经是`to be committed`了，因此我们可以顺利`commit`了。
输入`git commit`之后会进入如此的情景，这实际上是`vim`编辑，我们现在处于键入模式，可以输入内容，利用以下方式退出。![[git_study8.png|900]]![[vim-vi-workmodel.png|550]]
我输入了`add hello.txt`作为`commit`的内容，接下来我们利用`git log`来看`git`的状态，当然我们也可以利用`git log --all --graph --decorate`，这是对其结构的可视化表现，当然由于我们目前只有一个文件(即一个节点，目前看不出来结构)，总之，我们可以发现其一些特征:![[git_study9.png|850]]
第一行即为翻译之后的hash值，后面的`author`与`Date`即为元数据，最后的`add hello.txt`即为备注。
因此我们可以利用`git cat-file -p hash_number`的方式来解读，即：![[git_study10.png|850]]
我们会发现这个`commit`包含很多东西，这实际上与我们前面提到的[[Git_study#2.Git的数据类型]]中的`commit`的数据类型是一致的，其中包含了`tree`，同时这里hash值我们输入钱几位就可以了。
![[git_study11.png|850]]
那么我们继续使用这个`git cat-file -p`，就可以找到`tree`中的`blob`，接着阅读文件内的实际内容。

这里我们再提一下`git log --all --graph --decorate`，这时候我们再创建一个节点，比如:
![[git_study12.png|850]]
这里我们对`hello.txt`增加了一行，并且重新`add`了之后再`commit`，我们再观察`log`与`log --all --graph --decorate`
![[git_study13.png|625]]
会发现，实际上`git log --all --graph --decorate`是展现了数据结构的内容。

我们这里对上图的(HEAD ->master)做一些解释，首先`master`其实是[[Git_study#引用]]，也就是说他是一个`commit`，只不过我们给了他一个新的名字，是主要分支，往往是我们更新的最新的代码，而`head`实际上是一个指针，指向的是我们规定的(或者说我们正在查看的)`commit`，比如说我们可以利用`git checkout hash_number`的方式来改变`head`

`git checkout hash_number`允许我们在历史记录中移动，比如说:![[git_study14.png|700]]
那么我们此时注意到利用`git log`以及观察文件内容会发现变成了原来的样子。![[git_study15.png|700]]
我们会发现此时`Head`指针已经移动到了我们指定的`commit`处。

同时我们可以借助`git diff hash_number1 hash_number2 filename`的方式来比较同一个文件的两个历史内容，如果我什么都不加，直接用`git diff filename`，实际上比较的就是`Head`指针处与现在文件的内容。
```ad-note
title:注意一下git diff中比较的两个内容所处位置
实际上，git中的各种snapshot为一起的，而新的文件自身为新的一起的

```
![[git_study16.png|775]]

另外，`git checkout filename`可以丢弃我对于文件的更改，直接把文件回调至`Head`的状态：
![[git_study17.png|800]]
# 4.Git的基础命令
因此这里我们稍微总结一下`git`的常见命令:
- `git help <command>`: 获取 git 命令的帮助信息
- `git init`: 创建一个新的 git 仓库，其数据会存放在一个名为 `.git` 的目录下
- `git status`: 显示当前的仓库状态
- `git add <filename>`: 添加文件到暂存区
- `git commit`: 创建一个新的提交
    - 如何编写 [良好的提交信息](https://tbaggery.com/2008/04/19/a-note-about-git-commit-messages.html)!
    - 为何要 [编写良好的提交信息](https://chris.beams.io/posts/git-commit/)
- `git log`: 显示历史日志
- `git log --all --graph --decorate`: 可视化历史记录（有向无环图）
- `git diff <filename>`: 显示与暂存区文件的差异
- `git diff <revision> <filename>`: 显示某个文件两个版本之间的差异
- `git checkout <revision>`: 更新 HEAD 和目前的分支

# 5.分支与合并(Branch and Merge)

### 1.Branch的相关用法

我们这里创建一个`animal.py`，并且`git add`之后`git commit`，这里`animal.py`为:
```python
import sys
def default():
    print('Hello')
def main():
    default()
if __name__=='__main__':
    main()
```
`branch`实际上是指分支，而`master`就是我们重要的一个分支，我们可以利用`git branch`指令以及`git branch -vv`指令来说明此刻`git`区域有哪些branch，即:![[git_study18.png|700]]
除此以外，我们还可以利用`git branch branch_name`来创建一个分支名，比如![[git_study19.png|700]]
这个时候我们采用`git checkout cat`，我们转换到`cat`这个分支上，我们补充一个`cat`函数，这是基于最基础函数的一个函数添加，即:
```python
import sys
def cat():
    print('Meow!')
def default():
    print('Hello')
def main():
    if sys.argv[1]=='cat':
       cat()
    else:
       default()
if __name__=='__main__':
    main()
```
那么我们可以指导，执行`python animal.py cat`，可以输出`Meow!`，即![[git_study20.png|475]]
这时候我们继续`git add`与`git commit`之后，利用`git log --all --graph --decorate --oneline`可以知道此刻的`git`状态![[git_study21.png|800]]
**这个时候我们需要把Head转到master引用上**，再创建一个`branch dog`，这样我们相当于基于`master`时候开发了`cat`和`dog`两个函数，即我们考虑`animal.py`为:
```python
import sys
def dog():
    print('Woof!')
def default():
    print('Hello')
def main():
    if sys.argv[1]=='dog':
        dog()
    else:
       default()
if __name__=='__main__':
    main()
```
并且我们在把`Head`转到`master`的基础上，再创造一个`branch dog`，即：![[git_study22.png|700]]
![[git_study23.png|750]]
这里要注意，由于`Head`是在`master`上的，我们进行`dog functionality`的开发是要用`branch dog`上开发的，因此我们需要切换到`git checkout dog`上。

同时我们可以把创建新分支与切换这两个动作用一个指令完成，即`git checkout -b dog`。

这个时候我们再利用`git log --all --graph --decorate --oneline`来查看`git`的状态，会发现此时发生了明显的变化，`branch`我们可以直观地发现了，即![[git_study24.png|775]]
### 2.Merge的用法

我们在[[Git_study#1.Branch的相关用法]]中，已经知道了可以并行地创建`Branch`，从而并行地对一个文件进行多个方向的处理，但是，我在形成了多个分支之后，可能会想形成一个综合的文件，以便使其包含所有的变化，因此我们就需要用到`Merge`。

我们要注意，`git merge`是只需要传入一个`branch`作为参数的，他默认将这个传入的`branch`与我们`Head`指针所指向的分支名称合并，并且所形成的新的内容我们还需要重新`git add`接着`git commit`，默认情况下，合并操作是将指定分支的更改应用到当前分支（HEAD所指向的分支）上，因此如果我们首先需要把这个`Head`先去`git checkout`到我们想要合并的分支上(往往这就是master)，接着我们再去`merge`，**由于我们很难去移动branch的位置，我们一般只能移动Head指针，所以我们需要先想清楚，我们merge时候的那个默认分支，也需要是我们后面提交的branch**

因此我们看这个例子，我们这个时候已经把`cat`和`dog`写完了，这是基于源文件也就是`master`的两个分支，这个时候如果`Head`位于`master`处，如果我们利用`git merge cat`我们知道`cat`就是基于`master`做的更改，此时的`merge`就会是`fast forward`，直接把`Head->Master`移动到`cat`处即可。
![[git_study25.png|800]]
那么现在我们想再把`dog`也合并进来，但是他们会出现冲突，我们观察会发现在`vscode`中代码情形是这样的：
![[Pasted image 20240416100020.png|800]]
我们会发现这里出现了冲突，因为`cat`和`dog`函数中都是判断`sys.argv[1]`的情况，因此产生了冲突，那么我们可以选择是保留原始的更改还是采取原始更改还是保留双方更改，一般都是两者都保留，并且我们需要人为地修改其中的冲突，`git`中说明冲突的符号是：
```git
<<<<<<< HEAD
// 你的更改
== == == ==
// 其他人的更改
>>>>>>> branch-name
```

需要手动编辑这些文件，解决冲突部分，并删除Git插入的冲突标记。解决冲突后，**需要使用`git add`命令将这些文件标记为已解决**，然后执行`git commit`来完成合并提交。![[git_study27.png|250]]
另外提一些在合并中出现问题的处理方法，
* 如果在合并过程中您需要中断操作（例如，去解决复杂的冲突），您可以使用git merge --abort来停止合并并回到合并前的状态。
* `git reset --soft HEAD~1`是把HEAD指针移动到上一次提交，同时保留工作目录和暂存区中的更改。

![[git_study28.png|800]]
我们在人工修改完成之后，使用`git merge --continue`来完成合并(此时只需要用add不需要用commit)。

![[git_study29.png|700]]

# 6.Git与远程仓库(包括Github)的交互

我们这里拿`github`来举例子来说明如何把本地文件上传到远端仓库中。

首先我们应该在`github`上创建属于我们的`repository(repo)`，即点击`new`，接着填写有关的内容，这里需要注意，如果我们选择了需要`readme.md`之后，远端仓库和本地仓库的历史与内容就是不一致的了，因此我们需要采取某些方式来解决：![[git_study30.png|1025]]
我们这里解释一下`readme.md`以及一系列的问题：
* github上默认branch是main，windows的git默认的branch是master，因此我在本地上修改以及`commit`之后，实际上是在master上进行的，接着我们push到远程仓库上，会push到master branch上，而不是github dafault的main branch
* 因此我们需要利用命令把windows中的branch从master改为了main再去push到remote，由于我在github上创建了readme.md文档，这个文档会出现在default的main branch中，但是这个readme.md是在远程仓库中的操作，在本地并没有，导致本地的main和remote的main不一样，因此如果我想正常地pull和push都会报错，我只能去强行合并这两个历史记录。
解决第一个问题，是利用`git pull origin main --allow-unrelated-histories`,即把`github`中的`origin`的内容都拉取到本地的`main`分支，并且我们后面的那个命令是允许不相干的历史相结合，从而把`readme.md`拉到本地了。

我们具体来说明一下步骤：
## 1.方法一(先初始化本地仓库再创建远端仓库)

利用`git init`在本地的文件存储的文件夹初始化`git`，我们利用`git remote add (remote_name) <url>`,一般默认远端仓库名称是`origin`,接着`git add filename`以及`git commit`之后，我们这个时候注意一下`git log --all --graph --decorate --oneline`，可以发现这时候`git`的情况是:
![[git_study31.png|800]]
我们需要注意绿色的`main`是本地的`branch`，而`origin/main`是远端仓库的`branch`，我们会发现这时候如果`git pull`会显示`fatal: refusing to merge unrelated histories`,因此我们需要使用`git pull origin main --allow-unrelated-histories`,强行合并之后我们再看`git`的情况![[git_study32.png|850]]
这个时候我们就可以利用`git push`来拉取内容，利用`git pull`来提交申请。

## 2.方法二(先创建远端仓库再clone一个本地仓库)

这个方法不需要使用`git init`以及`git add`，区别于[[Git_study#1.方法一(本地已经有文件commit了之后)]]，我们首先在`github`的`respositories`创建一个新的内容，输入相应的命令，接着在自己想要的文件夹中,使用`git clone`来把远端的仓库整个`clone`过来，语法为`git clone https://github.com/ShengJoezz/Sparse-Matrix-Compression-Method-Based-on-Cross-Linked-Lists-in-Fortran`，**注意是不需要初始化，也不需要add和commit的**
![[git_study33.png|925]]
`git clone`之后形成如此情况，我们注意到会自动形成本地指针`Head`指向本地的`branch`，接着我们可以顺利地`git pull`我们的内容了。
![[git_study34.png|950]]
语法为`git push (remote_name)(local_branch)`

但是我们仍要注意，在使用`git pull`之前，我们需要把相应的内容`git add/commit`，因为`git pull`实质是在对`git`的工作目录进行操作，而非我们的工作目录。

### 远端操作
- `git remote`: 列出远端
- `git remote add <name> <url>`: 添加一个远端
- `git push <remote> <local branch>:<remote branch>`: 将对象传送至远端并更新远端引用
- `git branch --set-upstream-to=<remote>/<remote branch>`: 创建本地和远端分支的关联关系
- `git fetch`: 从远端获取对象/索引
- `git pull`: 相当于 `git fetch; git merge`
- `git clone`: 从远端下载仓库

# 7.git的高级操作

## git config

全称应该是`git configuration`,其中configuration应该是配置的意思，因此利用`git config`可以用来显示配置，更改配置，添加配置，也可以把某个命令重命名为一个新的名字。

因此我们说明这几种功能的命令。
1. **查看配置**

```
git config --list [--show-origin]
```
这个命令会列出所有当前生效的配置及其来源(如global/local/system等)。加上`--show-origin`可以显示每个配置的确切来源。

2. **查看特定配置**

```
git config [--global|--local|--system] <key>
```
使用不同的作用域标志可查看该级别的某个特定配置值。如`git config --global user.name`。

3. **添加配置**

```
git config [--global|--local|--system] --add <key> <value>
```
使用`--add`选项可以为某个key添加一个新值,如果已存在该键,则添加一个新值,而不是覆盖。

4. **修改配置**

```
git config [--global|--local|--system] <key> <value> 
```
直接为某个配置键赋予新值即可修改配置。常用于修改用户名/邮箱等。

5. **移除配置**

```
git config [--global|--local|--system] --unset <key>
```
使用`--unset`选项移除某个配置项。

6. **配置别名**

```
git config --global alias.<alias-name> '<git-command>'
```
可以为常用的Git命令设置一个简短的别名,提高输入效率。如`git config --global alias.st 'status'`。

常用的配置包括:

- `user.name` 和 `user.email` - 设置提交时的用户信息
- `core.editor` - 设置Git使用的默认文本编辑器
- `commit.template` - 设置提交消息的模板文件路径  
- `push.default` - 设置`git push`不加参数时的行为
- `pull.rebase` - 是否在pull时使用rebase模式
- `alias.*` - 命令别名

## git stash

`stash`是暂存的意思，`git stash`命令是Git中用于临时存储未完成更改的实用工具。当你需要切换分支或者暂时搁置当前的工作时,可以使用stash来保存目前的工作进度,后续可以在恢复。以下是一些常见的`git stash`用法:

1. **暂存当前工作目录的状态**

```
git stash
```

这将当前工作目录中所有未跟踪的文件和已修改的文件存储到一个stash堆栈中,让你的工作目录回到之前的干净状态。(回到提交的最后一次状态）

2. **查看stash列表**

```
git stash list
```

显示所有存储在stash中的记录,以便后续选择恢复哪一个。

3. **恢复之前stash的内容**

```
git stash pop 
```

从stash堆栈中取出最近的一次stash,并将其内容应用到当前工作目录。同时该stash被永久删除了。

```
git stash apply stash@{0}
```

通过指定`stash@{n}`来应用特定的stash记录,但不会从stash列表中删除它。

4. **删除stash记录**

```
git stash drop stash@{0}
```

删除指定的stash记录。如果不指定,默认删除最近的stash。

5. **创建stash时添加消息**

```
git stash save "Stashing while fixing bug #123"
```

可以给新创建的stash添加一个消息便于标识。

6. **创建stash时只stash某些文件**

```
git stash push -- path/to/files
```

默认会暂存整个工作目录,加文件路径参数可以只stash指定文件。

7. **重新应用stash时创建新分支**

```
git stash branch new-branch-name stash@{0}
```

这会基于stash记录创建一个新分支,并自动将stash应用到新分支,非常方便。

stash功能使得你可以快速保存当前的工作现场,而无需提交代码。合理利用stash可以最大程度地避免由未完成工作引起的代码混乱。不过请注意,stash只能暂时保存工作目录中的更改,不包括未track的新文件。使用完后记得清理stash堆栈。

## git bisect

在Git版本控制系统中，`bisect` 是一个命令，用于通过二分查找的方式来定位引入错误或回归的特定提交。这个过程被称为“二分搜索”，因为它通过不断将搜索范围分成两半来逐步缩小引入问题的范围。

当你的项目在某个版本之后出现了错误，而你想要找出是哪一个具体的提交导致了这个问题，`bisect` 命令就会非常有用。以下是一些关于 `bispt` 的关键点：

1. **二分查找**：`git bisect` 使用二分查找算法来快速定位引起问题的提交。

2. **标记已知**：你需要告诉Git一个已知的错误提交（bad commit）和至少一个没有错误的提交（good commit）。

3. **逐步缩小**：Git会从好的提交开始，自动检查中间的提交，询问用户这个提交是好的还是坏的。

4. **自动化**：用户可以自动地通过脚本进行二分查找，也可以手动检查每个提交。

5. **结束搜索**：一旦找到引入问题的提交，你可以使用 `git bisect reset` 来退出二分查找状态，回到原来的位置。

6. **报告问题**：找到问题提交后，可以报告bug或进一步调查问题的原因。

使用 `git bisect` 的基本流程如下：

```shell
git bisect start            # 开始二分查找
git bisect bad <commit>     # 标记已知的错误提交
git bisect good <commit>   # 标记一个或多个已知的好的提交
git bisect run <script>     # 自动运行二分查找，<script> 是测试当前提交是否有问题的脚本
# 或者
git bisect checkout          # 手动检查当前提交是否有问题，然后标记为 good 或 bad
git bisect reset            # 结束二分查找并重置HEAD到开始二分查找前的状态
```

## `.gitignore`文件

其实就是`git ignore`，也就是说我们有一些文件是自动生成的，是不会用于提交的，因此我们使用这个文件说明哪些文件不需要提交。

`.gitignore`文件是Git中一个非常有用的功能,它允许我们在提交代码时排除掉某些不需要被Git跟踪的文件或目录。正确使用`.gitignore`文件可以确保我们的代码库中只包含必要的源代码和资源文件,避免一些系统文件、缓存文件或者构建产物等无关文件被不小心提交。

以下是`.gitignore`文件的一些常见用法:

1. **忽略指定文件**

在`.gitignore`中直接写入要忽略的文件名,例如:

```
debug.log
temp.txt
*.tmp
```

2. **忽略指定目录**

如果要忽略整个目录,需要在目录名后加上`/`斜杠,例如:

```
build/
dist/
node_modules/
```

3. **使用通配符**

`.gitignore`支持glob模式的通配符,例如:

```
*.log    # 忽略所有.log文件
logs/*   # 忽略logs目录下所有文件
!main.log  # ! 号可以对指定模式进行例外处理
```

4. **忽略不同操作系统的文件**

可以在`.gitignore`中添加不同操作系统下的系统文件,例如:

```
# Windows
Thumbs.db
Desktop.ini

# macOS
.DS_Store

# Linux
*~
```

5. **注释**

`#`号开头的行被视为注释行,便于描述和解释忽略的原因。

6. **全局gitignore**  

可以为当前用户配置一个全局的`.gitignore`文件,在任何本地仓库中都会生效。

```
git config --global core.excludesfile ~/.gitignore_global
```

7. **参考gitignore模板**  

各种流行项目或编程语言都有自己的`.gitignore`模板文件,可以去网上参考并复制过来作为基础。比如Node项目可以用[https://github.com/github/gitignore/blob/main/Node.gitignore](https://github.com/github/gitignore/blob/main/Node.gitignore)

合理地使用`.gitignore`可以有效防止一些敏感数据或系统文件被不小心提交到代码库中,使得仓库只保留必要的文件,有利于减小仓库体积、避免冲突等。同时也要注意不可过度使用`.gitignore`来"藏匿"一些关键文件,防止引起版本控制的混乱。


# 8.Github的workflow

## 1.先初始化创建新的仓库

在`github`中创建一个`respository`，可能会引入`readme.md`文档，接着利用先`cd`到自己想要的文件夹，再`git clone`，这会创建一个子文件夹，并把所有的内容克隆过来。
```shell
git clone https://github.com/your-username/your-repo.git
```
注意，利用`clone`命令之后不需要再利用`git remote add`了，他会自动添加。

## 2.正常写文件提交

即正常创建文件并修改，即以下步骤:

```shell
git add .         # 添加当前目录下所有修改
git add filename  # 添加指定文件
```

接着利用:
```shell
git commit -m "commit message" # 提交与输入提交信息
```

这样我们可以利用
```shell
git push origin main
```
这样我们可以创建了一个主要的分支，这个`main`分支作为我们的稳定版本的提交分支。

## 3.开发流程

提醒一下，在开发之前的习惯是同步一下主线的代码，即：

```shell
git checkout main
git pull
```

### 1.创建新的branch.

我们开发流程的时候，我们需要用新的`branch`来处理新的`feature`，即

```shell
git checkout main     # 切换到main分支 
git pull              # 确保获取main上最新代码
git checkout -b new-feature  # 创建并切换到新分支
```

### 2.在新的branch上修改文件并提交

```shell
# 在new-feature分支上正常修改文件
git add .
git commit -m "Implement new feature XYZ"
```

### 3. 推送新分支到远程仓库

```shell
git push origin new-feature # 注意这里应该是new-feature而不是main
```

## 4.团队中的审查

在GitHub上可以看到新推送的`new-feature`分支,点击发起Pull Request将其合并到main分支。等待团队审查你的代码修改,如果通过则可以在GitHub上点击确认合并Pull Request到main分支。

这里做一些关于`pull request`和`push origin main`的区别说明：

发起Pull Request(简称PR)确实不是用来直接将代码提交到远程仓库的,而是一个代码审查和合并的过程。
在GitHub的工作流程中,发起Pull Request通常包含以下几个步骤:
1. 从主干分支(如main)创建一个新的开发分支,并在该分支上进行功能开发和代码修改。
2. 使用`git push`命令将你的开发分支推送到远程GitHub仓库。
3. 在GitHub网站上,你会看到你新推送的分支,这时可以点击"Compare & pull request"发起一个新的Pull Request。
4. 填写PR的标题和描述,说明这个PR包含了什么功能或修复。
5. 通常会让项目的维护者或其他开发人员review这个PR中的代码修改。他们可以讨论、提出修改建议。
6. 进行必要的回复修改和新的推送后,等待维护者批准并将你的PR合并到主干分支中。

所以,Pull Request本身并不是用来推送代码的,而是一个"请求"去合并你的分支代码到主线代码库的过程。它为团队审查、讨论代码提供了一个平台,保证了代码质量。
使用`git push`命令只是将你分支的最新提交推送到远程,让这个分支与远程保持同步,这是PR被合并所需的前提条件。
PR被批准合并后,你的功能分支代码才真正被并入了主线代码库。所以发起PR和推送代码是两个不同但又相互关联的步骤。发起PR是GitHub工作流中的一个关键环节,通过它保证了代码审查,提高了代码质量和开发效率。

## 5.删除已合并的本地和远程分支(可选)

```shell
git checkout main
git pull   # 更新main分支
git branch -d new-feature  # 删除本地new-feature分支  
git push origin --delete new-feature # 删除远程new-feature分支
```

