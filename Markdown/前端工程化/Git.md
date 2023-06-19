https://blog.csdn.net/u012244479/article/details/130049929

## 基本概念

- **版本库 .git 文件**：.git 文件的作用就是在它创建的时候，会自动创建 master 文件，并且将 HEAD 指针指向 master 分支；
- Git 的四个工作区域：

  - **工作区：**就是我们看到的本地项目文，日常我们也是在这进行开发；
  - **暂存区：**介于工作区和本地仓库之间，暂时存放文件的地方，主要是标记修改的内容，（通过 add 命令将工作区的文件添加到缓冲区）；
  - **本地仓库（.git）：**一般我们的项目根目录下有个隐藏目录 `.git` ，使用 commit 命令可以将暂存区的文件添加到本地仓库；
  - **远程仓库：**当我们使用 gitHub 托管我们的项目时，它就是一个远程仓库；

- **HEAD：** 有一个 HEAD 指针指向当前分支（只有一个分支的情况下会指向 masetr，master 会指向最新提交）

## 文件状态

当我们需要查看一个文件的状态：

```
git status
```

1. changes not staged for commit：表示工作区有该内容，但是缓存区没有，需要 git add；
2. changes to be commited：表示文件在缓存区，需要 git commit ；
3. nothing ti commit，work tree clean：这个时候，将本地的代码推送到远端即可；

## Git 处理分支的方式

1. commit 操作会为每个目录下的每个文件计算 hash 校验，然后在 Git 仓库里将把这些保存为树对象
2. Git 会创建一个提交对象（commit Object），这个对象会包含一个指向暂存内容快照的指针，还包含了提交的相关息以及指向它的父对象的指针
3. 然后把当前版本的文件快照保存到 Git 仓库中（使用 blob 对象保存），最终将校验加入到暂存区等待提交
4. 首次产生的提交对象没有父对象，普通提交操作产生的提交对象有一个父对象，由多个分支合并产生的提交对象有多个父对象

## Git 仓库中的五个对象

- 三个 blob 对象（保存着文件快照）
- 一个树对象（记录着目录结构和 blob 对象索引）
- 一个提交对象（包含着指向前述树对象的指针和所有提交信息）

**注意：**Git 保存的不是文件级的变化或者差异，而是一系列不同时刻的文件快照

## 常见命令

### 代码回滚场景

#### 在工作区的修改

当文件只在工作区有修改，还没有进行 add 和 commit 操作时，可以使用以下命令来回滚工作区的修改

这种情况被丢弃的内容是不可操作的

```
git checkout -- [filePath]
```

#### 已添加到暂存区的修改

如果文件已通过 `git add` 将改动添加到了暂存区，但是还没有 `commit` ，可以使用以下命令来回滚修改

```
git reset HEAD [filePath]
```

执行该命令回滚后会保留该文件的改动，可以重新编辑进行再次 `add`

#### 已添加到本地仓库的修改

如果文件已经通过 `git commit` 将文件添加到了本地仓库，但是还没有 push，可以使用以下命令来回滚修改

```javascript
git reset <需要回滚的commit-id>  // 回滚到某次 commit，但是改动的文件会保留在工作区
git reset -hard <需要回滚的commit-id> // 文件不会保留在工作区，会直接丢弃
```

这种情况下的回滚，该 commit 之后的提交记录会被丢弃 ???

#### 已添加到远程仓库的修改

如果代码已经提交到远程仓库，此时需要使用以下命令 `git revert` 进行回滚，

#### reset 和 revert 的差异

| 命令       | 是否抹掉历史                                                 | 适用场景                                                     | 颗粒度             |     |
| ---------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------ | --- |
| git reset  | **是**，回滚的历史将消失                                     | 本地提交的改动，但是还未 push 到远端的提交（不介意抹去历史） | 可以作用于特定文件 |     |
| git revert | **否**，历史提交记录不受影响，回滚后还会生成一次新的提交记录 | 已经提交到远端的改动（需要留痕）                             | 不可作用城文件层面 |     |

### git reset 回滚某次提交、

使用 git reset 是不会产生 commits 的，它只是更新一个 branch 指向另一个 commit，说白了它就是将一个分支的 HEAD 指向了之前某一次的提交，一般用来移除当前分支的一些提交

使用 git reset （代码回滚）进行强制回滚，约等于重置 HEAD （当前分支的版本顶端）到另一个 commit 上，所有在该 commit 之后的提交记录会被丢弃！

```
git reset --hard HEAD~2 (数字代表要回退几个版本)
```

**--hard 和 --soft 和 mixed（默认）的区别 **

- --hard：就是删除提交记录并不保存所删除记录所做的更改（暂存区和工作区都同步到你指定的提交）
- --soft：虽然重置了最近两次的提交，但是还是保存了提交所做的更改，所有的在 original HEAD 到你重置的那个 commit 之前所有的变更都放在暂存区中（缓存区和工作区都不会被改变)
- --mixed：默认参数，所以在该分支上从 original HEAD 到重置的那个 commit 之间的所有变更都作为 local modifications 保存在工作区中（未 staged），我们可以重新修改再做 commit（暂存区和指定的提交同步，工作区不受影响）

### 复制 A 分支上某些提交记录到 B 分支上（git rebase）

如果我们想把在 A 分支上提交过的某些记录也复制到 B 分支上，使用命令如下：

```
 git rebase [startpoint] [endpoint]  --onto  [branchName]
```

**说明：**

- `[startpoint]  [endpoint]` ` 指定一个区间（和 git rebase 使用方法相同）

- `--onto` 是指将指定的区间（**前**）的提交要复制到哪个分支（**后**）上

- 复制提交后，当前 HEAD 是 处于游离状态，此时我们需要将 B 分支所指向的提交 id 设置为当前 HEAD 所指向的提交 id 就可以了

  ```
  git checkout brance-b
  git reset --hard [commitID]
  ```

- 、

### 找回合流后删除的分支

一般我们会在开发功能的时候在新的分支上开发，上线后再将我们的分支合并到主分支上，再删除对应的开发分支

如果我们需要重建已经合并的分支，可以通过提交记录找到对应的分支节点，基于该 commit 再新建分支

```
git branch {目标分支} {commitId}
```

### 找回本地误删除的分支

当我们使用 `git branch -D xx ` 删除本地分支后，发现删错了，

这个时候可以通过 reflog 找到分支被删前的 commit，基于目标 commit 重建分支

```
git branch <分支名> <目标commit>
```

### 恢复特定 commit 中的某个文件

```
git checkout <目标 commit> -- <文件>
```

### 修改提交信息

#### 修改最近一次的提交

**方式一**：

- 输入命令： `git commit --amend` ，会进入对最后一次提交信息编辑的 vim 编辑器界面
- 修改为正确的提交信息后，按 `ESC` 退出到普通模式
- 普通模式下按 : 进入命令模式
- 输入 wq 即可保存修改并退出 vim 编辑器

**方式二：**

通过命令行直接修改：

```git
git commit --amend --message="new commit message"
git commit --amend --author="xxx@xx.com"
```

【如果修改是已经提交到远程仓库的 commit 信息】上面两种方式都必须要用 `git push --force` 将修改提交到远程仓库，不然会报错

#### 修改最近两个或者两次以上的 commit 信息

如果要修改之前某一次或多次提交的 commit 信息，就要用到 git rebase 的功能了：

- 输入命令：`git rebase -i HEAD~｛N｝` ，其中 N 表示最近 N 次的提交

### rebase 命令

rebase 一般使用场景如下：

- 修改已提交的 `commit message` ：修改错字，错误说明等
- 分支提交混乱，需要梳理：多人开发分支提交先后产生冲突，`merge`之后 git 记录呈现树状
- 合并多次修改提交：上线打包前，针对某一功能的需求进行提交说明合并（如果是公共仓库或者功能尽量少操作）

使用命令：

```javascript
git rebase -i  [startpoint]  [endpoint]
```

**说明：**

- `-i` 的意思是 `--interactive` ，即弹出交互式的界面让用户编辑完成合并操作；
- ` [startpoint]  [endpoint]` 指定的是一个编辑区间（这个区间是一个**前开后闭**的区间），如果不指定 endpoint，该区间的终点默认是当前分支 HEAD 所指向的 commit

命令列表翻译一下\*\*：

- pick：保留该 commit，缩写 :p
- reword：保留该 commit，但我需要修改该 commit 的注释，缩写 :r
- edit：保留该 commit，但我要停下来修改该提交（不仅仅是修改注释）缩写 :e
- squash: 将该 commit 和前一个 commit 合并，缩写 :s
- fixup：将该 commit 和前一个 commit 合并，但我不要保留该提交的注释信息，缩写 :f
- exec：执行 shell 命令，缩写 :x
- drop：我要丢弃该 commit，缩写 :d

编辑跟 linux 的 vim 相似，先输入 i 或者 insert 键，会出现一个插入的标识，然后就可以编辑蓝色区域的命令了

编辑好后就输入 :wq 回车保存，再 push 就可以啦

### 找回已删除的内容

`git reflog` 可以找回被 reset 丢掉的 commit 、被删掉的分支等，

但下面这些情况的“删除”，这个命令也是“无能为力”：

- **非本地操作的记录：**`git reflog ` 能管理的是本地工作区的操作记录，如果是其他他或者其他机器上操作的记录它就没辙了
- **未 commit 的内容：**如果是在 工作区或者暂存区就被回滚的内容（未被 commit），这种类型也是不能处理
- **“超时”的内容：**`git reflog ` 保留的记录有时间限制（默认 90 天），超时的会被自动清理，或者开发者自己主动执行清理命令也会提前清理掉
- 输入命令：`git rebase -i HEAD~｛N｝` ，其中 N 为

### 配置命令

- 列出当前配置

```
git config --list
```

- 列出 Repository 配置

```
git config --local --list
```

- 列出全局配置

```
git config --global --list
```

- 列出系统配置

```
git config --system --list
```

- 配置用户名

```
git config --global user.name "your name"
```

- 配置用户邮箱

```
git config --global user.email "youremail@github.com"
```

### 分支管理

- 查看本地分支

```
git branch
```

- 查看远程分支

```
git branch -r
```

- 查看本地和远程分支

```
git branch -a
```

- 从当前分支，切换到其他分支

```
git checkout <branch-name>
```

- 创建并切换到新建分支

```
git checkout -b <branch-name>
```

- 删除分支

```
git branch -d <branch-name>
```

- 当前分支与指定分支合并

```
git merage <branch-name>
```

- 查看哪些分支已经合并到当前分支

```
git branch --merged
```

- 查看哪些分支没有合并到当前分支

```
git branch --no-merged
```

- 查看各个分支最后一个提交对象的信息

```
git branch -v
```

- 删除远程分支

```

```

- 重命名分支

```
git branch -m <oldbranch-name> <newbranch-name>
```

- 拉取远程分支并创建本地分支

```
git checkout -b 本地分支名 origin/远程分支名

git fetch origin <branch-name>:<local-branch-name>
```

### fetch 指令

> 将远程仓库内容更新到本地；

```
git fetch origin <branch-name>:<local-branch-name>
```

1. 一般而言，这个 origin 是远程主机名，一般默认是 origin；
2. branch-name 是要拉取的分支；
3. local-branch-name 就是本地新建一个新分支，将 origin 下的某个分支代码下载到本地分支；

- 将某个远程主机的更新，全部取回本地

```
git fetch <远程主机名>
```

- 取回特定分支，可以指定分支名

```
git fetch <远程主机名> <分支名>
```

- 将某个分支的内容取回到本地下某个分支

```
git fetch origin :<local-branch-name>
```

### 撤销

- 撤销工作区修改

```
git checkout --
```

- 暂存区文件撤销（不覆盖工作区）

```
git reset HEAD
```

- 版本回退

```
git reset --(soft | mixed | hard ) < HEAD ~(num) > |
```

1. --hard：回退全部，包括 HEAD、index、working tree；
2. --mixed：回退部分，包括 HEAD、index；
3. --soft：回退 HEAD；

### 状态查询

- 查看状态

```
git status
```

- 查看历史操作记录

```
git reflog
```

- 查看日志

```
git log
```

### 文档查询

- 展示 Git 命令大纲

```
git help (--help)
```

- 展示 Git 命令大纲全部列表

```
git help -a
```

- 展示具体命令说明手册

```
git help
```

### 文件暂存

- 添加改动到 stash

```
git stash save -a "message"
```

- 恢复改动

```
git stash pop <stash@{ID}>
```

- 删除暂存

```
git stash drop <stash@{ID}>
```

- 查看 stash 列表

```
git stash list
```

- 删除全部缓存

```
git stash clear
```

### 差异比较

- 比较工作区与缓存区

```
git diff

```

- 比较缓存区与本地库最近一次 commit 内容

```
git diff -- cached
```

- 比较工作区与本地最近一次 commit 内容

```
git diff HEAD
```

- 比较两个 commit 之间差异

```
git diff
```

## 分支命名

- master 分支
  - 主分支，用于部署生产环境的分支，确保稳定性；
  - master 分支一般由 develop 以及 hotfix 分支合并，任何情况下都不能直接修改代码；
- develop 分支
  - develop 为开发分支，通常情况下，保存最新完成以及 bug 修复后的代码；
  - 开发新功能时，feature 分支都是基于 develop 分支下创建的；
- feature 分支
  - 开发新功能，基本上以 develop 为基础创建 feature 分支；
  - 分支命名：feature/ 开头的为特性分支， 命名规则: feature/user_module、 feature/cart_module；
- release 分支
  - release 为预上线分支，发布提测阶段，会 release 分支代码为基准提测；
- hotfix 分支
  - 分支命名：hotfix/ 开头的为修复分支，它的命名规则与 feature 分支类似；
  - 线上出现紧急问题时，需要及时修复，以 master 分支为基线，创建 hotfix 分支，修复完成后，需要合并到 master 分支和 develop 分支；

## 忽略文件 .gitignore

> 这个文件会去忽略一些不需要纳入 Git 管理、也不希望出现在未跟踪文件列表；

```javascript
# 此行为注释 会被Git忽略

# 忽略 node_modules/ 目录下所有的文件
node_modules


# 忽略所有.vscode结尾的文件
.vscode

# 忽略所有.md结尾的文件
*.md

# 但README.md 除外
!README.md

# 会忽略 doc/something.txt 但不会忽略doc/images/arch.txt
doc/*.txt

# 忽略 doc/ 目录下所有扩展名为txt文件

doc/**/*.txt

```
