## 基本概念

1. 版本库 .git 文件：
   - 当使用 git 管理文件时，会有一个 .git 文件，称作版本库；
   - .git 文件的作用就是在它创建的时候，会自动创建 master 文件，并且将 HEAD 指针指向 master 分支；
2. 工作区：
   - 本地项目存放文件的位置；
3. 暂存区：
   - 暂时存放文件的地方，通过 add 命令将工作区的文件添加到缓冲区；
4. 本地仓库：
   - 通常情况下，使用 commit 命令可以将暂存区的文件添加到本地仓库；
   - 通常而言，HEAD 指针指向的是 master 分支；
5. 远程仓库：
   - 当我们使用 gitHub 托管我们的项目时，它就是一个远程仓库；

## 文件状态

当我们需要查看一个文件的状态：

```
git status
```

1. changes not staged for commit：表示工作区有该内容，但是缓存区没有，需要 git add；
2. changes to be commited：表示文件在缓存区，需要 git commit ；
3. nothing ti commit，work tree clean：这个时候，将本地的代码推送到远端即可；

## 常见命令

### 配置命令

- 列出当前配置

```
git config --list	
```

- 列出Repository配置

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

### fetch指令

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

- 比较缓存区与本地库最近一次commit内容

```
git diff -- cached
```

- 比较工作区与本地最近一次commit内容

```
git diff HEAD
```

- 比较两个commit之间差异

```
git diff
```

## 分支命名

