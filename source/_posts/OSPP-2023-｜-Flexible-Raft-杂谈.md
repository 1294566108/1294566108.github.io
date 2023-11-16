---
title: OSPP-2023 ｜ Flexible Raft 杂谈
date: 2023-11-13 02:20:37
tags: 开源之夏
---
<a name="dlWxD"></a>

# 前言

> 在 OSPP-2023 官方选题中，我最终中选了 SOFA Stack 社区中 [结合 NWR 实现 Flexible Raft ](https://summer-ospp.ac.cn/org/prodetail/2395a0390?list=org&navpage=org)这一选题，最近也收到成功结项的消息。
> 关于 Flexible Raft，其实我们最早可以追溯到 Heidi Howard 博士于2016年发布的一篇论文[《改进分布式共识》](https://arxiv.org/pdf/1608.06696v1.pdf)，Howard 是剑桥大学计算机科学与技术系系统研究小组的分布式系统研究员，这篇论文一经发出，受到广泛关注。
> 截至目前，网络上还并没有找到该篇论文的中文翻译，于是我自己把这件事情给做了，详见：[Flexible Paxos：重新审视法定人数交集-论文中译](https://1294566108.github.io/2023/10/26/FlexiblePaxos%EF%BC%9A%E9%87%8D%E6%96%B0%E5%AE%A1%E8%A7%86%E6%B3%95%E5%AE%9A%E4%BA%BA%E6%95%B0%E4%BA%A4%E9%9B%86%E8%AE%BA%E6%96%87%E7%BF%BB%E8%AF%91/)。中译版最初是通过机译之后，再对不太准确的地方进行手工校验修正，文章格式和文字风格最大限度保持了原样，若仍有不准确的地方读者可以留言斧正。
> 该篇论文基于 paxos 算法，提出了一个比 paxos 更 Flexible 的共识协议改进，能够更加灵活的去调整协商和共识效率。而本篇文章将基于 FPaxos 的核心思想进而过渡到 Flexible Raft 的实现思路。

<a name="dko9Q"></a>

# 分布式一致性

分布式系统通常由异步网络连接的多个节点构成，每个节点有独立的计算和存储。通常来说，分布式一致性一般指的是数据的一致性。比如分布式存储系统，通常以多副本冗余的方式实现数据的可靠存储。同一份数据的多个副本必须保证一致，而数据的多个副本又存储在不同的节点中，这里的分布式一致性问题就是存储在不同节点中的数据副本的取值必须一致。**分布式系统最朴实的目标是把一堆普通机器的计算/存储等能力整合到一起，然后整体上能够像一台超级机器一样对外提供可扩展的读写服务。**<br />如果不保证一致性，那么就说明这个系统中的数据根本不可信，数据也就没有意义，那么这个系统也就没有任何价值可言。<br />在分布式系统中，各个节点之间需要进行通信来保持数据的一致性，而通信的实现方式有共享内存和消息传递两种模型。基于消息传递通信模型的分布式系统，不可避免的会发生下面的错误：机器宕机，进程可能会慢、被杀死或者重启，消息可能会延迟、丢失、重复。那么在这种复杂的情况下该如何保证一致性呢？这时就需要交给我们的 Paxos/Raft 算法去实现了。<br />Paxos/Raft 算法能够快速正确的在一个分布式系统中对某个数据的值达成一致，并且保证无论发生任何异常，都不会破坏整个系统的一致性。
<a name="mkSIN"></a>

# Paxos 旧约

在遥远的古希腊，有一座小岛，它位于爱奥尼亚海的东面。当时的人们在岛上进行民主决策实践。我们的主人公 Lamport 早先年考古希腊群岛，就被这座小岛所吸引，可能是从这个背景中获得了灵感，于是，他将小岛的名字 Paxos 用于命名自己提出的共识协议，以表示分布式系统中的协商和共识过程，这就是Paxos 名称的由来。<br />![image.png](https://s2.loli.net/2023/11/13/gs6CvtRdDwA4rVF.png)
<a name="PjJ1w"></a>

## Basic Paxos 算法过程

Propser 有两个重要属性，提案编号 N, 提案 V, 简记 Proposer(N, V)。<br />Acceptor 有三个重要属性，响应提案编号 ResN, 接受的提案编号 AcceptN, 接收的提案 AcceptV， 简记Acceptor（ResN, AcceptN, AcceptV）。
<a name="KeTeP"></a>

### 第一阶段: Prepare准备阶段

**Proposer：**Proposer 生成**全局唯一且递增**的提案编号N，向所有Acceptor发送Prepare请求，这里无需携带提案内容，只携带提案编号即可, 即发送 Proposer(N, null)。<br />**Acceptor：**Acceptor 收到 Prepare 请求后，有两种情况：

1. 如果 Acceptor 首次接收 Prepare 请求, 设置 ResN=N，同时响应ok
2. 如果 Acceptor **不是**首次接收 Prepare 请求，则：

- 若请求过来的提案编号 N**小于等于**上次持久化的提案编号 ResN，则不响应或者响应 error。
- 若请求过来的提案编号 N**大于**上次持久化的提案编号 ResN, 则更新 ResN = N，同时给出响应。响应的结果有两种，如果这个 Acceptor 此前没有接受过提案， 只返回 ok。否则如果这个 Acceptor 此前接收过提案，则返回 ok 和上次接受的提案编号 AcceptN，接收的提案 AcceptV。
  <a name="qDx15"></a>

### 第二阶段: Accept接受阶段

**Proposer：**Proposer 收到响应后，有两种情况：

1. 如果收到了**超过半数**响应 ok , 检查响应中是否有提案，如果有的话，取提案 V= 响应中最大 AcceptN 对应的 AcceptV，如果没有的话，V则有当前 Proposer 自己设定。最后发出 accept 请求，这个请求中携带提案 V。
2. 如果**没有收到超过半数**响应 ok , 则重新生成提案编号 N，重新回到第一阶段，发起 Prepare 请求。

**Acceptor：**Acceptor 收到 accept 请求后，分为两种情况：

1. 如果发送的提案请求 N **大于**此前保存的 RespN，接受提案，设置 AcceptN = N, AcceptV = V, 并且回复 ok。
2. 如果发送的提案请求 N **小于等于**此前保存的 RespN，不接受，不回复或者回复 error。

**Proposer：**Proposer 收到 ok 超过半数，则 V 被选定，否则重新发起 Prepare 请求。
<a name="jW7NP"></a>

### 第三阶段: Learn学习阶段

**Learner：**Proposer 收到多数 Acceptor 的 Accept 后，决议形成，将形成的决议发送给所有 Learner。
<a name="sDgZM"></a>

# Flexible Paxos

<a name="Uq0ok"></a>

## 关于作者

**下面是微软剑桥实验室对Heidi Howard的介绍：**

> 我是**微软剑桥研究院机密计算小组**的高级研究员。我的研究处于分布式计算理论和实践的交叉点，重点是开发弹性和值得信赖的分布式计算机系统。此前，我是**剑桥大学三一大厅**的计算机科学研究员VMware Research的附属/访问研究员和剑桥大学计算机科学与技术系的附属讲师。我于2019年因对分布式共识的研究获得了剑桥大学的博士学位。我最出名的可能是我在Paxos算法方面的工作，特别是 **Flexible Paxos 的发明**。

![image.png](https://s2.loli.net/2023/11/13/Oibz4e9pxr7INm1.png)
<a name="T3PA5"></a>

## 为什么要拥抱FPaxos

我们想象这样一个场景：假设 paxos 集群中成员总数是 n，prepare 阶段参与的成员数是 Q1，accept 阶段参与的成员数是Q2，那么 Basic Paxos 正确性保障最本质的诉求是 Q1 与 Q2 两个阶段要有重合成员，以确保信息可以传递。当前一个普遍的实现机制是：只要 Q1 与 Q2 两个阶段都有多数派（majority）成员参与即可。<br />可是，我们发现只要保证 prepare 与accept 两个阶段参与的成员数超过成员总数，即 Q1 + Q2 > N，那一样能够满足 Q1 与 Q2 两个阶段要有重合成员的要求，如果是这样，那我们选择的参数就非常灵活了。

> 那么，动态调节 Q1 和 Q2 的意义在哪里？我们知道 prepare 阶段事实上可以看作 Leader 选举，accept 阶段则可以看成日志同步，即涉及 IO 操作。我们是否可以考虑把Q1调大，而Q2调小，来减小 IO 操作带来的压力；又或者我们不再局限于多数派模型，动态设置更灵活的两阶段成员数量，来更好的支持异地容灾呢？

我们来举一个具体的例子来更好的理解 Flexible Paxos 的优势。<br />现在有一个 n=5 的集群节点，按照以往的 majority 模式，我们 prepare 和 accept 阶段需要的成员数都是3。如果我们设置FPaxos 「n=5，prepare=4，accept=2」，此时的IO依赖于两个节点存活响应，那么我们是不是IO的压力就小了，日志同步这种慢操作所依赖的节点更少了。<br />另外一个比较经典的例子，AWS曾经提出了可用区（Availability+Zone）的概念，在每个区域 （Region）都有多个可用区。AZ之间物理隔离，独立供电，一个AZ故障，不会影响另外一个AZ，但AZ之间是连通，且网络耗时低。简单可以将AZ理解为独立机房或逻辑机房，这样可以利用AZ的隔离性，对业务进行跨AZ部署，实现高可用。在这样的异地容灾背景下，如果使用多数派模型，像 [2,2,2] 这样的 3 AZ 场景，无论是 prepare 阶段还是 accept 阶段，他最多只能接受挂掉一个 zone，因为 6 节点的多数派是 4。那其实 [2,2,2] 相较于 [2,2,1] 是没有增加额外的容灾能力的，后者也最多只允许宕掉一个 zone。但是如果我们引入了 Flexible Paxos 就不一样了。如果我们将 [2,2,2] 的 Q1 设置为 4，Q2 设置为 3，这样我们能抗住挂掉一个 AZ，这时还剩下 4 个节点，即使宕掉的 AZ 中有 Leader，仍然可以重新选举出一个新的 Leader，并且在accept 阶段还可以另外支持宕掉一个节点，毕竟Q2=3。
<a name="ZEcYX"></a>

# Flexible Raft

有人说过这样一句话：在 Paxos 算法找到一个开源的“杀手级”工业应用之前，Raft 算法的赶超速度几乎是不可阻挡的。Paxos 协议族会逐步成为 Raft 算法的优化策略（所以我们看到这次SOFA-JRaft 在 OSPP-2023 开源之夏给出的选题，[结合 NWR Quorum 模型实现 Flexible Raft](https://summer-ospp.ac.cn/org/prodetail/2395a0390?list=org&navpage=org) 这个课题，说明 Raft 算法已经开始拥抱 Flexible Raft 这个灵活派的新特性了）。<br />对于 Raft 算法来说，其实要像 Flexible Paxos 算法一样实现 Flexible Raft 也是大同小异的。Paxos 的 prepare 阶段类似于 Raft 的 Leader 选举，Paxos 的 Accept 阶段类似于 Raft 的日志复制。那么我们如何在兼容多数派模型的情况下，同时引入 Flexible 模型呢？<br />社区给出的答案是：我们可以结合 Quorum NWR 模型来实现这一设计，具体思考可以参考 SOFA-JRaft 的[讨论](https://github.com/sofastack/sofa-jraft/issues/975)。举个例子，在一个总节点数为 5 的集群中，多数派模型的NWR结构是 [n=5，r=3，w=3]；而 Flexible Raft 的 NWR 可以设计为 [n=5，r=2，w=4] 或者 [n=5，r=4，w=2]，甚至可以 [n=5，r=5，w=1]，只需要满足 r+w = n+1 就好了。<br />所以接下来，我会给出在本次 OSPP-2023 开源之夏活动中，对于 Flexible Raft 的设计与具体实现。
<a name="aIYaI"></a>

## 使用规则

Flexible Raft 的功能使用方法很简单，这里以 jraft-example 模块下我提供给社区的 Flexible Raft 功能测试用例模块为例子：FlexibleRaftServer 中，我们只需要使用 **nodeOptions.enableFlexibleRaft(true)** 打开 Flexible Raft 开关，然后调用 **nodeOptions.setFactor(6, 4)** 对读写 factor 因子进行设置，这样就能够启动 Flexible Raft了。

``` java
    public FlexibleRaftServer(final String dataPath, final String groupId, final PeerId serverId,
                              final NodeOptions nodeOptions) throws IOException {
        // init raft data path, it contains log,meta,snapshot
        FileUtils.forceMkdir(new File(dataPath));

        // here use same RPC server for raft and business. It also can be seperated generally
        final RpcServer rpcServer = RaftRpcServerFactory.createRaftRpcServer(serverId.getEndpoint());
        // GrpcServer need init marshaller
        FlexibleGrpcHelper.initGRpc();
        FlexibleGrpcHelper.setRpcServer(rpcServer);

        // register business processor
        FlexibleRaftService flexibleRaftService = new FlexibleRaftServiceImpl(this);
        rpcServer.registerProcessor(new FlexibleGetValueRequestProcessor(flexibleRaftService));
        rpcServer.registerProcessor(new FlexibleIncrementAndGetRequestProcessor(flexibleRaftService));
        // init state machine
        this.fsm = new FlexibleStateMachine();
        // set fsm to nodeOptions
        nodeOptions.setFsm(this.fsm);
        // set storage path (log,meta,snapshot)
        // log, must
        nodeOptions.setLogUri(dataPath + File.separator + "log");
        // meta, must
        nodeOptions.setRaftMetaUri(dataPath + File.separator + "raft_meta");
        // snapshot, optional, generally recommended
        nodeOptions.setSnapshotUri(dataPath + File.separator + "snapshot");
        // enable flexible raft
        nodeOptions.enableFlexibleRaft(true);
        // set flexible raft factor
        nodeOptions.setFactor(6, 4);
        // init raft group service framework
        this.raftGroupService = new RaftGroupService(groupId, serverId, nodeOptions, rpcServer);
        // start raft node
        this.node = this.raftGroupService.start();
    }
```

我们对 Majority Raft 模式的使用进行了兼容，如果需要使用 Majority Raft，可以使用下面代码给出的方式设置开启 FlexibleRaft 为 false，或者更简单的方法不进行任何对FlexibleRaft 参数的操作，因为 Jraft 默认是使用 Majority Raft 的。

``` java
nodeOptions.enableFlexibleRaft(false);
```

<a name="IDvDs"></a>

## 设计思路

<a name="VrQya"></a>

### NWR 模型

我们知道，NWR 是一种在分布式存储系统中用于控制一致性级别的策略。N 在分布式存储系统中，代表有多少份备份数据 。W 代表一次成功的更新操作要求至少有 W 份数据写入成功。R 代表一次成功的读数据操作要求至少有 R 份数据成功读取。<br />满足 W + R > N 的情况下，读 Quorum 和写 Quorum 一定存在交集，这个相交的成员一定拥有最新的数据，那么这个分布式系统一定是满足强一致性的。<br />例如，在一个 N=3 的 Raft 集群中，W 是 2、R 是 2 的时候，W+R>N，这种情况对于集群而言就是强一致性的。<br />![image.png](https://s2.loli.net/2023/11/13/IDzW7ST5vKuXso8.png)<br />所以基于以上思路，首先我们要对 Quorum 这一核心类进行设计。Quorum 类结构很简单，只需要{int w，int r} 来表示读写节点数量即可。

```
public class Quorum {
    private int w;

    private int r;

    public Quorum(int w, int r) {
        this.w = w;
        this.r = r;
    }
}
```

<a name="P2Up7"></a>

### Configuration 升级

设计好 Quorum 类之后，如何在每个节点运行过程中随时获取或者修改？这时我们需要整合到 Configuration 类中了。在原有的 Configuration 类中，是没有 quorum、readFactor、writeFactor 和 isEnableFlexible 的，因为这几个属性是为了适配 Fleixble Raft 而新添加的元素。

```java
public class Configuration implements Iterable<PeerId>, Copiable<Configuration> {

    private static final Logger   LOG              = LoggerFactory.getLogger(Configuration.class);

    private static final String   LEARNER_POSTFIX  = "/learner";

    private Quorum                quorum;

    private Integer               readFactor;

    private Integer               writeFactor;

    private Boolean               isEnableFlexible = false;

    private List<PeerId>          peers            = new ArrayList<>();

    // use LinkedHashSet to keep insertion order.
    private LinkedHashSet<PeerId> learners         = new LinkedHashSet<>();
}
```

<a name="oH6Lm"></a>

### Quorum 计算规则

我们根据使用规则可以知道，用户端要使用 Flexible Raft 功能需要打开 NodeOptions 中 Flexible Raft 的开关，并且设置好 factor 因子的大小。那么我们又是如何根据 factor 因子计算出一个 raft 集群的w和r是多少呢？<br />为了实现 w 和 r 的计算逻辑，我们提供了一个计算工厂BallotFactory类专门处理NWR的计算规则。

``` java
checkValid：校验读写factor是否都不为空，并且其和为10
calculateWriteQuorum：根据size与writeFactor计算w的大小
calculateReadQuorum：根据计算出的w，根据公式 r = n - w + 1 计算
calculateMajorityQuorum：计算 Majority 模式下的 w 和 r
buildFlexibleQuorum：如果打开 Flexible 模式，用于计算该模式下的 Quorum
buildMajorityQuorum：如果打开 Majority 模式，用于计算该模式下的 Quorum
convertConfigToLogEntry：将 configuration 转换为 LogEntry
convertOldConfigToLogOuterEntry：将 oldConfiguration 转换为 LogEntry
```

```java
public final class BallotFactory {
    private static final Logger     LOG                  = LoggerFactory.getLogger(BallotFactory.class);
    private static final String     defaultDecimalFactor = "0.1";
    private static final BigDecimal defaultDecimal       = new BigDecimal(defaultDecimalFactor);

    public static Quorum buildFlexibleQuorum(Integer readFactor, Integer writeFactor, int size) {
        // if size equals 0,config must be empty,so we just return null
        if (size == 0) {
            return null;
        }
        // Check if factors are valid
        if (!checkValid(readFactor, writeFactor)) {
            LOG.error("Invalid factor, factor's range must be (0,10) and the sum of factor should be 10");
            return null;
        }
        // Partial factor is empty
        if (Objects.isNull(writeFactor)) {
            writeFactor = 10 - readFactor;
        }
        if (Objects.isNull(readFactor)) {
            readFactor = 10 - writeFactor;
        }
        // Calculate quorum
        int w = calculateWriteQuorum(writeFactor, size);
        int r = calculateReadQuorum(readFactor, size);
        return new Quorum(w, r);
    }

    public static Quorum buildMajorityQuorum(int size) {
        // if size equals 0,config must be empty,so we just return null
        if (size == 0) {
            return null;
        }
        int majorityQuorum = calculateMajorityQuorum(size);
        return new Quorum(majorityQuorum, majorityQuorum);
    }

    private static int calculateWriteQuorum(int writeFactor, int n) {
        BigDecimal writeFactorDecimal = defaultDecimal.multiply(new BigDecimal(writeFactor))
            .multiply(new BigDecimal(n));
        return writeFactorDecimal.setScale(0, RoundingMode.CEILING).intValue();
    }

    private static int calculateReadQuorum(int readFactor, int n) {
        int writeQuorum = calculateWriteQuorum(10 - readFactor, n);
        return n - writeQuorum + 1;
    }

    private static int calculateMajorityQuorum(int n) {
        return n / 2 + 1;
    }

    public static boolean checkValid(Integer readFactor, Integer writeFactor) {
        if (Objects.isNull(readFactor) || Objects.isNull(writeFactor)) {
            LOG.error("When turning on flexible mode, Both of readFactor and writeFactor should not be null.");
            return false;
        }
        if (readFactor + writeFactor == 10 && readFactor > 0 && readFactor < 10 && writeFactor > 0 && writeFactor < 10) {
            return true;
        }
        LOG.error("Fail to set quorum_nwr because the sum of read_factor and write_factor is {} , not 10",
            readFactor + writeFactor);
        return false;
    }

    public static LogEntry convertConfigToLogEntry(LogEntry logEntry, Configuration conf) {
        if (Objects.isNull(logEntry)) {
            logEntry = new LogEntry();
        }
        logEntry.setEnableFlexible(false);
        logEntry.setPeers(conf.listPeers());
        final LogOutter.Quorum.Builder quorumBuilder = LogOutter.Quorum.newBuilder();
        LogOutter.Quorum quorum = quorumBuilder.setR(conf.getQuorum().getR()).setW(conf.getQuorum().getW()).build();
        logEntry.setQuorum(quorum);
        return logEntry;
    }

    public static LogEntry convertOldConfigToLogOuterEntry(LogEntry logEntry, Configuration conf) {
        if (Objects.isNull(logEntry)) {
            logEntry = new LogEntry();
        }
        logEntry.setEnableFlexible(false);
        logEntry.setOldPeers(conf.listPeers());
        final LogOutter.Quorum.Builder quorumBuilder = LogOutter.Quorum.newBuilder();
        LogOutter.Quorum quorum = quorumBuilder.setR(conf.getQuorum().getR()).setW(conf.getQuorum().getW()).build();
        logEntry.setOldQuorum(quorum);
        return logEntry;
    }
}
```

<a name="nVrqZ"></a>

### Node 节点初始化

启动节点后，一定会调用 NodeImpl#init 方法来初始化一个节点。在原有初始化init方法的基础上，额外添加下面的 Flexible 逻辑以适配灵活派模型。

``` java
// 省略非 Flexible 模式下的相关代码
Configuration initialConf = options.getInitialConf();
    	// 判断开启 Flexible 模式后，校验设置的 factor 是否合规
        if (initialConf.isEnableFlexible()
            && !checkFactor(initialConf.getWriteFactor(), initialConf.getReadFactor())) {
            return false;
        }
        this.conf = new ConfigurationEntry();
        this.conf.setId(new LogId());
        // if have log using conf in log, else using conf in options
        if (this.logManager.getLastLogIndex() > 0) {
            checkAndSetConfiguration(false);
        } else {
            this.conf.setConf(this.options.getInitialConf());
            // initially set to max(priority of all nodes)
            this.targetPriority = getMaxPriorityOfNodes(this.conf.getConf().getPeers());
        }

        if (!this.conf.isEmpty()) {
            Requires.requireTrue(this.conf.isValid(), "Invalid conf: %s", this.conf);
        } else {
            LOG.info("Init node {} with empty conf.", this.serverId);
        }
        // 判断开启 Majority Mode 后，设置 Majority Quorum
        if (Objects.isNull(conf.getConf().getQuorum()) && !conf.getConf().isEnableFlexible()) {
            Quorum quorum = BallotFactory.buildMajorityQuorum(conf.getConf().size());
            conf.getConf().setQuorum(quorum);
        }

        // 初始化预投票选票Ctx：prevVoteCtx
        if (!prevVoteCtx.init(conf.getConf(), conf.getOldConf())) {
            LOG.error("Fail to init prevVoteCtx.");
            return false;
        }
        // 初始化投票选票Ctx：voteCtx
        if (!voteCtx.init(conf.getConf(), conf.getOldConf())) {
            LOG.error("Fail to init voteCtx.");
            return false;
        }
```

<a name="YvBSV"></a>

### 写操作约束

我们根据选择的 Majority/Flexible 的计算规则，生成的 WriteQuorum 最终会在初始化 Ballot 的时候赋值到 quorum 属性。Ballot 选票会在上文提到的 preVoteCtx 与 voteCtx 中进行初始化，用于投票计算，当收到任意节点的投票后，quorum 大小减一。

``` java
    public boolean init(final Configuration conf, final Configuration oldConf) {
        this.peers.clear();
        this.oldPeers.clear();
        this.quorum = this.oldQuorum = 0;
        int index = 0;
        if (conf != null && !conf.isEmpty()) {
            for (final PeerId peer : conf) {
                this.peers.add(new UnfoundPeerId(peer, index++, false));
            }
            // 获取conf中的Quorum.W
            quorum = conf.getQuorum().getW();
        }

        if (oldConf == null) {
            return true;
        }
        index = 0;
        for (final PeerId peer : oldConf) {
            this.oldPeers.add(new UnfoundPeerId(peer, index++, false));
        }

        if (!oldConf.isEmpty()) {
            this.oldQuorum = oldConf.getQuorum().getW();
        }
        return true;
    }
```

对于投票是否完成并达成共识，可以使用 isGranted 方法去判断。

``` java
    public boolean isGranted() {
        return this.quorum <= 0 && this.oldQuorum <= 0;
    }
```

<a name="eFcUc"></a>

### 读操作约束

在 ReadIndexHeartbeatResponseClosure 中定义了心跳响应的属性，如下述代码所示。failPeersThreshold 参数即允许心跳失败的阈值，可以看到使用到了我们提供的quorum.getR() 来获取 Read Quorum。

``` java
        public ReadIndexHeartbeatResponseClosure(final RpcResponseClosure<ReadIndexResponse> closure,
                                                 final ReadIndexResponse.Builder rb, final Quorum quorum,
                                                 final int peersCount) {
            super();
            this.closure = closure;
            this.respBuilder = rb;
            this.quorum = quorum;
            this.failPeersThreshold = peersCount - quorum.getR() + 1;
            this.ackSuccess = 0;
            this.ackFailures = 0;
            this.isDone = false;
        }
```

可以看到 ReadIndexHeartbeatResponseClosure 心跳响应回调重写了 run 方法，用来判断读请求的操作是否还满足心跳要求，即当前 leader 是否还是 leader。

``` java
        @Override
        public synchronized void run(final Status status) {
            if (this.isDone) {
                return;
            }
            if (status.isOk() && getResponse().getSuccess()) {
                this.ackSuccess++;
            } else {
                this.ackFailures++;
            }
            // Include leader self vote yes.
            if (this.ackSuccess + 1 >= this.quorum.getR()) {
                LOG.info("Reading successfully...");
                this.respBuilder.setSuccess(true);
                this.closure.setResponse(this.respBuilder.build());
                this.closure.run(Status.OK());
                this.isDone = true;
            } else if (this.ackFailures >= this.failPeersThreshold) {
                LOG.info("Reading failed...");
                this.respBuilder.setSuccess(false);
                this.closure.setResponse(this.respBuilder.build());
                this.closure.run(Status.OK());
                this.isDone = true;
            }
        }
```

<a name="KIDDh"></a>

### 节点如何宣判死亡

在之前的 Majority 模式下，我们判断节点死亡的方式很简单，只需要根据多数派节点数是否存活即可下决定，或者说参考 Read Quorum。<br />但当我们引入了 Flexible 模式，情况就不同了。因为我们希望在写多读少的场景下，如果死亡节点数即使不再满足 Read Quorum 的数量，仍然不能让leader下线，此时系统是可以继续写的。所以我们的判断逻辑就与之前截然不同了，在最少存活节点数的选择上，我们会选择 Read Quorum/ Write Quorum 的最小值。下面的源码中 **targetCount** 的计算方式即是兼容两种模式的实现。<br />可能有人会问这会不会影响 Majority 模型呢？答案是不会，因为多数派模式下的 ReadQuorum 和 WriteQuorum 大小是相等的。所以新增的这个判断适配的是灵活排模型。

``` java
    private boolean checkDeadNodes0(final List<PeerId> peers, final long monotonicNowMs, final boolean checkReplicator,
                                    final Configuration deadNodes) {
        final int leaderLeaseTimeoutMs = this.options.getLeaderLeaseTimeoutMs();
        int aliveCount = 0;
        long startLease = Long.MAX_VALUE;
        for (final PeerId peer : peers) {
            if (peer.equals(this.serverId)) {
                aliveCount++;
                continue;
            }
            if (checkReplicator) {
                checkReplicator(peer);
            }
            final long lastRpcSendTimestamp = this.replicatorGroup.getLastRpcSendTimestamp(peer);
            if (monotonicNowMs - lastRpcSendTimestamp <= leaderLeaseTimeoutMs) {
                aliveCount++;
                if (startLease > lastRpcSendTimestamp) {
                    startLease = lastRpcSendTimestamp;
                }
                continue;
            }
            if (deadNodes != null) {
                deadNodes.addPeer(peer);
            }
        }

        // If the writeFactor in a cluster is less than readFactor and the number of nodes
        // is less than r and greater than or equal to w, we hope to still be in a writable state.
        // Therefore, read requests may fail at this time, but the cluster is still available
        Quorum quorum = this.conf.getConf().getQuorum();
        int targetCount = this.conf.getConf().isEnableFlexible() && quorum.getW() < quorum.getR() ? quorum.getW()
            : quorum.getR();
        if (aliveCount >= targetCount) {
            updateLastLeaderTimestamp(startLease);
            return true;
        }
        return false;
    }
```

<a name="LDxkK"></a>

### 灵活派属性持久化

当节点宕机后重启，需要知道下线前节点是 flexible mode 还是 majority mode。设置的 Read Factor 与 Write Factor大小是多少等等一系列参数。所以我们需要对这些参数做一个持久化。<br />在 log.proto 中添加了适配 flexible raft 的新字段：

```protobuf
message Quorum{
  optional int32 w = 1;
  optional int32 r = 2;
}

message PBLogEntry {
  required EntryType type = 1;
  required int64 term = 2;
  required int64 index = 3;
  repeated bytes peers = 4;
  repeated bytes old_peers = 5;
  required bytes data = 6;
  optional int64 checksum = 7;
  repeated bytes learners = 8;
  repeated bytes old_learners = 9;
  optional int32 read_factor = 10;
  optional int32 write_factor = 11;
  optional int32 old_read_factor = 12;
  optional int32 old_write_factor = 13;
  optional bool  is_enable_flexible = 14;
  optional Quorum quorum = 15;
  optional Quorum old_quorum = 16;
}
```

同理的，在 raft.proto 也需要添加适配灵活派新字段：

```protobuf
message EntryMeta {
    required int64 term = 1;
    required EntryType type = 2;
    repeated string peers = 3;
    optional int64 data_len = 4;
    // Don't change field id of `old_peers' in the consideration of backward
    // compatibility
    repeated string old_peers = 5;
    // Checksum fot this log entry, since 1.2.6, added by boyan@antfin.com
    optional int64 checksum = 6;
    repeated string learners = 7;
    repeated string old_learners = 8;
    optional int32 read_factor = 9;
    optional int32 write_factor = 10;
    optional int32 old_read_factor = 11;
    optional int32 old_write_factor = 12;
    optional bool  isEnableFlexible = 13;
    optional Quorum quorum = 14;
    optional Quorum old_quorum = 15;
};

message SnapshotMeta {
    required int64 last_included_index = 1;
    required int64 last_included_term = 2;
    repeated string peers = 3;
    repeated string old_peers = 4;
    repeated string learners = 5;
    repeated string old_learners = 6;
    optional int32 read_factor = 7;
    optional int32 write_factor = 8;
    optional int32 old_read_factor = 9;
    optional int32 old_write_factor = 10;
    optional bool isEnableFlexible = 11;
    optional Quorum quorum = 12;
    optional Quorum old_quorum = 13;
}
```

V1/V2两个版本的编码解码器(encoder/decoder)也需要适配新规则，具体代码此处就不贴了。
<a name="PIXjI"></a>

## 一致性检测

我们书写的 Flexible Raft 是否满足线性一致性呢，这需要我们使用 Jepsen 框架去进行校验了。具体的工作我也有形成文档，可以参考我之前这篇博客：[JRaft-jepsen 线性一致性测试](https://1294566108.github.io/2023/09/11/JRaft-jepsen-%E7%BA%BF%E6%80%A7%E4%B8%80%E8%87%B4%E6%80%A7%E6%B5%8B%E8%AF%95/)。<br />最终的检测结果已经以PR的形式提交到Jraft-jepsen仓库，详见：[https://github.com/sofastack/sofa-jraft-jepsen/pull/3](https://github.com/sofastack/sofa-jraft-jepsen/pull/3)。
<a name="FqOno"></a>

# 写在最后

感谢中科院软件所"开源软件供应链点亮计划"提供这次宝贵的机会，让我有机会实现 SOFA-JRaft 这个 flexible raft 新特新。<br />同时也要感谢刘源远导师和 SOFA-Stack 社区在项目开发期间给予的帮助，让我更好地去完成并完善这次开发。<br />在这里还要推荐一本书：[《深入理解分布式共识算法》](https://baike.baidu.com/item/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3%E5%88%86%E5%B8%83%E5%BC%8F%E5%85%B1%E8%AF%86%E7%AE%97%E6%B3%95)，它帮助我更好地理解 Raft 算法并深入了 SOFA-JRaft 的源码学习，希望它也能帮助大家更好的去了解、学习分布式共识算法。