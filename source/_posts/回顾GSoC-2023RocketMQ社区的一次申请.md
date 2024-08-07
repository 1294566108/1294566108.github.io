---
title: GSoC-2023 RocketMQ 社区的申请全过程
date: 2023-10-14 22:37:03
tags: 开源之夏
---
> GSoC-2023：[官网链接](https://summerofcode.withgoogle.com/)  [组织列表](https://summerofcode.withgoogle.com/programs/2023/organizations)  [时间列表](https://developers.google.com/open-source/gsoc/timeline?hl=zh-cn)
> GSoC-2023：Integrate RocketMQ 5.0 client with Spring [选题链接](https://issues.apache.org/jira/browse/GSOC-122)
> 本次选题相关 ISSUE：[Rocketmq-Clients Issue](https://github.com/apache/rocketmq-clients/issues/275)  [Rocketmq-Spring Issue](https://github.com/apache/rocketmq-spring/issues/553)

# 前言

在 GSoC-2023，我申请的课题是 **RocketMQ-5.0-gRPC 客户端集成 Spring** 这一课题。不过遗憾的是，最后课题并未中选，但是本课题的开发任务还是由我来完成，到目前为止，代码也已经合并到官方仓库 [[PR 链接]](https://github.com/apache/rocketmq-spring/pull/554)。题目虽未能入选，但是仍由我参与开发任务，这与 GSoC 的活动机制有关，在接下来的内容中我会进行详细说明。

## 契机

其实在年初的时候还不知道有这样一个活动，那时候只知道国内的 **开源软件供应链点亮计划（OSPP）** 和 **阿里巴巴编程之夏（ASoC）**，后面在一篇[公众号推文](https://mp.weixin.qq.com/s/VDF-yJ267uHczEO7QNeUqg)中，偶然了解到 GSoC。
这几个开源活动的申请时间线也比较有趣，GSoC在申请结束并公布中选情况后，差不多就到了 OSPP/ASoC 的申请开发时间，OSPP申请截止后，又到了 **GitLink编程夏令营（GLCC）** 的申请时间，所以如果GSoC没用中选也没关系，可以在后面几个活动中继续尝试。**而且某些“落选”的题目，也很可能成为接下来一个活动的题目。**

## 准备

1. **尽快与导师进行联系。** 当选题公布出来的时候，官网里往往会留下导师的邮箱。如果你熟悉这个社区，甚至可以提前写出你关于选题的一些思考和想法，然后与导师进行邮箱交流，这时导师可能会继续给你提出要求，因此你需要继续去深入了解相关部分的代码。如果你能添加上导师的 WeChat，交流起来显然会更方便。不过这不意味着你一定能入选，但是这对你进行下一步的思考以及写提案会有很大的帮助。
2. **选题与竞争。** 既然是开源活动，参与者肯定不少，因此打算做这个选题的人也不止一个。所以往往你还需要去 Github 里找到对应 issue，留言表示自己对该选题的兴趣和意愿，希望进一步沟通。如果你想知道目前这个 Task 的潜在参与者有多少，可以根据社区活跃度与留言人数进行粗略的判断。
3. **进行有思考的对话并大胆提问。** 与导师交流，不管是提问还是回复消息，都需要仔细思考后再行动。如果导师向你进一步提问，你可以不用急着立刻回复他，但是一定要经过深思熟虑后回答。但是这也不意味着你跟导师交流就要怯懦，在某些话题下，例如你好奇目前选题的竞争人数，或者完全不理解题目要求是什么，你可以大胆提问，相信导师会为你释疑。
4. **不建议同时准备太多选题。** 这样会忙于奔波，最后可能一个社区也入选不了。所以你在根据自我能力与项目难度进行综合评估后，选择一个你觉得最合适的选题，把精力集中在一个选题上，这样能够大大提高入选概率。

# 为什么谷歌编程之夏能吸引全球众多开发者

**官网是这样介绍GSoC的：**

> Google Summer of Code is a global, online program focused on bringing new contributors into open source software development. GSoC Contributors work with an open source organization on a 12+ week programming project under the guidance of mentors.


用中文翻译即是，学生会用大概一个暑假的时间在导师的指导下为开源社区写代码，Google 为你的工作支付报酬。而 GSoC 的报酬很让人心动，例如一个 **Medium Size Projects** 可以获得1800 刀，而一个 **Large Size Projects** 可以获得3600 刀，具体表格可以参照 [2023 年贡献者奖金](https://developers.google.com/open-source/gsoc/help/student-stipends)。所以这也导致了，他的筛选条件十分严格，中选难度很大。

![image.png](https://s2.loli.net/2023/10/14/eGkE7bi8XUCBvnw.png)

另外，自2005年以来，谷歌编程之夏计划已经连接了来自112个国家的19000名新的开源贡献者和来自133个国家的18000名导师。Google Summer of Code已经为800个开源组织生成了超过4300万行代码。

# 以学生的视角出发，体悟这次开源活动

> 其实于我而言，很多时候并不知道自己是基于怎样的考量才开始做一件事。自打挣扎着从高考的棺椁里爬出来，才终于见到一点点粘稠的阳光。想想自己一路走来，在负而前驱的道路上，活得虚伪又狼狈。
> 
> 处心积虑想把每一件事都顾虑周全，总会有些左支右绌；然亦步亦趋地跟在人群后面，又总觉得少了些特立独行的影子。但人总是裹挟着平凡在成长，所以我不敢抬头，亦不敢低头，唯有平视才能与自己和解。

大概是今年三月初的时候，偶然得知了 GSoC 这样一个开源活动。但最担心的一个问题是，跟导师和社区都不太熟，给导师发邮件联系不知道他们会不会回复。后面还是鼓起勇气写下了自己对选题不太成熟的理解。迈出第一步，才发现的确是多虑了，社区导师都比较好说话，只要你有想法和思考，他们都愿意和你交流。
![image.png](https://s2.loli.net/2023/10/14/yvP89wTYdh3kUzq.png)

为了让社区导师更好地了解我的情况，可以附上自己的简历，不用像求职简历那样详尽，但是一定要有自己的基本信息（例如姓名和学校），以及为什么能够胜任这个选题的原因（比如是否具有相关能力、技术栈、经历）。另外，基于当时我自己对 rocketmq 浅显的了解，我又写了篇对课题的思考与实现思路，作为附件和简历一起发给了导师。但这些都不是最重要的事情，因为最后提交到官网的**提案**才是你中选的关键。
![image _2_.png](https://s2.loli.net/2023/10/14/Xk39p1xNdYBjPRH.png)

在了解大致开发任务之后，可以到 github 上留言，这意味着让社区导师和潜在参与者可以知道你有当前选题的意向。在这里我们可以清楚地看到有多少人打算做这个题目，并且了解他们的background。
![image.png](https://s2.loli.net/2023/10/14/iYl7Fhyurc6ejPJ.png)

一个比较有意思的点是，我课题的 Issue 下面很是热闹，大多数希望参加是国内的同学，但仍有不少国外的朋友。这也是第一次切身感受到 GSoC 的国际影响力。

另外，在GSoC官网里面还一个比较抽象的统计，详情如下图所示，GSoC 在全世界录取人数最多的 top12 所学校均来自印度。
![image.png](https://s2.loli.net/2023/10/14/75yY138t2AqhBJg.png)

接下来还要完成一件重要的事情，就是熟悉社区现有版本代码的使用。在跑通社区提供的用例之后，才来到编码的核心环节。

总体来说，这次开发任务也并不轻松，因为要同时参考 rocketmq-clients 与 rocketmq-spring 两个仓库的代码。所以在后面将近一个多月的时间里，看着源码反复琢磨着设计思路，之后在完成编码后进行测试用例的覆盖。

在完成这些事情后，距离提案提交截至还有一两个星期左右，我把代码上传到仓库，然后继续完善提案，并准备一份English-Version提案，所以需要做一个 Translation，把最后的 English-Version 提案最后提交到官网。

提案的内容详尽程度是越详细越好，不用担心太长会显得臃肿，如果能贴上代码是更好的。

在提交提案时一定要注意，尽量提前几天提交，不要赶DDL。在截止日期之前，提案是可以被反复修改的。

![image.png](https://s2.loli.net/2023/10/14/O8LKHPBbXiuYaWo.png)

之后，便是一个月的漫长等待时间。因为国际时间的缘故，中国学生收到邮件的时间大概在半夜三点左右。那天一直熬到凌晨三点，并没有消息推送到邮箱。后面实在熬不住睡着了，大概在凌晨五点的时候我突然醒来，收到了GSoC的落选信。
![image.png](https://s2.loli.net/2023/10/14/F41iWEaTQkuoKMU.png)

虽然题目没有中选，但由于我提案的 Ranking 排第一，所以还是有机会继续完成剩下的工作。

不过谈到 GSoC 中选率也是比较感人的，这里以 Apache RocketMQ 和 Apache Dubbo 两个社区为例：

![image.png](https://s2.loli.net/2023/10/15/Q9eufKb4G3WpJdl.png)
![image.png](https://s2.loli.net/2023/10/15/9Ul5dsxXtgRKcjH.png)
上图是 Apache Rocketmq 社区和 Apache Dubbo 社区放置到 GSoC 上的选题，但是今年这两个社区中选的人数都只有2人。

谷歌官方的评选逻辑是，首先导师对每个题目下的提案进行排序，谷歌会给每个组织/社区一个中选名额，题目还会经过谷歌官方的筛选，如果你的题目被谷歌选中了，并且你的排名还在第一，这才意味着中选。但即使最后题目没能被 GSoC 官方选中，如果你的提案被导师认可，仍然可以去完成这个工作，不过就没有 GSOC 的奖金可以拿了。

今年的 GSoC 之旅就以这样的结局收尾了。第一次尝试去参加全球性的开源活动，也算是 coding 路上一件饶有兴趣的事情。

月光如水照缁衣，新的生活还要继续。