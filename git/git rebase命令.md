git rebase是一个非常有用的命令，但是可能熟悉它的人比较少。下面介绍一下git rebase的几种常见用法。



### git rebase branch

我们在作分支合并的时候最常用的就是merge操作，但是执行merge之后，会产生一个新的commit，例如：Merge branch 'branch-1'。这个commit把两个branch合并到一起并作了一次新的提交。但是，如果使用rebase的话就会避免这个问题。

```

* 5eaa8f8 (HEAD -> master) commit 8

* fdadab0 commit 7

* 690e761 (branch-2) commit 10

* f8bcb41 commit 9

*   a1e3e91 Merge branch 'branch-1'

|\  

| * da2448e (branch-1) commit 6

| * c4ef94a commit 5

* | c70cc70 commit 4

* | 31fde3f commit 3

|/  

* faf3890 commit 2

* 0f1f7a8 commit 1

```

上述git history描述如下：首先，在master创建进行了两次提交（commit 1， commit 2），checkout新的分支branch 1，在master上进行两次提交（commit 3， commit 4），在branch 1上进行两次提交（commit 5， commit 6），将分支checkout到master，执行git merge branch-1，然后产生了`a1e3e91 Merge branch 'branch-1'`这个提交。



接着，checkout 新分支branch 2，在master上进行两次提交（commit 7，commit 8），checkout到branch 2上，进行两次提交（commit 9, commit 10），checkout到master执行git rebase branch-2，这是可以看到，在git history上并没有产生多余的提交。



git rebase branch-2命令会把你的"master"（即当前分支）分支里的每个提交(commit)取消掉，并且把它们临时保存为补丁(patch)(这些补丁放到".git/rebase"目录中),然后把"master"分支更新到最新的"branch-2"分支，最后把保存的这些补丁应用到"master"分支上。这些新应用的commit会产生新的commit。上例中commit 7，commit 8已经是新的commit号（原有的commit号分别是：cc4df2d，c86a6da）。



当我们从远程拉代码的时候如果使用：git pull --rebase，则会自动执行rebase。



### git rebase --interactive

git rebase用来修复commit，该命令可以简写为git rebase -i。执行该命令之前需要当前branch已经设定过upstream，使用`git branch --set-upstream-to=<remote>/<branch>`可以设定upstream。执行rebase -i命令后的交互如下：

```

GNU nano 2.5.3                    File: /home/focusj/3stone/diamond/.git/rebase-merge/git-rebase-todo                                                



pick c4a4b5d test

squash c4a4b6d test

drop c4a4b7d test



# Rebase 0df8fd7..c4a4b5d onto 0df8fd7 (1 command(s))

#

# Commands:

# p, pick = use commit

# r, reword = use commit, but edit the commit message

# e, edit = use commit, but stop for amending

# s, squash = use commit, but meld into previous commit

# f, fixup = like "squash", but discard this commit's log message

# x, exec = run command (the rest of the line) using shell

# d, drop = remove commit

#

                                                                 [ Read 20 lines ]

^G Get Help     ^O Write Out    ^W Where Is     ^K Cut Text     ^J Justify      ^C Cur Pos      ^Y Prev Page    M-\ First Line  M-W WhereIs Next

^X Exit         ^R Read File    ^\ Replace      ^U Uncut Text   ^T To Spell     ^_ Go To Line   ^V Next Page    M-/ Last Line   M-] To Bracket

```

下边说下几个command的用处：

p：保留当前commit，不做处理。

r：修改commit message。

e：对这个commit作修改。比如某个commit漏掉了什么配置，想要再提交新的文件，可以使用这个command。

s：保留这个commit的修改，但是把它合并到前一个commit中。

d：删除commit。

使用命令时，只需要把命令放到mouge commit前边保存退出既能生效。



write on 2017-1-3