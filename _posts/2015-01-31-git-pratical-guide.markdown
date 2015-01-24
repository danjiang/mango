---
title: Git 实用指南 
author: 但江
location: 成都 
category: programming
---

Git 是一款免费的，开源的分布式版本管理系统，目的在于快速而有效地管理从小项目到庞大的项目。 每一个 Git 克隆都是一个完整的版本库，有完整的历史信息和充分的修订跟踪能力，不需要依赖于网络访问或者一个中心服务器，分支和合并可以快速简单地完成

#### 初次运行 Git 前的配置

	$ git config --global user.name "John Doe"
	$ git config --global user.email johndoe@example.com

#### Git 仓库管理

要对现有的某个项目开始用 Git 管理，只需到此项目所在的目录，执行

	$ git init

从现有仓库克隆

	$ git clone git://github.com/schacon/grit.git mygrit

#### Git文件操作

文件状态流转

![Git File Status](/images/git-file-status.png)

检查当前文件状态

	$ git status

跟踪新文件

	$ git add README

暂存已修改文件

	$ git add benchmarks.rb

提交更新

	$ git commit
	$ git commit -m "Story 182: Fix benchmarks for speed"

移除文件

	$ git rm grit.gemspec

移除跟踪但不删除文件

	$ git rm --cached readme.txt

移动文件

	$ git mv file_from file_to

查看提交历史

	$ git log

在Git管理的项目下创建一个名为 .gitignore 的文件，列出要忽略的文件模式

	# 此为注释 – 将被 Git 忽略
	*.a       # 忽略所有 .a 结尾的文件
	!lib.a    # 但 lib.a 除外
	/TODO     # 仅仅忽略项目根目录下的 TODO 文件，不包括 subdir/TODO
	build/    # 忽略 build/ 目录下的所有文件
	doc/*.txt # 会忽略 doc/notes.txt 但不包括 doc/server/arch.txt

#### 远程操作

查看当前的远程库

	$ git remote

从远程仓库抓取数据

	$ git fetch [remote-name]

推送数据到远程仓库

	$ git push [remote-name] [branch-name]

查看远程仓库信息

	$ git remote show [remote-name]

远程仓库的删除和重命名

	$ git remote rename pb paul
	$ git remote rm paul

#### 分支操作

当前分支和某一分支合并，Choose to merge when you have a feature on a separate branch and want to bring that code into master or another branch

	$ git merge origin/master

当前分支和某一分支衍合，Choose to rebase when you want to stay in sync with the main branch when you’re working on a long-lived side branch

	$ git rebase master

这个命令再给你一次机会添加忘记了提交的文件，或者修复最近一次提交中误输入的信息，会将这次提交的内容和最近一次提交的内容合并，历史记录中只会看到合并的提交信息

	$ git commit –amend

#### 回滚操作

产生一个新的提交并回滚以前的某一个提交

	$ git revert dd61ab32 

回滚到的某一个提交，并更新本地目录

	$ git reset --hard 69988cc

强制更新remote

	$ git push origin -f

#### 工作流

**commit** 提交

提交信息尽量写做了什么改动， fix #1 会自动将 ID 为 1 的 issue 变为 resolved

**master** 主干

代表最新发布的版本，只能从其他分支来合并更新

**development** 开发分支

主要的开发分支，常规的改动提交，或者从其他分支合并更新

**feature/** 为前缀，功能分支

当开发一个需要较长时间或复杂的功能时，可以从开发分支创建一个功能分支，完成的时候，再合并到开发分支，等待下一次发布

**hotfix/** 为前缀，热修复分支

当需要修复已经发布版本的某个问题，但是又不想引入正在开发的新功能，可以从最新的主干创建热修复分支，完成的时候，再合并到开发分支，同时也要合并到开发分支

**release/** 为前缀，发布分支

当需要打包发布新版本，可以从开发分支创建发布分支，为打包发布所做的更新提交到发布分支，完成的时候，再合并到开发分支，同时也要合并到开发分支

**tag** 打标签

每次发布的新版本，都应该用版本号为名称打标签，如 V1.0.1 V1.5.1

#### 推荐资料和工具

* [SourceTree][sourcetreee]
* [Git 在线教程][gitbook]
* [Remove sensitive data][remove-sensitive-data]

[sourcetreee]: http://www.sourcetreeapp.com
[gitbook]: http://git-scm.com/book/zh
[remove-sensitive-data]: https://help.github.com/articles/remove-sensitive-data/
