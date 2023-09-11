---
title: JRaft-jepsen 线性一致性测试
date: 2023-09-11 13:47:03
tags: SOFA-JRaft
---

**Jepsen前置资料参考：**[Jepsen中文官方文档](https://jaydenwen123.gitbook.io/zh_jepsen_doc/)   [Jepsen作者博客](https://aphyr.com/)
**Cloujre前置资料：**[Clojure入门文章-英文版](https://aphyr.com/posts/301-clojure-from-the-ground-up-welcome)   [Clojure中文文档](https://siddontang.gitbooks.io/lean-clojure/content/hello-clojure.html)

# 线性一致性概述

### 概念

如果说你对线性一致性（Linearizability）概念不太熟，那一定知道强一致性（strong consistency），或者说原子一致性（atomic consistency），也可以理解为的 CAP 理论中的 C。

### Raft线性一致性的实现

#### 线性一致性写

所有的 read/write 都会来到 Leader，write 会有 Log 被序列化，依次顺序往后 commit，并 apply 然后在返回，那么一旦一个 write 被 committed，那么其前面的 write 的 Log 一定就被 committed 了。 所有的 write 都是有严格的顺序的，一旦被 committed 就可见了，所以 Raft 是线性一致性写。

#### 线性一致性读

线性一致性读有很多种方法可以去实现，例如以下介绍了四种实现线性一致性读的办法：

1. Raft Log read：每个 read 都有一个对应的 Log，和 write 一样，将非事务请求以事务请求的逻辑去进行处理。在 Read Log 被 Apply 的时候读，那么此时这个 read Log 之前的 write Log 也一定被 applied 了，那么读到的数据一定是最新的。
2. ReadIndex：我们知道 Raft log read，会有 Raft read log 的复制和提交的开销，所以出现了 ReadIndex。当 read 请求发送给 Leader 的时候：**（1）首先需要确认 read 必须返回最新 committed 的结果**。但是一个节点刚当选 Leader 的时候并不知道最新的 committed index，这个时候需要提交一个 Noop Log Entry 来提交之前的 Log Entry，然后开始 Read；**（2）确认当前的 Leader 是不是还是 Leader**。可能由于网络分区，这个 Leader 已经被孤立了，所以 Leader 在返回 read 之前，先和 Replica-Group 的其他成员发送 heartbeat 确定自己 Leader 的身份；通过上述两条才可以保证读到的是最新刚被 committed 的数据。
3. Lease read：主要是通过 lease 机制维护 Leader 的状态，来减少了 ReadIndex 每次 read 发送 heartheat 的开销。
4. Follower read：先去 Leader 查询最新的 committed index，然后拿着 committed Index 去 Follower read，从而保证能从 Follower 中读到最新的数据，当前 Etcd 和 SOFA-Jraft 就实现了 Follower read。

关于SOFA-JRaft实现线性一致性读可参考文章：[SOFAJRaft 线性一致读实现剖析](https://www.sofastack.tech/blog/sofa-jraft-linear-consistent-read-implementation/)

# Jepsen 概述

Jepsen 是由 Kyle Kingsbury 采用函数式编程语言 Clojure 编写的验证分布式系统一致性的测试框架，作者使用它对许多著名的分布式系统（etcd, cockroachdb...）进行了“攻击”（一致性验证），并且帮助其中的部分系统找到了 bug。
网上已有文章对其原理进行过简述，此处贴上链接即可：[当 TiDB 遇上 Jepsen](https://juejin.cn/post/6844903491442327566)

# Jraft-Jepsen 部署验证一致性

> 下面的内容是作者自己踩坑总结出来的部署流程，介绍了如何手动部署一套Jepsen框架对JRaft代码进行一致性验证。

### 打开gitpod/linux

运行ubuntu容器，其中1个control节点，5个jraft-test节点。

```shell
docker run -itd --name ubuntu-1 --hostname control --privileged=true ubuntu
docker run -itd --name ubuntu-2 --hostname jraft1 --privileged=true ubuntu
docker run -itd --name ubuntu-3 --hostname jraft2 --privileged=true ubuntu
docker run -itd --name ubuntu-4 --hostname jraft3 --privileged=true ubuntu
docker run -itd --name ubuntu-5 --hostname jraft4 --privileged=true ubuntu
docker run -itd --name ubuntu-6 --hostname jraft5 --privileged=true ubuntu
```

查找所有容器的ip

```shell
docker inspect --format='{{.Name}} - {{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' $(docker ps -aq)
```

进入容器

```shell
docker exec -it ubuntu-1 bash
```

修改hosts文件，添加ip与域名的对应关系

```shell
vim /etc/hosts
```

此处根据查询出的容器ip对应填写即可

```shell
127.0.0.1       localhost
::1     localhost ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
172.17.0.2      control
172.17.0.3      jraft1
172.17.0.4      jraft2
172.17.0.5      jraft3
172.17.0.6      jraft4
172.17.0.7      jraft5
```

### 设置Docker-SSH免密登录

安装SSH服务

```shell
apt-get update 
apt-get install openssh-server //安装ssh服务
```

开启SSH服务

```shell
/etc/init.d/ssh start
ps -e | grep ssh  //检查是否开启ssh
```

#### jraft-test节点

在Test-Node的docker容器内，编辑文件/etc/ssh/sshd_config，添加一行PermitRootLogin yes表示ssh允许root登录。

```shell
echo "PermitRootLogin yes" >> /etc/ssh/sshd_config
# 或者 vim /etc/ssh/sshd_config 并手敲一行PermitRootLogin yes
```

随后一定要重启ssh服务

```shell
service ssh restart
```

设置root密码

```shell
passwd root
```

#### control节点

```shell
ssh-keygen //生成公钥私钥
ssh-copy-id -i /root/.ssh/id_rsa.pub root@jraft1
ssh-copy-id -i /root/.ssh/id_rsa.pub root@jraft2
ssh-copy-id -i /root/.ssh/id_rsa.pub root@jraft3
ssh-copy-id -i /root/.ssh/id_rsa.pub root@jraft4
ssh-copy-id -i /root/.ssh/id_rsa.pub root@jraft5
```

私钥文件格式问题：您需要确保您的SSH私钥文件格式正确，并且Jepsen测试工具可以正确识别它。通常，SSH私钥文件格式为OpenSSH格式。您可以尝试使用以下命令将私钥文件转换为OpenSSH格式：

```shell
ssh-keygen -p -f /root/.ssh/id_rsa -m pem -t rsa
```

### Ubuntu工具安装

control节点安装：

```shell
apt-get update 
apt-get install leiningen
apt-get install wget
apt-get install git
apt-get install vim
apt-get install sudo
apt-get install iptables
```

test节点安装：

```shell
apt-get update 
apt-get install leiningen
apt-get install wget
apt-get install vim
apt-get install sudo
apt-get install iptables
```

**ubuntu容器安装jdk-8：**
进入目录：

```shell
mkdir /usr/local/java && cd /usr/local/java
```

wget下载jdk8：

```shell
wget --no-cookies --no-check-certificate --header "Cookie: gpw_e24=http%3A%2F%2Fwww.oracle.com%2F; oraclelicense=accept-securebackup-cookie" "http://download.oracle.com/otn-pub/java/jdk/8u141-b15/336fa29ff2bb4ef291e347e091f7f4a7/jdk-8u141-linux-x64.tar.gz"
```

解压：

```shell
tar -zxvf jdk-8u141-linux-x64.tar.gz -C /usr/local/java
```

编辑配置文件：此处需要注意，有可能在安装其他安装包时会自动安装jdk11，此处需要修改两个配置文件才能生效，并且修改为jdk8.

```shell
vim ~/.bashrc
```

```shell
export JAVA_HOME=/usr/local/java/jdk1.8.0_141
export PATH=$JAVA_HOME/bin:$PATH
```

```shell
vim /etc/profile
```

```shell
export JAVA_HOME=/usr/local/java/jdk1.8.0_141
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
export PATH=${JAVA_HOME}/bin:$PATH
```

执行命令使配置文件生效：

```shell
source /etc/profile
```

### 安装clojure-control

**此操作在control节点完成：**
复制链接中的shell脚本：[https://raw.githubusercontent.com/killme2008/clojure-control/master/bin/control](https://raw.githubusercontent.com/killme2008/clojure-control/master/bin/control)

```shell
cd ~/bin
vim control
```

将shell脚本粘贴到control中

```shell
chmod 777 control
```

设置control系统变量

```shell
export CONTROL_ROOT=1
```

```shell
export PATH=$PATH:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/root/bin
```

### 下载jraft测试代码

**克隆远程仓库代码：**

```shell
git clone jraft仓库（自测仓库/官方仓库）
```

**安装jar到本地仓库：**

```shell
mvn clean install -DskipTests=true
```

### 部署atomic-server

**此操作在control节点执行：**

```shell
control run jraft build
control run jraft deploy
```

### 开启测试

**bash开启测试**
configuration-test

```shell
bash run_test.sh --testfn configuration-test --username root --password 123 --ssh-private-key /root/.ssh/id_rsa
```

bridge-test

```shell
bash run_test.sh --testfn bridge-test --username root --password 123 --ssh-private-key /root/.ssh/id_rsa
```

pause-test

```shell
bash run_test.sh --testfn pause-test --username root --password 123 --ssh-private-key /root/.ssh/id_rsa
```

crash-test

```shell
bash run_test.sh --testfn crash-test --username root --password 123 --ssh-private-key /root/.ssh/id_rsa
```

partition-test

```shell
bash run_test.sh --testfn partition-test --username root --password 123 --ssh-private-key /root/.ssh/id_rsa
```

partition-majority-test

```shell
bash run_test.sh --testfn partition-majority-test --username root --password 123 --ssh-private-key /root/.ssh/id_rsa
```

**测试方法**

```shell
configuration-test
bridge-test
pause-test
crash-test
partition-test
partition-majority-test
```

### 潜在问题及解决方案

如果出现报错：

```shell
Could not find artifact apache-codec:commons-codec:jar:1.2 in central (https://repo1.maven.org/maven2/)
Could not find artifact apache-codec:commons-codec:jar:1.2 in clojars (https://repo.clojars.org/)
This could be due to a typo in :dependencies, file system permissions, or network issues.
If you are behind a proxy, try setting the 'http_proxy' environment variable.
Uberjar aborting because jar failed: Could not resolve dependencies
```

解决办法：

```shell
cd ~/.m2/repository/apache-codec/commons-codec/1.2
```

可以看到里面是空的，此时拉取远程仓库jar包即可

```shell
wget https://repo1.maven.org/maven2/commons-codec/commons-codec/1.2/commons-codec-1.2.pom
wget https://repo1.maven.org/maven2/commons-codec/commons-codec/1.2/commons-codec-1.2.jar
```