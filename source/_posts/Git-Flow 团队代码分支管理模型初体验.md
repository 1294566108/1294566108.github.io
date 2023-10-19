---
title: Git-Flow 团队代码分支管理模型初体验
date: 2023-10-19 17:04:03
tags: Git
---
# **前言**
> 我们知道，Git最初是由Linux开发者Linus用了仅仅两周时间纯C语言编写而成。在日常工作中git是不可或缺的工具。除了需要熟练掌握Git命令，我们还需要了解究竟如何将 Git Flow 这一款适用于代码分支管理到模型应用到团队中，这也是本文的目的。
>
> 大约在13年前（2010年），Vincent Driessen 提出了赫赫有名的 **Git 分支管理模型**。开发者如果要开发一个新的功能（feature），需要从develop分支checkout一个新的feature分支，然后做自己的修改。修改完成后，还需要merge回develop分支。接下来针对Git 分支管理模型，我们在[原文](https://nvie.com/posts/a-successful-git-branching-model/)的基础上，进行一些讨论与分析。


除了像git这样的分布式版本控制系统，n年前还有过 svn、cvs 等等一系列版本控制系统，它们的区别在于：**前者是分布式，后者是集中式**。在传统的 CVS 或 Subversion 中，合并分支是一件让人担惊受怕的事情。
集中式的版本控制系统每次在写代码时都需要从服务器中拉取一份下来，并且如果服务器丢失了，那么所有的就都丢失了。分布式与集中式的区别在于，每个人的电脑都是服务器，你可以自由在本地回滚，提交，当你想把自己的代码提交到主仓库时，只需要合并推送到主仓库就可以了。
**集中式它们都有一个主版本号，所有的版本迭代都以这个版本号为主；而分布式因为每个客户端都是服务器，git没有固定的版本号，但是有一个由哈希算法算出的id，用来回滚用的。** 同时后者也有一个master仓库，这个仓库是一切分支仓库的主仓库，我们可以推送提交到master并合并到主仓库上，主仓库的版本号会迭代一次，我们客户端上的git版本号无论迭代多少次，都跟master无关，只有合并时，master才会迭代一次。 
# **分支管理模型概览**
**我们首先用四段话来概括Vincent Driessen提出的模型的核心内容：**

1. **对于feature分支：** 我们首先需要从 develop 分支 **checkout 一个新的feature分支**，然后做自己的开发/修改。操作完成后，再 **merge 回 develop 分支**。
2. **对于develop分支：** 开发人员将 feature/release 等等辅助分支合并回 develop 分支前，需要成功跑完CI流水线，然后进行合并。develop 分支集成其他分支所做的一切修改。
3. **对于release分支：** 我们需要切一个 release 分支，创建一条新的CD流水线。release 成功结束后，一般会 merge 回 master 分支，**并在master分支打下版本Tag。**
4. **对于hotfix分支：**如果 release 之后发现有bug，需要紧急处理，这时我们就需要**热修复**，从 master 分支的某个 tag 切出来，修复好问题后再 **merge 回 master**，打下新的版本 tag，而且还要同步 **merge 回 develop**。

**模型图如下：**
![](https://cdn.nlark.com/yuque/0/2023/png/25620162/1697460498461-0e89379d-b33b-433b-b08d-26acaeb67086.png#averageHue=%23f1f1f1&clientId=ue1d968ce-7d38-4&from=paste&id=ucd3d2183&originHeight=1524&originWidth=1150&originalType=url&ratio=2&rotation=0&showTitle=false&status=done&style=none&taskId=ueae78598-16fe-4de8-a502-9ff3550592f&title=)
# **主线分支**
Git 中央存储库中包含两个重要的分支，**一个是master，另一个是develop**。它们在项目的生命周期中都一直存在。
这两个分支有如下特性：

1. **origin/master分支**反映的一直都是**发布就绪的状态**。master分支上的代码也是生产服务的代码。原则上，master分支的代码都是可发布的，所以我们对merge到master的代码有严格的要求。
2. **origin/develop分支**反映**当前项目的修改状态。** 该分支集成其他分支所做的一切修改。甚至可以运行一个自动化脚本，每天晚上将各个分支的修改merge到develop分支。当develop分支中的代码趋于稳定，准备发新版的时候，应该将其merger到master分支，并标记本次发布的版本号。

![](https://cdn.nlark.com/yuque/0/2023/png/25620162/1697460498404-90be3392-1089-4779-8119-61942dac2bcc.png#averageHue=%23f3f3f3&clientId=ue1d968ce-7d38-4&from=paste&id=ua0a083b9&originHeight=804&originWidth=534&originalType=url&ratio=2&rotation=0&showTitle=false&status=done&style=none&taskId=u89a5136c-b296-4f65-a0a5-33a2636cf60&title=)
# **辅助分支**
**如 master 和 develop 旁边的其他分支，它们的生命周期有限，最终会从代码库中被移除。** 而我们使用这些分支，主要来帮助各个团队之间并行开发，或者是为新版本发布做准备，亦或修复当前生产环境的bug。
我们使用的分支有以下几种：**Feature branches、Release branches 与 Hotfix branches。** 各个分支根据不同的目的被创建，对它们的操作也遵循严格的规则。比如分支如何创建、开发完成之后merge到的对象等。另外，这些分支其实都是普通的git分支。只是根据我们使用的目的策略给他们赋予了不同的功能。
接下来我们来详细介绍这三种辅助分支。
## **Feature 分支**
**Feature 分支主要用来开新功能。** 一般来说，只要功能还没有开发完善，它就应该一直存在。**但最终应该被 merge 回 develop 分支或者丢弃**。Feature 分支遵循以下规则：

1. 从 develop 分支上创建 feature 分支
2. feature 分支最终 merge 回 develop 分支
3. 分支的命名规则：除了master、develop、release- 、or hotfix- 的任何名字，一般可以用feature-为前缀命名。

**Feature 分支通常只存在于开发人员的版本库中，而不应该存在于 origin 仓库中。** 但考虑到团队成员协作开发的情况，彼此之间需要定期 merge 对方的代码，这是就需要借助 develop 分支来实现了。
![](https://cdn.nlark.com/yuque/0/2023/png/25620162/1697460498469-194addca-9def-4230-b70a-3a027f1c48cb.png#averageHue=%23efefea&clientId=ue1d968ce-7d38-4&from=paste&id=ud2480851&originHeight=714&originWidth=266&originalType=url&ratio=2&rotation=0&showTitle=false&status=done&style=none&taskId=uabb3a25f-0b07-4219-8953-bd4ee0721a3&title=)
### 创建feature分支
```
git checkout -b feature-1020 develop
```
### 合并feature分支
```
git checkout develop            #切换到develop分支
git merge --no-off feature-1020 #合并feature-1020的代码到本地master分支
git branch -d feature-1020      #删除本地feature分支
git push origin develop         #推送到远程develop仓库
```
## **Release 分支**
**Release 分支主要用来为代码发布做准备。** 在合并代码之前，它允许做小的 bug 修改、为版本发布做准备工作（指定版本号、建数据表等）。通过在 Release 分支上做这些操作，可以保证 develop 分支是干净的，不影响当前新功能的开发。Release 分支遵循下面的规则：
1.  从 develop上创建release分支
2.  release 必须 merge 回 develop 和 master
3.  分支需要以 release-* 来命令
当完成本次发版计划的所有功能，并且新功能也到达了预期的状态，那么就是时候创建 release 分支了。这个时候，本次计划发版的所有功能分支，都应该被 merge 回develop 分支。其他的不在本次版本计划中，需要等到下次创建 release 分支的时候再进行 merge。在创建 release 分支的时候，即已经确定了本次发版的版本号。
### 创建release分支
release 分支从 develop 分支中创建。举例说明：当前生产环境的版本是1.1.5，接下来我们计划要发新版。当开发状态基本满足发版的需求时，我们决定本次的版本号为1.2。因为我们创建 release 分支，并给分支指定一个版本号：
```
git checkout -b release-1.2 develop             #新建并签出release-1.2分支
./bump-version.sh 1.2                           #执行虚构的shell脚本，修改部分文件，以反映当前新的版本号
git commit -a -m "Bumped version number to 1.2" #提交当前release-1.2分支版本修改的变更
```
这里 bump-version.sh 是虚构的一个shell脚本，用来修改部分文件，以反映当前新的版本号（这当然也可以手工来修改这些文件）。
直到新版上线之前，release分支都始终应该存在。在这段期间里，还可以继续修改bug（当然是在relase分支上修改，而不是develop）。这个时候，给release分支增加新的功能是被明确禁止的，新的功能必须merge到develop分支，等到下一次版本发布。
### 完成release分支
当 release 分支的状态已经完全可以发版时，我们还需要执行以下操作：

1. release 分支**需要merge到master分支上。** 因为 master 分支上的提交才真正表示一个新的发布版本。
2. 给 master 分支上的这次提交打tag，方便未来参考该历史版本。
3. 代码还**需要 merge 回 develop 分支，** 这样可以保证未来的版本也包含了 release 中的修改。

**合并到master：**
```
git checkout master             //签出master分支
git merge --no-ff release-1.2   //合并release-1.2分支提交的代码
git tag -a 1.2                  //为本次提交打上tag
```

为了保存release中所作的修改，我们还需要将release分支merge到develop：
```
git checkout develop            //签出develop分支
git merge --no-ff release-1.2   //合并release-1.2分支变更到develop
```

这的merge操作可能会导致冲突（因为我们在release中做了修改）。如果真是这样，修复它，然后重新提交。
现在所有工作已经完成，release分支已经不再被需要了。
```
git branch -d release-1.2       //删除release-1.2分支
```
## **Hotfix分支**
**Hotfix分支主要用来修复当前线上出现的Bug。** 和release分支的相同点在于，也是为新的发布版本做准备。但对于该版本，前期却是没有任何计划的。**当生产环境的版本出现不期望的状况并需要立即修复时，Hotfix应运而生。**
![](https://cdn.nlark.com/yuque/0/2023/png/25620162/1697460498522-11d26a05-fdde-4a8e-ae8a-03520ec51e13.png#averageHue=%23f1f1f1&clientId=ue1d968ce-7d38-4&from=paste&id=u02af9cbb&originHeight=852&originWidth=632&originalType=url&ratio=2&rotation=0&showTitle=false&status=done&style=none&taskId=u50d863d0-353c-445b-9a33-c9d39662f2a&title=)
当生产环境出现严重的bug，必须立即去解决。hotfix是从当前生产环境的master分支上的tag标签生成的。hotfix分支遵循下面的规则：
1.  从master分支上创建
2.  最终merge到master和develop分支
3.  分支命名规则为：hotfix-*
Hotfix分支的核心在于：当前开发团队仍然可以继续开发，由另外一个人来快速修复bug。
### 创建hotfix分支
Hotfix从master分支上创建。举个例子，当前生产服务的运行版本是1.2。比较麻烦的是，服务上出现了一个bug，我们需要立即修复。 我们就可以创建一个hotfix分支，着手修改这个bug：
```
git checkout -b hotfix-1.2.1 master                #创建并切换到hotfix-1.2.1分支
./bump-version.sh 1.2.1
git commit -a -m 'Bumped version number to 1.2.1'  #提交当前分支版本修改的变更
```

创建分支之后，不要忘记去做版本修改。然后我们就可以在hotfix分支上一步一步修复这个bug了。
```
git commit -m 'Fixed server production problem'    #提交修改bug后的分支变更
```
### hotfix分支完成修复
当修复完成后，代码需要 merge 到 master 分支和 develop 分支，这样后续的版本中也会包含该修改。这跟 release 分支的操作是完全相同的。

1. 首先，切换到 master 分支，merge 做的修改，然后打标签。
```
git checkout master              #切换到master分支
git merge --no-ff hotfix-1.2.1   #合并分支
git tag -a 1.2.1                 #打标签
```

2. 接下来，将修改merge到develop分支：
```
git checkout develop             #切换到develop分支
git merge --no-ff hotfix-1.2.1   #合并当前修改到develop分支
```
**注意：** 对于这个规则，存在一个例外情况：当前有一个待发布的 release 分支已经存在了。**如果可以延迟到伴随这个 release 发版，才修复这个问题，hotfix 分支就需要 merge 到 release 分支上，而不是develop。** 因为当release分支完成之后，最终修改还是会 merge 到 develop 分支上（如果当前的服务非常需要这个修复，不能等到下次发版，你就还需要merge到develop分支上了，**也就是始终要保证develop包含master分支**）。

3. 最终从代码库中移除分支：
```
git branch -d hotfix-1.2.1
```
