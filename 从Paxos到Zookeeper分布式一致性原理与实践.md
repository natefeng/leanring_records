# 1 丶分布式架构

## 1.1 从集中式到分布式

集中式系统具有明显的单点问题，大型主机虽然在性能和稳定性方面表现卓越，但是这并不代表永远都不会出故障。一 旦整个系统将处于不可用状态，其后果相当严重。

### 1.1.1 集中式的特点

所谓的集中式系统就是一台或者多台主计算机组成中心节点，数据几种存储在这个中心节点中，并且整个系统的所有业务单元都集中部署在整个中心节点上，系统的所有功能均由其集中处理。在集中式系统中，每个终端或者客户端机器仅仅负责数据的录入和输出，而数据的存储与控制处理完全交由主机来完成。

集中式最大的特点就是部署结构简单。由于集中式系统往往基于底层性能卓越的大型主机，因此无需考虑对服务进行多个节点的部署，也就不需要考虑多个节点之间的分布式协作问题

### 1.1.2 分布式的特点

在**《分布式系统概念与设计》**这本书中，对分布式系统做了如下定义：

分布式系统是一个硬件或者软件组件分布在不同的网络计算机上，彼此之间仅仅通过消息传递进行通信和协调的系统

特征：

- **分布性**

  分布式系统中的多台计算机都会在空间上随意分布，同时，机器的分布情况也会随时变动。

- **对等性**

  分布式系统中的计算机没有主/从之分，即没有控制整个系统的主机，也没有被控制的从机，组成分布式系统的所有结点都是对等的。副本是分布式系统最常见的概念之一，指的是分布式系统中对数据和服务提供的一种冗余方式，在常见的分布式系统中，为了提供高可用的服务，往往会对数据和服务进行副本处理。数据副本是指在不同的节点上持久化同一份数据，当一个节点上存储的数据丢失时，可用从副本上读取数据。另一类副本就是服务副本，指多个节点提供同样的服务。

- **并发性**

  分布式多个节点中，可能会并发的操作一些共享的资源，诸如数据库存储和分布式存储。

- **缺乏全局时钟**

  分布式系统是由一系列在空间上随意分布的多个进程组成的，具有明显的分布性，这些进程之间通过交换消息来互相通信。因此，在分布式系统中，很难定义两个事件谁先谁后，原因就是分布式系统中缺乏一个全局的时钟序列控制。

  

### 1.1.3 分布式环境的各种问题

**通信异常**

从集中式向分布式演变的过程中，必然引入了网络因素，而由于网络的不可靠性，因此也引入了额外的问题。分布式系统中需要各个节点进行通信，都会伴随这网络不可用的风险。并且由于网络通信，操作也会延时更久，单机内存访问明显要比网络通信快许多。因此也会影响消息收发的过程。

**网络分区**

网络发生异常情况，导致组成分布式的所有结点，只有部分节点可以进行正常通信，而另外一些不能。我们将这个现象称为脑裂。当出现网络分区的时候，分布式系统会出现局部小集群。这些局部小集群会独立完成原本需要整个分布式系统才能完成的功能，包括对数据的事务处理，这就对分布式一致性提出了非常大的挑战

**三态**

分布式系统的每一次请求和响应，都存在三态的概念，成功丶失败丶超时。

传统的单机系统中，应用程序在调用一个函数，能够得到一个非常准确的回应，成功或者失败。

​        超时现象

- 网络，请求的消息没有到接收方
- 接收方响应的消息请求方没收到

**节点故障**

## 1.2 从ACID到CAP/BASE

分析分布式事务处理与数据一致性上遇到的种种挑战

### 1.2.1 ACID

事务的ACID

- 原子性
- 一致性
- 隔离性
- 持久性

四种隔离级别

- 读未提交 造成脏读
- 读已提交 造成幻读
- 可重复读 可以使用Mvcc+行锁+间隙锁解决幻读
- 串行化  效率低，但是不会造成数据问题

### 1.2.2 分布式事务

单机数据库中，很容易实现一套满足ACID特性的事务处理系统。但是在分布式数据库中，数据分散在不同的机器中，如何对这些数据进行分布式的事务处理具有非常大的挑战

设想一个最典型的分布式事务场景，跨银行的转账操作设计调用两个异地的银行服务，其中一个需要取款，另外一个需要存款。两个服务共同构成了一个完整的分布式事务，但是由于某种原因存款服务失败了，那么取款服务必须进行回滚，否则用户发现自己的钱不见了。

可以看到，一个分布式事务可以看作是多个分布式的操作序列组成的。

### 1.2.3 CAP和BASE理论

如果我们期望实现一套严格满足ACID特性的分布式事务，很可能出现的情况就是在系统的可用性和严格一致性之间出现冲突，因为当我们要求在分布式系统中具有严格一致性的时候，很可能就需要牺牲掉系统的可用性。

于是如何构建一个兼顾可用性和一致性的分布式系统成为了无数工程师探讨的难题，出现了诸如CAP和BASE这样的分布式经典理论。

**CAP定理**

CAP定理告诉我们，一个分布式系统不可能同时满足一致性(C：Consistency)，可用性(A ：Availability) 和 分区容错性(P：Partition tolerance) 这三个基本需求，最多只能满足其中的两项。

- **一致性**

  在分布式环境中，一致性是指数据在多个副本下是否能够保持一致的特性。比如将数据副本分布在不同节点上的系统来说，如果第一个节点数据进行了更新操作并且更新成功后，却没有使得第二个节点上的数据得到更新，那么当请求对第二个节点进行读操作的时候，会读取到旧数据。在分布式系统中，如果能够对一个数据项的改变执行成功后，所有的用户都可以立刻在不同节点上读取到最新的值，那么这样的系统就被认为具有强一致性(严格一致性)

- **可用性**

  可用性指的是提供的服务必须一直处于可用的状态，对于用户的每一个操作都在有限时间内返回结果

  有限的时间内指的是系统设计之处的运行指标，比如以Goole为例，搜索“分布式“这一关键字，Goole能够在0.3秒左右的时间返回大约上千万检索条目。

  返回结果也是可用性非常重要的一个指标，应该返回一个正确的响应结果，比如成功或者失败，而不是返回一些让用户困惑的结果。比如超时。

- **分区容错性**

  网络分区是指不同节点在不同的子网络，由于特殊原因造成这些子网络不连通，但是子网络内部网络又是正常的，从而导致系统被切分了。

<img src="https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210507175432951.png?versionId=CAEQHBiBgMCvxJ742hciIDMwMTIyNjQzZWEyMDQxOGNhNjYwYjdhYWYyNWNmYWRj" alt="image-20210507175432951" style="zoom:67%;" />

![image-20210507175531625](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210507175531625.png)

必须首先需要有P，因为放弃P代表了数据都放在一个节点。但是又违背了分布式这一概念，分布式的组件必然要放在不同的节点中，所以系统架构师往往需要把精力花在如何根据业务特点在C(一致性)和A(可用性)之间寻求平衡。

**BASE理论**

BASE是Basically Available(基本可用)，Soft state(软状态)和Eventually consistent(最终一致性)三个短语的简写。

BASE是对CAP中一致性和可用性权衡的结果。是基于CAP定理演化而来的。其核心思想是即时无法做到强一致性，也可以使得系统达到最终一致性。

- **基本可用**

  指的是分布式系统中出现不可预知的故障的时候，允许损失部分可用性，但是不等价于系统不可用

  比如

  - 响应时间上的损失 ： 正常情况下搜索引擎需要0.5m返回结果，出现不可预知的故障可能需要2秒才能返回
  - 功能上的损失：正常情况下，一个APP可能拥有注册，购物，登录各种功能。但是比如当双11的时候，可能使得不重要的服务先停掉，让其核心服务高效并且稳定的运行。

- **弱状态**

  弱状态也称为软状态，是指允许系统中的数据存在中间状态，即允许系统在不同的节点的数据副本之间存在延时

- **最终一致性**

  最终一致性强调的是系统中所有的数据副本，在经过一段时间后，最终能够达到一个一致的状态。最终一致性也可以认为是特殊的弱一致性。

在实际工程实践中，最终一致性存在以下五种主要变种：

- **因果一致性**

  进程A更新完某项数据后通知进程B，那么进程B之后对该数据的访问都应该可用获取到进程A的值，并且进程B要对该数据项进行更新的话，也应该基于基于进程A更新后的最新值。与此同时，与进程A无因果关系的进程C访问数据则没有这样的保证。

- **读己之所写**

  进程A更新完一个数据之后，它自己总是可用访问到最新的值。特殊的因果一致性。

- **会话一致性**

  指的是对系统数据的访问框定在了一个会话中，系统能保证有效的会话内读己之所写

- **单调读一致性**

  如果从系统中读取一个数据项后，那么系统对于该进程后续的任何数据访问都不应该返回更旧的值

- **单调写一致性** 

  保证一个系统可以被来自同一个进程的写操作顺序执行

# 2丶一致性协议

## 2.1 2PC与3PC

### 2.1.1 2PC

二阶段提交，为了使基于分布式系统架构下所有节点在进行事务处理过程中能够保持原子性和一致性而设计的一种算法。通常，也被认为是一种一致性协议，用来保证分布式系统数据的一致性。

顾名思义，二阶段提交协议就是将事务的提交过程分为了两个阶段来进行处理，其执行流程如下：

**阶段一：提交事务请求**

1.事务询问

​    协调者向所有的参与者发送事务内容，询问是否可以进行事务操作，并开始等待各参与者的响应。

2.执行事务

​    各参与者执行事务，并且将undo和redo信息记录到事务日志。

3.各参与者向协调者反馈事务循环的内容 

  如果参与者成功执行了此操作，则会返回Yes，表示事务可以执行。如果参与者没有成功的执行事务，那么会响应No，表示事务不可以执行。

**阶段二：执行事务提交**

协调者会根据参与者的反馈情况来决定是否要进行事务提交操作，正常情况下，包含以下两种可能

- 执行事务提交

  假设参与者都执行事务成功返回了Yes，那么就会进行事务提交

  1.发送提交请求

     协调者向所有参与者发送commit请求

  2.事务提交

    参与者接收到Commit请求后，会执行事务提交操作，并且在提交完成之后释放在事务期间持有的资源

  3.反馈事务提交结束

    参与者向协调者发送ACK响应。

  4.完成事务

     协调者收到ACK消息后，代表完成事务。

- 中断事务

  假设参数这有一个执行失败返回了No，那么就会执行事务回滚

  1.发送rollback请求

    协调者向所有的参与者发送rollback请求

  2.事务回滚

     协调者收到rollback请求后，读取事务日志进行回滚操作

  3.反馈事务回滚结束

     参与者向协调者发送ACK响应

  4.中断事务

  ​    协调者接收到所有参与者反馈的ACK消息后，完成事务中断。

![image-20210507195918250](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210507195918250.png)

![image-20210507195925794](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210507195925794.png)

**优缺点：**

- **优点：实现简单**
- **缺点：同步阻塞丶单点问题丶数据不一致丶太过保守**

同步阻塞：

在二阶段的提交过程中，所有参与该事务操作的逻辑都会处于阻塞状态。也就是说，各个参与者都需要等待其他参与者响应的过程，并且无法执行其他操作。

单点问题：

协调者非常关键，如果协调者出现故障，并且此时还处于阶段二，那么就会造成所有参与者都将会一直处于锁定事务资源的状态，而无法继续完成事务。

数据不一致：

如果处于二阶段，即事务提交的时候，发送Commit请求之后出现了局部网络异常，或者说未发送完所有的参与者然后出现故障，那么就会造成有的数据已经提交，有的数据未提交，出现数据不一致的情况

太过保守：

如果协调者指示参与者进行事务提交询问的时候，参与者出现故障而导致协调者无法收到来自所有的参与者响应信息的话，协调者需要根据超时机制来判断是否需要进行中断事务，比较保守。没有良好的容错机制。

### 2.1.2 3PC

是基于对2PC问题的一些改进算法

分为了3个阶段(Phase-Commit)提交

**阶段一：CanCommit**

1. 事务询问

   协调者向所有的参与者发送一个包含事物内容的canCommit询问是否可以执行事务提交操作，并开始等待各参与者的响应。

2. 各参与者向协调者反应事物询问的响应

   参与者在接受到来自协调者的canCommit请求后，正常情况下，如果其认为自身可以顺利执行事务，那么会反馈Yes响应，并且进入准备状态，否则反馈No响应

**阶段二：PreCommit**

阶段二中，协调者会根据各参与者的反馈情况来决定是否可以进行事务的PreCommit操作，正常情况下，包含两种可能

- 执行事务预提交
  1. 协调者向所有参与者节点发送PreCommit的请求，并且进入Prepared阶段
  2. 参与者收到请求，将在阶段一收到的事务内容写入到undo,redo事务日志
  3. 如果参与者成功执行了事务操作，那么就会反馈给协调者Ack响应，同时等待最终的指令：提交或者中止

-  中断事务：所有的参与者有一个反馈No响应则会进行中断事务
  1. 协调者向所有的参与者节点发送abort请求
  2. 无论是收到协调者发送的abort请求，或者是等待协调者响应超时都会中断事务（减少阶段二当中的同步阻塞）

**阶段三：doCommit**

 如果所有参与者都返回了Yes并且预提交后也收到了参与者的Ack响应，会进行真正的事务提交阶段，也分为两种情况

- 执行提交
  1. 协调者会从预提交状态转换为提交状态，并且向所有的参与者发送Commit请求。
  2. 参与者收到Commit请求后，会进行Commit提交，释放占用的事务资源，然后返回给协调者Ack响应
  3. 协调者收到所有的参与者反馈的Ack响应消息后，完成事务
- 中断事务
  1. 如果阶段二写入事务日志失败，那么就会事务的回滚，首先向所有的参与者发送abort请求。
  2. 参与者接收到abort请求后，会将undo信息来进行事务会滚，并且释放资源，或者是等待协调者发送提交或者中止请求超时，那么也会进行事务的回滚。
  3. 响应给协调者Ack信息
  4. 协调者收到所有参与者反馈的Ack消息后，中断事务

一旦进入阶段三，会出现两种故障

- 协调者出现问题
- 协调者和参与者的网络通信出现问题

减少了2PC的同步阻塞范围。但是也会出现数据不一致的问题

如果协调者向所有参与者进行Commit提交的时候，可能出现网络问题或者是提交了部分协调者，剩余的部分协调者等待超时进行了事务回滚，那么就会造成数据不一致的问题。

## 2.2 Raft与Paxos算法

### 2.2.1 Raft算法

由于Paxos算法复杂，实现困难，而分布式系统领域又需要一种高效而又易于实现的分布式一致性算法，Raft算法应运而生。

#### 2.2.2.1 Raft角色

一个Raft集群包含若干个节点，Raft把这些节点分为三种状态：Leader，Follower和Candiate

- **Leader(领导者)**：负责日志的同步管理，处理来自客户端的请求，与Follow保持headrtBeat的联系
- **Follower(追随者)**：响应Leader的日志同步请求，响应Candidate的邀票请求，以及把客户端请求到Follower到事务请求重定向(转发)给Leader。
- **Candidate(候选者)**：负责选举投票，集群刚启动时或者Leader宕机的时候，状态为Follower的节点转换为Candidate并且发起选举，当在集群中获得多半投票后，Candidate状态转换为Leander状态

#### 2.2.2.2 Raft三个子问题

由于Paxos算法的复杂性，Raft为了便于理解，将一致性问题分解成了三个相对独立的子问题

- **选举(Leader Election)** : 当Leader宕机或者集群刚启动的时候，新的Leader需要被选举出来
- **日志复制(Log Replioation)** : Leader接受来自客户端的请求并且以日志条目的形式复制到集群的其他节点，并且强制要求其他节点和自己保持一致。
- **安全(Safety)** ：如果有任何的服务器节点已经应用了一个确定的日志条目到它的状态急中，那么其他服务器节点不能在同一个日志索引上应用不同的指令。

     #### 2.2.2.3 Raft算法之Leader Election原理

当集群刚启动时候或者Leader宕机的时候，所有的Follower没有收到来自Leader的心跳包，会认为此时集群中没有Leader，所以需要进行选举。每个Follower都会有一个随机的超时时间，100ms～500ms不等，如果到达指定的超时时间，那么会进行增加本地Term(任期)一票，然后切换为Candidate状态进行Leader Election。步骤如下：

- 增加节点本地的current term，切换到candidate状态
- 投自己一票
- 并行给其他节点发送RequestVote RPCs

等待其他节点的回复在这个过程中，根据来自其他节点的消息，可能出现三种结果

1. 收到majority的投票，包含自己的一票，则赢得选举，成为Leader
2. 被告知已经有Leader，则切换为Follower，进行与Leader的日志信息同步
3. 一段时间内没有收到majority的投票，保持Candidate状态，重新发出选举 

 第一种情况，赢得了选举之后，新的leader会立刻给所有节点发消息，广而告之，避免其余节点触发新的选举。在这里，先回到投票者的视角，投票者如何决定是否给一个选举请求投票呢，有以下约束：

- 在任一任期内，单个节点最多只能投一票
- 候选人知道的信息不能比自己的少（这一部分，后面介绍log replication和safety的时候会详细介绍）
- first-come-first-served 先来先得

   第二种情况，比如有三个节点A B C。A B同时发起选举，而A的选举消息先到达C，C给A投了一票，当B的消息到达C时，已经不能满足上面提到的第一个约束，即C不会给B投票，而A和B显然都不会给对方投票。A胜出之后，会给B,C发心跳消息，节点B发现节点A的term不低于自己的term，知道有已经有Leader了，于是转换成follower。

   第三种情况，没有任何节点获得majority投票，比如下图这种情况：

<img src="https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210513141430268.png" alt="image-20210513141430268" style="zoom:50%;" />

 总共有四个节点，Node C、Node D同时成为了candidate，进入了term 4，但Node A投了NodeD一票，NodeB投了Node C一票，这就出现了平票 split vote的情况。这个时候大家都在等啊等，直到超时后重新发起选举。如果出现平票的情况，那么就延长了系统不可用的时间（没有leader是不能处理客户端写请求的），因此raft引入了randomized election timeouts(随机选举超时时间)来尽量避免平票情况。同时，leader-based 共识算法中，节点的数目都是奇数个，尽量保证majority的出现。

#### 2.2.2.4 Log replication(日志复制)

当有了leader，系统应该进入对外工作期了。客户端的一切请求来发送到Leader，Leader来调度这些并发请求的顺序，并且保证Leader与Follower状态的一致性，raft的做法是，将这些请求以及执行顺序告知followers。Leader和followers以相同的执行顺序来执行这些请求，保证状态一致。

**Relicated state machines**

共识算法的实现一般是基于复制状态机，何为复制状态机：

简单来说：**相同的初始状态 + 相同的输入 = 相同的结束状态**、

不同的节点以相同且确定的函数，来处理一个不确定的值，并且保证所有节点的相同的执行顺序，使用replcated log是一个非常不错的主意。

在raft中，leader将客户端请求封装到一个log entry，将这些log entry复制到所有的follower节点，然后大家按相同的顺序应用log entry中的command，则状态肯定是一致的。

请求完整流程

当系统收到一个来自客户端的写请求，到返回给客户端，整个过程从leader到视角来看会经历如下步骤：

- leader append log entry(将log entry写入事务日志)
- Leader issue AppendEntries RPC in parallel (leader并行到发出appendEnties日志)
- leader wait for majority response
- leader apply entry to state machine （leader将该entry应用到状态机，真正的改变状态）
- leader reply to client （回复客户端）
- leader notify follower apply log （让所有的follower也将entry应用到状态机，简单理解就是commit日志）

logs由顺序编号的log entry组成，每个log entry除了command，还具有log entry时的leader term，raft算法为了保证高可用，并不是强一致性，而是最终一致性

为什么 Leader 向 Follower 发送的 Entry 是 AppendEntries 呢？

因为 Leader 与 Follower 的心跳是周期性的，而一个周期间 Leader 可能接收到多条客户端的请求，因此，随心跳向 Followers 发送的大概率是多个 Entry，即 AppendEntries。当然，在本例中，我们假设只有一条请求，自然也就是一个Entry了。

 在上面的流程中，leader只需要日志被复制到大多数节点即可向客户端返回，一旦向客户端返回成功消息，那么系统就必须保证log（其实是log所包含的command）在任何异常的情况下都不会发生回滚。这里有两个词：commit（committed），apply(applied)，前者是指日志被复制到了大多数节点后日志的状态；而后者则是节点将日志应用到状态机，真正影响到节点状态。

#### 2.2.2.5 Safety

在这一部分，主要讨论raft算法在各种各样的异常情况下是如何工作的。

衡量一个分布式算法，有许多属性，如：

- safety ：nothing bad happens (没有坏事发生)
- liveness : something good eventually happens （好事终究会发生）

在任何系统模型下，都需要满足safety属性，最终一致性则保证了liveness。

<img src="https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210513144217990.png" alt="image-20210513144217990" style="zoom:33%;" />

**Election Safety**：

选举安全性，一个任期内最多一个leader被选举出，系统中有多余的一个leader被称之为脑裂，非常严重的问题，会导致数据丢失。在raft中，两点保证了这个属性：

- 一个节点某一任期只能投一票
- 只有获得majority投票的节点才会成为leader

**log maching** 

如果两个节点上的某个log entry的log index相同并且term相同，那么在该index之前的所有log entry应该都是相同的。如何做到的？依赖于以下两点

- If two entries in different logs have the same index and term, then they store the same command.
- If two entries in different logs have the same index and term, then the logs are identical in all preceding entries

![image-20210513144610165](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210513144610165.png)

 **注意**：上图的a-f不是6个follower，而是某个follower可能存在的六个状态

  leader、follower都可能crash，那么follower维护的日志与leader相比可能出现以下情况

- 比leader日志少，如上图中的ab
- 比leader日志多，如上图中的cd
- 某些位置比leader多，某些日志比leader少，如ef（多少是针对某一任期而言）

  当出现了leader与follower不一致的情况，leader强制follower复制自己的log

leader会维护一个nextIndex数组，记录了Leader可以发送每一个follower的log index，初始化为leader最后一个log index加1，前面也提到，leader选举成功之后立刻会发送给所有followers发送AppendEntries RPC(不包含任何log entry，也就是发送log index)，那么流程总结：

- leader初始化nextIndex[x]为leader最后一个log index + 1
- AppendEntries里preLogTerm prevLogIndex来自logs[nextIndex[x]-1]
- follower根据AppendEntries的preLogIndex查看是否与自己的logs里面的logIndex一致，如果一致接受AppendEntries，否则拒绝
- 如果follower拒绝，那么leader则会 nextindex[x] -= 1;跳转到第二行继续执行，直到找到相同的logindex并且数据一致
- 同步nextIndex[x]后的所有log entries

**State Machine Safety**

如果节点将某一位置的log entry应用到了状态机，那么其他节点在同一位置不能应用不同的日志，简单来说就是所有节点在同一位置上应该应用相同的日志。

# 3丶ZooKeeper

## 3.1. ZooKeeper是什么？

是一个开放源代码的分布式协调服务

ZooKeeper是一个典型的分布式一致性解决方案，分布式应用可以基于它实现诸如数据发布、负载均衡、分布式协调/通知、集群管理、Master选举、分布式锁和分布式队列等功能。ZooKeeper可以保证如下分布式一致性特性。

- 顺序一致性

  同一个客户端发起的事务请求，最终会严格地按照其发起顺序被应用到ZooKeeper中去。

- 原子性

  所有事物请求的处理结果在整个集群中所有机器上的应用情况是一致的。要么都没有应用，要么整个集群的所有机器都应用来该事物

- 实时性

  通常人们看到实时性的第一反应是，一旦一个事物被成功应用，那么客户端立刻能从服务器上读取到这个事物变更后的最新数据状态。这里需要注意的是，ZooKeeper仅仅保证在一定的时间段内，客户端最终一定能从服务器上访问到最新的数据状态

- 可靠性

  一旦服务器成功应用来某一个事务，并完成对客户端的响应，那么该事物所引起的服务端状态变更将会被一直保留下来。

- 单一视图

  无论客户端连接都是哪个Zookeeper服务器，其看到的服务器数据模型都是一致的

## 3.2 ZooKeeper的设计目标

ZooKeeper致力于提供一个高性能、高可用、且具有严格顺序访问控制能力(主要是写操作的严格顺序性)的分布式协调服务，高性能使得ZooKeeper能够应用于那些对系统吞吐有明确要求的大型分布式系统中。

- 目标一：简单的数据模型

  使得分布式程序能够通过一个共享的、树形结果的名字空间来进行相互协调。是ZooKeeper服务器内存中的一个数据模型，其由一系列被称为ZNode的数据节点组成，总的来说，其数据模型类似于一个文件系统，不过是将数据存储到内存中，和访问Linu系统一样

- 目标二：可以构建集群

  一个ZooKeeper集群通常由一组机器组成，只要集群中存在超过一半到机器能够正常工作，那么整个集群就能正常对外服务。

  Zookeeper的客户端程序会选择和集群中任意一台机器来共同创建一个TCP连接，而一但客户端和某台ZoKeeper服务器之间的连接断开后，客户端会自动连接到集群中的其他机器。

- 目标三：顺序访问

  对于来自客户端的更新请求，ZooKeeper都会分配一个全局唯一的递增编号，这个编号反映了所有事物操作的先后顺序

- 目标四：高性能

  将全量数据存储在内存中，并且直接服务于客户端的所有非事物请求(读请求可以和集群中任意一台进行获取数据，而写请求则必须转发到集群中的Leader)

## 3.3 ZooKeeper的基本概念

**集群角色**

通常在分布式系统中，最典型的集群模式就是 Master/Slave。

而ZooKeeper中，没有采用主从集群，而是引入了Leader、Follower和Observer三种角色。Leader服务器为客户端提供读和写服务，Follower和Observer提供读服务，唯一的区别是Observer不参与Leader选举，也不参与写操作的“过半成功”策略。

**会话**

Session是指客户端会话，指的是服务器和客户端有一个TCP长连接，客户端通过心态来与服务器保持有效的会话，并且也能向服务器发送请求和收到来自服务器的Watch事件通知。Session的sessionTimeOut值用来设置一个客户端会话的超时时间，如果客户端在指定的时间内连接上服务器，那么之前创建的会话仍然有效

**数据节点**

谈到分布式的时候，通过说的节点是指组成集群的每一台机器，然而，在ZooKeeper中，节点分为两类，第一类同样是指集群的机器，第二类是指数据单元。数据模型是一棵树，由斜杠/进行分割的路径，就是一个Znode，例如/foo/path1。

ZNode可以分为持久节点和临时节点两类。持久代表一直保存，除非主动移除，而临时节点，它的生命周期和客户端会话绑定，一旦会话失效，那么临时节点都会被移除。

**版本**

对于每个ZNode，ZooKeeper都会为其维护一个叫做State的数据结构，State中记录了这个ZNode的三个数据版本，分别是version(当前ZNode的版本),cversion(当前ZNode子节点的版本)和aversion(当前ZNode的ACL版本)。可以用版本来实现分布式锁，CAS操作。

**Watch**

客户端可以向服务器注册一个对某个节点关心的事件，当某个关心的事件发生，那么会告诉客户端

**ACL**

类似于UNIX文件系统的权限控制。

- CREATE：创建子节点的权限
- READ：获取节点数据和子节点列表的权限
- WRITE：更新节点数据的权限
- DELETE：删除子节点的权限
- ADMIN：设置节点ACL的权限

## 3.4 为什么选择ZooKeeper？

开源的，高性能的、解决分布式一致性问题、工业级产品

## 3.5 ZAB协议

ZAB协议并不像Paxos算法那样，是一种通用的分布式一致性算法，而是专门为ZooKeeper设计的崩溃可恢复的原子消息广播算法。ZooKeeper中主要依赖ZAB协议来实现分布式数据一致性，基于该协议，ZooKeeper实现了集群中各副本之间数据的一执行。ZAB协议的这个主备模型架构保证了同一时刻集群中只能有一个主进程来广播服务器的状态变更。

![image-20210514171624655](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210514171624655.png)

### 3.5.1 协议介绍

ZAB协议包括两种基本都模式。分别是崩溃恢复模式和消息广播。

当整个服务框架在启动的时候，或者当Leader宕机的时候，ZAB协议就会进入恢复模式并且选举出新的Leader，当选举了新的Leader服务器，并且集群中已经有过半的机器和Leader服务器完成了数据同步以后，那么就会退出崩溃恢复模式。然后会进入消息广播模式，当一台同样遵守ZAB协议的服务器启动后加入集群发现已经存在一个Leader正在负责消息广播，那么新加入的服务器会自动进入崩溃恢复模式进行数据同步。

重点讲解一下ZAB协议的消息广播和崩溃恢复过程

**消息广播**

ZAB协议的消息广播过程使用的是一个原子广播协议，类似于一个二进制提交过程。针对客户端的事物请求，Leader服务器会为其生成对应的事物提案Proposal,并发送都其余所有的机器。与二进制提交不同的是，二进制提交需要全部返回成功或者失败，而ZAB的消息广播只需要获得过半机器的同意(包括自己)即可，这种模式是无法解决Leader崩溃数据不一致的问题，因此ZAB协议添加了另外一个模式，崩溃恢复模式。

Leader服务器会为每个事务请求生成一个对应的Proposal来进行广播，并且在广播事务Proposal之前，会为其生成一个全局单调递增的唯一ID，我们称之为事务ID(即ZXID)。具体的，在消息广播过程中，Leader会为每一个Follower服务器各自分配一个单独的队列，然后将需要广播的事务Proposal依次放入这些队列中去。并且根据FIFO策略进行消息发送，每一个Follwoer服务器在接受到这个事务Proposal之后，都会将其以事务日志的形式写入到本地磁盘，并且返回给Leader一个ACK响应，如果Leader收到过半Follower的ACK后，就会广播一个commit消息给所有的Follower以通知事务提交，同时也会完成自身的事务提交。

**崩溃恢复**

ZAB协议需要确保那些在**Leader服务器上提交的事务被所有服务器都提交**。比如说Leader服务器提交了一个事务，但是他还未来得及发送commit消息给所有的Followers就宕机了

**数据同步**

ZAB协议是如何处理需要被丢弃的事务Proposal，假如Leader提出了一个Proposal，然后宕机了，当恢复的话是否要进行Proposal提交呢？答案是否定的，需要越过。

如何越过呢？ZAB协议中的事务编号ZXID是一个64位的数字，高32位代表epoch数值，每当选举出来一个新的Leader就会将epoch值加1，相当于每一个Leader都拥有不同的epoch值，并且是单调递增的。低32位是一个计数器，针对客户端的每一个请求都会加1操作。

当该Leader恢复的时候，集群中肯定有更高的epoch值，那么该Leader就无法进行Proposal提交了。

### 3.5.2 深入ZAB协议

![image-20210515105312226](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210515105312226.png)

CEPOCH ：Follower进程向准Leader发送自己处理过最后都一个事务Proposal的epoch值

NEWEPOCH：Leader进程根据接受各进程的epoch值，选出其中最大的一个epoch值，然后对其进行加1操作，同时将newEpoch值发送给所有的Followers

ACK-E：当Followers收到Leader带来的newEpoch值，判断是否大于当前自己的epoch值，如果大于，则应用newEpoch值，然后响应给Leader

NEWLEADER：当Leader收到来自过半的Follower响应会将自己转换为真正的Leader状态，并且将Leader发送给Followers

ACK—LD：Followers进程反馈Leader进程发来的NEWLeader消息

COMMIT—LD：Leader会进行与所有Follower进行数据的同步，要求所有的Follower与自己的数据达到一致性，并且让其commit，commit本质上其实就是一个位置标识，标记了该位置之前的所有log已经被提交了。

PROPOSE：Leader生成一个针对客户端事务请求的Proposal

ACK：Follower进程反馈Leader进程发来的Proposal消息

COMMIT：Leader发送Commit消息，要求进程提交事务Proposal。

在ZAB协议中，每一个进程可能处于三种不同状态之一。

- LOOKING：Leader选举阶段，类似于Raft算法里面的Candidate。
- FOLLOWING：Follower状态+Follower服务器与Leader进行数据同步的状态
- LEADING：Leader服务器作为主进程领导状态

与Raft算法不同的区别其实就是，许多命令有区别，比如Candidate在ZAB里面称为LooKing，term任期号在ZAB里面被称为epoch，是通过一个64位的zxid 事务id的高32位来进行控制的，并且低32位是针对客户端事务请求的计数器。其他的都大同小异。

# 4、使用ZooKeeper

## 4.1 部署与运行

有两种运行模式，集群和单机

http://mirrors.hust.edu.cn/apache/zookeeper/stable/ 下载zk 修改配置文件名为zoo.cfg，指定dir存放数据位置

zoo.cfg参数详详细解释

tickTime：zookeeper的基本时间单位，毫秒值，默认值2000，那么就是2s

initLimit：初始化连接的时候，follower和leader之间的最长心跳时间。该参数默认为10，那么说明时间限制为tickTime * 10 = 20000，那么就是20s，说明如果20s内leader还未和所有的follower初始化取得联系，那么就会报错

syncLimit：表示Leader与所有follower之间发送心跳，请求和应答的最长时间长度。ZAB协议中，是所有的follower向leader去发送心跳包，如果在syncLimit * tickTime时间内没有收到来自Leader的响应，那么则认为当前集群已经无Leader，需要进行重新选举Leader。默认是5，那么意味着5*2000，那么就是10秒。

dirDIr: 数据存放的目录

clientPort ：默认客户端连接zk服务器的端口，默认是2181

server.X=A.B.C 其中X是一个数字，表示这是第几号Server，它的值和myid文件的值对应，A是该server所在的IP地址，B是配置该server和集群中的Leader交换消息所使用的端口(发送心跳包)，C配置选举leader时所使用的端口，

### 4.1.1 单机模式

单机模式下，直接https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/zkServer.sh start

就可以了，不需要配置，只需要指定dataDir就可以直接启动了

### 4.1.2 集群模式

集群模式，伪集群cp三个一样的zk目录，进行端口号的修改和dataDir的修改。

并且在dir目录下创建一个myid的文件，写自己的服务id，用于zk服务器名字的确认

<img src="https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210515142917973.png" alt="image-20210515142917973" style="zoom:50%;" />

再加上这个

## 4.2 客户端脚本

### 4.2.1 创建

```bash
create [-s][-e] path data acl
# -s 代表 创建持久节点，-e代表创建临时节点 acl用于权限控制
```

### 4.2.2 读取

```bash
ls path [wathch] # 查看指定节点下的子节点
get path [wathch]# 查看指定节点下的data值
```

### 4.2.3 更新

```bash
set path data [version] #version用于对指定的版本进行修改，可以实现cas功能
```

### 4.2.4 删除

```bash
delete path [version]
```

## 4.3 JAVA客户端API使用

### 4.3.1 创建会话

![image-20210515145039466](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210515145039466.png)

coeectString指的是Zookeeper服务器列表，host:port字符串组成，如果是集群，则中间以逗号隔开。也可以直接指定 host:port/zk-book,这样都会基于这个根目录来进行操作。

sessionTimeout指的是客户端与服务器保持会话的超时时间，zk客户端与服务器通过心跳来维持会话的有效性，一旦在sessionTimeout之间没有进行有效的会话检测，那么该会话就会失效。

Watcher指的是zk允许客户端传入一个watch接口的实现类对象，用来指定关心的节点或者事件触发，然后回调。

sessionPasswd和sessionID传入可以复用之前的会话。

canBeReadOnly指的是如果一个机器和zk集群中过半的机器失去了联系，那么该机器不再提供任何读写请求，但是在某些使用场景下，zk服务器发生此类故障的时候，我们还是希望可以提供读服务。所以该参数就是指定这个的。

```java
package org.foo.demo;

import org.apache.zookeeper.WatchedEvent;
import org.apache.zookeeper.Watcher;
import org.apache.zookeeper.ZooKeeper;
import org.apache.zookeeper.data.Stat;

import java.io.IOException;
import java.util.concurrent.CountDownLatch;

public class TestZk {


    public static void main(String[] args) throws IOException, InterruptedException {
        CountDownLatch countDownLatch = new CountDownLatch(1);
        ZooKeeper zooKeeper = new ZooKeeper("127.0.0.1:2181,127.0.0.1:2182" +
                "127.0.0.1:2183", 3000, new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                if (event.getState() == Event.KeeperState.SyncConnected){
                    System.out.println("已经成功连接上");
                    countDownLatch.countDown();
                }
            }
        });
        countDownLatch.await();

        System.out.println("我他妈要热四了");
        Thread.sleep(Integer.MAX_VALUE);
    }
}

```

### 4.3.2 创建节点

```java
  public void create(final String path, byte data[], List<ACL> acl,CreateMode createMode,  StringCallback cb, Object ctx)  
```

参数一：path 需要创建指定的指定的节点

参数二：data[] 需要创建指定的节点对应的值

参数三：ACL权限相关，后面说

参数四：创建数据节点的模型，创建大致分为临时节点和持久节点

参数五：也可以同步创建，该构造器是异步的创建节点API

- rc 

  0：接口调用成功

  -4: 客户端与服务端连接已断开

  -110：指定节点已存在

  -112:   会话已经过期

参数六：ctx是上下文，可以传入上下文再回调的时候获取进行操作

```java
package org.foo.demo;

import org.apache.zookeeper.*;
import org.apache.zookeeper.data.Stat;

import java.io.IOException;
import java.util.concurrent.CountDownLatch;

public class TestZk {


    public static void main(String[] args) throws IOException, InterruptedException, KeeperException {
        CountDownLatch countDownLatch = new CountDownLatch(1);
        ZooKeeper zooKeeper = new ZooKeeper("127.0.0.1:2181,127.0.0.1:2182" +
                "127.0.0.1:2183", 3000, new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                if (event.getState() == Event.KeeperState.SyncConnected){
                    System.out.println("已经成功连接上");
                    countDownLatch.countDown();
                }
            }
        });
        countDownLatch.await();
        String path = "/zk-cluster";
        System.out.println("我他妈要热四了");
        zooKeeper.create(path, "苏荣是笨蛋".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE,CreateMode.EPHEMERAL,(rc, p, ctx, name) -> {
            System.out.println(rc); // 0代表创建成功
            System.out.println(p);
            System.out.println(ctx);
            System.out.println(name);
        },"苏荣是憨憨");
        Stat stat = new Stat();
        byte[] data = zooKeeper.getData(path, new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                if (event.getType() == Event.EventType.NodeDataChanged) {
                    System.out.println(event.getPath());
                    System.out.println();
                }
            }
        }, stat);
        System.out.println(new String(data));
        System.out.println(stat.getAversion());

        zooKeeper.setData(path, "苏荣是笨蛋".getBytes(), -1);
        
        Thread.sleep(Integer.MAX_VALUE);
    }
}

```

### 4.3.3 读取数据

读取数据，包括子节点列表的获取和节点数据的获取，zk提供了不同的api。

首先看读取子节点列表，当前Watcher注册代表子节点列表发生变更会通知客户端

```java
public void getChildren(final String path, Watcher watcher,
        Children2Callback cb, Object ctx)
```

参数一：获取指定节点的子节点列表

参数二：指定感兴趣的事件，用于回调

参数三：cb,异步回调方法

参数四：ctx上下文

```java
   @InterfaceAudience.Public
    interface Children2Callback extends AsyncCallback {
      the node on given path.
       /**
       rc 
      0：接口调用成功
     -4: 客户端与服务端连接已断开
     -110：指定节点已存在
     -112:   会话已经过期
     path:指定的节点path
     ctx：上下文
     childern：子节点列表
     stat：新节点的状态
     */
        public void processResult(int rc, String path, Object ctx,
                List<String> children, Stat stat);
    }
```

```java
package org.foo.demo;

import org.apache.zookeeper.*;
import org.apache.zookeeper.data.Stat;

import java.io.IOException;
import java.util.List;
import java.util.concurrent.CountDownLatch;

public class TestZk {


    public static void main(String[] args) throws IOException, InterruptedException, KeeperException {
        CountDownLatch countDownLatch = new CountDownLatch(1);
        ZooKeeper zooKeeper = new ZooKeeper("127.0.0.1:2181,127.0.0.1:2182" +
                "127.0.0.1:2183", 3000, new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                if (event.getState() == Event.KeeperState.SyncConnected){
                    System.out.println("已经成功连接上");
                    countDownLatch.countDown();
                }
            }
        });
        countDownLatch.await();
        String path = "/zk-cluster1";
        System.out.println("我他妈要热四了");
        zooKeeper.create(path, "苏荣是笨蛋".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE,CreateMode.PERSISTENT,(rc, p, ctx, name) -> {
            System.out.println(rc); // 0代表创建成功
            System.out.println(p);
            System.out.println(ctx);
            System.out.println(name);
        },"苏荣是憨憨");
        zooKeeper.create(path+"/c1", "苏荣".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE,CreateMode.EPHEMERAL,(rc, p, ctx, name) -> {
            System.out.println(rc); // 0代表创建成功
            System.out.println(p);
            System.out.println(ctx);
            System.out.println(name);
        },"苏荣是瓜皮");
        Stat stat = new Stat();
        byte[] data = zooKeeper.getData(path, new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                if (event.getType() == Event.EventType.NodeDataChanged) {
                    System.out.println(event.getPath());
                    System.out.println();
                }
            }
        }, stat);
        System.out.println(new String(data));
        System.out.println(stat.getAversion());

        zooKeeper.setData(path, "苏荣是笨蛋".getBytes(), -1);

        zooKeeper.getChildren(path, false, new AsyncCallback.Children2Callback() {
            @Override
            public void processResult(int rc, String path, Object ctx, List<String> children, Stat stat) {
                System.out.println(rc);
                System.out.println(path);
                System.out.println(ctx);
                System.out.println(children);
                System.out.println(stat);
                System.out.println(stat.getAversion());
            }
        },"憨憨");
        Thread.sleep(Integer.MAX_VALUE);
    }
}

```

getData

用来获取内容

```java
   byte[] data = zooKeeper.getData(path, new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                if (event.getType() == Event.EventType.NodeDataChanged) {
                    System.out.println(event.getPath());
                    System.out.println();
                }
            }
        }, stat);
```

### 4.3.4 更新数据

```java
 public Stat setData(final String path, byte data[], int version)
```

参数一：path设置数据的节点

参数二：新要修改的数据

参数三：版本号，-1默认更新最新的版本号，可以用它来实现乐观锁，CAS等操作

### 4.3.5 检查节点是否存在

```java
public void exists(String path,boolean wathch,StatCallback cb,Object ctx);
public Stat exists(String path,Wathch watch)
```

参数path：指定数据节点的节点路径，检查该节点是否存在

参数Wathch：注册的Watcher，用于监听以下三类事件

- 节点被创建
- 节点被删除
- 节点被更新

参数cb：注册一个异步回调函数

参数ctx：用于传递上下文信息的对象。

参数boolean watch : 指定是否服用ZK默认的Wather

### 4.3.6 权限控制

为什么需要权限控制？

我们通常会搭建一个zk集群来对多个应用进行提供服务，那么需要对各个服务数据进行隔离，防止被其他节点的进程干扰或者人为修改操作，所以我们需要对zk上的数据访问进行权限控制。

zk提供了多种权限控制模式，分别是word、digest、auth、ip和super。

我们主要讲解digest模式下如何进行权限控制。

World: 只有一个用户：anyone

Id ：使用ip地址认证

auth：使用已添加认证的用户认证

digest：使用用户名：密码方式认证

```java
addAuthInfo(String scheme,byte[] auth)
```

参数一：权限控制访问模式

参数二：具体的权限信息。



```java
package org.foo.demo;

import org.apache.zookeeper.*;

import java.util.concurrent.CountDownLatch;

public class TestAuthZK {
    public static void main(String[] args) throws Exception{
        CountDownLatch countDownLatch = new CountDownLatch(1);
        ZooKeeper zooKeeper = new ZooKeeper("127.0.0.1:2181,127.0.0.1:2182" +
                "127.0.0.1:2183", 3000, new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                if (event.getState() == Event.KeeperState.SyncConnected){
                    System.out.println("已经成功连接上");
                    countDownLatch.countDown();
                }
            }
        });
        countDownLatch.await();
        String path = "/zk-cluster1234";
        zooKeeper.addAuthInfo("digest","foo:fengjiahao".getBytes());
        zooKeeper.create(path, "苏荣是笨蛋".getBytes(), ZooDefs.Ids.CREATOR_ALL_ACL, CreateMode.PERSISTENT,(rc, p, ctx, name) -> {
            System.out.println(rc); // 0代表创建成功
            System.out.println(p);
            System.out.println(ctx);
            System.out.println(name);
        },"苏荣是憨憨");
        CountDownLatch countDownLatch1 = new CountDownLatch(1);
        ZooKeeper zooKeeper1 = new ZooKeeper("127.0.0.1:2181,127.0.0.1:2182" +
                "127.0.0.1:2183", 3000, new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                if (event.getState() == Event.KeeperState.SyncConnected){
                    System.out.println("已经成功连接上");
                    countDownLatch1.countDown();
                }
            }
        });
        countDownLatch1.await();
        zooKeeper1.addAuthInfo("digest","foo:fengjiahao".getBytes());
        byte[] data = zooKeeper1.getData(path, false, null);
        System.out.println(new String(data));

    }
}

```

## 4,4 开源客户端

### 4.4.1 ZkClient

```bash
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
</dependency>
  <dependency>
            <groupId>com.github.sgrochupf</groupId>
            <artifactId>zkclient</artifactId>
  </dependency>
```

![image-20210515170645016](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210515170645016.png)

支持一次性创建多个节点路径，比如/book/ex/c/a

前面的路径都不存在也支持直接创建

其他的基本和原始的类似。

### 4.4.2 Curator

Curator解决了很多ZK客户端非常底层的细节开发工作，包括连接重新连接，反复注册(原始客户端注册一次Wathch就会失效，需要重复注册)和NodeExistsException

首先需要引入依赖

```bash
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.4.13</version>
</dependency>
```

#### 4.4.2.1 创建会话

```java
package org.foo.demo;


import org.apache.curator.RetryPolicy;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.retry.ExponentialBackoffRetry;

public class TestZkCurator {
    public static void main(String[] args) throws Exception {
        //baseSleepTimeMS
        //maxRetries 最大重试次数
        ExponentialBackoffRetry exponentialBackoffRetry = new ExponentialBackoffRetry(1000, 3);
        CuratorFramework framework = CuratorFrameworkFactory.builder()
                .connectString("127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183")
                .canBeReadOnly(true)
                .connectionTimeoutMs(60000) //连接创建超时时间
                .retryPolicy(exponentialBackoffRetry) //重试策略  默认有四种实现，也可以自定义实现重试策略
                .sessionTimeoutMs(15000)//会话超时时间
                .build();
        framework.start();

        System.out.println("连接成功");
        Thread.sleep(Integer.MAX_VALUE);


    }
}

```

```java
  new RetryPolicy() {
            @Override
            public boolean allowRetry(int retryCount, long elapsedTimeMs, RetrySleeper sleeper) {
                return false;
            }
        };
```

重试策略参数详解：

retryCount ：重试次数

elapsedTimeMs：从第一次重试开始已经花费的时间

Sleeper ：用于sleep指定时间。

用于重新连接

```java
    public ExponentialBackoffRetry(int baseSleepTimeMs, int maxRetries, int maxSleepMs)
    {
        super(validateMaxRetries(maxRetries));
        this.baseSleepTimeMs = baseSleepTimeMs;
        this.maxSleepMs = maxSleepMs;
    }
```

参数一：初始Sleep时间

参数二：最大重试次数

参数三：最大sleep时间

给定一个初始Sleep时间，再这个基础上结合重试次数，通过以下公式计算出当前需要sleep时间。

当前sleep时间 = baseSleepTimeMs * Math.max(1,Random.next(1 << (maxRetries+1))

可以看出，随着重试次数的增加，计算出的sleep会越来越大，如果在maxSleep范围内，则使用当前sleep，否则使用maxSleep

#### 4.4.2.2 创建节点

```java
client.create().path(path) //默认创建持久节点，创建指定路径的节点
client.create().forpath(path,"init".getBytes()); //创建一个带指定值节点
client.create.withMode(CreateMode.EPHEMERAL).forPath(path); //创建一个临时节点
client.create.createingParentsIfNeeded().withMode(CreateMode.EPHEMERAL).forPath(path); //创建一个临时节点，并且自动递归创建父节点
```

#### 4.4.2.3 删除节点

```java
client.delete.forPath(path);
client.delete.deletingChildrenIfNeeded().forPath(path);//删除一个节点，并且递归删除其所有子节点
client.delete.withVersion(version).forPath(path);
client.delete.guranteed().forPath(path); //强制保证删除一个节点
```

#### 4.4.2.4 读取数据

``` java
client.getData().forPath(String path) //读取指定路径的数据内容,返回值是byte[]
client.getData().storingStatIn(Stat stat).forPath(path);
//Curator通过传入一个旧的Stat变量的方式来存储服务器返回的最新节点的状态信息。
  
```

#### 4.4.2.5 更新数据

```java
client.setData().forPath(path); //更新一个节点的数据内容
client.setData.withVersion(version).forPath(path) //向指定版本的数据设置
```

#### 4.4.2.6 异步接口

如何通过Curator实现异步操作？

BackgroundCallback

```java

/**
 * Functor for an async background operation
 */
public interface BackgroundCallback
{
    /**
     参数一：zk客户端
     参数二：服务端事件
    */
    public void processResult(CuratorFramework client, CuratorEvent event) throws Exception;
}
```

```java
public enum CuratorEventType
{
    /**
     * Corresponds to {@link CuratorFramework#create()}
     */
    CREATE, //创建节点

    /**
     * Corresponds to {@link CuratorFramework#delete()}
     */
    DELETE,//删除节点

    /**
     * Corresponds to {@link CuratorFramework#checkExists()}
     */
    EXISTS,//是否存在节点

    /**
     * Corresponds to {@link CuratorFramework#getData()}
     */
    GET_DATA,//获取数据事件

    /**
     * Corresponds to {@link CuratorFramework#setData()}
     */
    SET_DATA,//设置数据事件

    /**
     * Corresponds to {@link CuratorFramework#getChildren()}
     */
    CHILDREN,//获取子列表

    /**
     * Corresponds to {@link CuratorFramework#sync(String, Object)}
     */
    SYNC, //同步

    /**
     * Corresponds to {@link CuratorFramework#getACL()}
     */
    GET_ACL,//获取权限

    /**
     * Corresponds to {@link CuratorFramework#setACL()}
     */
    SET_ACL,//设置权限事件

    /**
     * Corresponds to {@link CuratorFramework#transaction()}
     */
    TRANSACTION,//

    /**
     * Corresponds to {@link CuratorFramework#getConfig()}
     */
    GET_CONFIG,

    /**
     * Corresponds to {@link CuratorFramework#reconfig()}
     */
    RECONFIG,//

    /**
     * Corresponds to {@link Watchable#usingWatcher(Watcher)} or {@link Watchable#watched()}
     */
    WATCHED,//被一个事件监听

    /**
     * Corresponds to {@link CuratorFramework#watches()} ()}
     */
    REMOVE_WATCHES //移除监听事件

    /**
     * Event sent when client is being closed
     */
    CLOSING //关闭事件
}
```

```java
public T inBackground(BackgroundCallback callback,Object contenxt,Executor executor);
```

executor参数，在ZK中，所有的异步通知事件处理都是通过一个EventThread线程处理的，由于是单线程所以是串行处理，但是如果碰上复杂的业务场景，那么执行速度可能会变慢，可以指定一个线程池

#### 4.4.2.3 事件监听

```java
//参数一：客户端实例
//参数二：路径
//参数三：是否开启数据压缩
public NodeCache(CuratorFramework client, String path, boolean dataIsCompressed)
    {
        this.client = client.newWatcherRemoveCuratorFramework();
        this.path = PathUtils.validatePath(path);
        this.dataIsCompressed = dataIsCompressed;
    }
```

原生Wathch需要反复注册，Curator引入了Cache来实现对Zk服务端事件的监听。Cache分为两种类型，节点监听和子节点监听。

NodeCache

```java
package org.foo.demo;

import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.cache.NodeCache;
import org.apache.curator.framework.recipes.cache.NodeCacheListener;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.apache.zookeeper.CreateMode;

public class TestNodeCache {
    static String path = "/zk-book/nodeCache";
    public static void main(String[] args) throws Exception {
        ExponentialBackoffRetry exponentialBackoffRetry = new ExponentialBackoffRetry(1000, 3);
        CuratorFramework client = CuratorFrameworkFactory.builder()
                .connectString("127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183")
                .canBeReadOnly(true)
                .connectionTimeoutMs(60000) //连接创建超时时间
                .retryPolicy(exponentialBackoffRetry) //重试策略  默认有四种实现，也可以自定义实现重试策略
                .sessionTimeoutMs(15000)//会话超时时间
                .build();
        client.start();
        System.out.println("连接成功");
        client.create().creatingParentsIfNeeded().withMode(CreateMode.EPHEMERAL).forPath(path,"init".getBytes());
        final NodeCache nodeCache = new NodeCache(client,path,false);
        nodeCache.start(true);
        nodeCache.getListenable().addListener(new NodeCacheListener() {
            @Override
            public void nodeChanged() throws Exception {
                System.out.println("数据被改变了");
                System.out.println(new String(nodeCache.getCurrentData().getData()));
            }
        });
        client.setData().forPath(path,"哎 我郁闷4⃣死了".getBytes());

        Thread.sleep(1000);
        client.delete().deletingChildrenIfNeeded().forPath(path);
        Thread.sleep(Integer.MAX_VALUE);




    }
}

```

PathChildrenCache

```java
package org.foo.demo;

import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.cache.PathChildrenCache;
import org.apache.curator.framework.recipes.cache.PathChildrenCacheEvent;
import org.apache.curator.framework.recipes.cache.PathChildrenCacheListener;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.apache.zookeeper.CreateMode;

public class TestPathChildrenCache {
    static String path = "/zk-book1/nodeCache";

    public static void main(String[] args) throws Exception {
        ExponentialBackoffRetry exponentialBackoffRetry = new ExponentialBackoffRetry(1000, 3);
        CuratorFramework client = CuratorFrameworkFactory.builder()
                .connectString("127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183")
                .canBeReadOnly(true)
                .connectionTimeoutMs(60000) //连接创建超时时间
                .retryPolicy(exponentialBackoffRetry) //重试策略  默认有四种实现，也可以自定义实现重试策略
                .sessionTimeoutMs(15000)//会话超时时间
                .build();
        client.start();
        System.out.println("连接成功");
        client.create().creatingParentsIfNeeded().withMode(CreateMode.PERSISTENT).forPath(path, "init".getBytes());
        PathChildrenCache cache = new PathChildrenCache(client, path, true);
        cache.start();
        cache.getListenable().addListener(new PathChildrenCacheListener() {
            @Override
            public void childEvent(CuratorFramework client, PathChildrenCacheEvent event) throws Exception {
                PathChildrenCacheEvent.Type childAdded = PathChildrenCacheEvent.Type.CHILD_ADDED;
                if (event.getType() == childAdded){
                    System.out.println("新加入了一个子节点");
                    System.out.println(event.getInitialData());
                    cache.getCurrentData().forEach(System.out::println);
                }
            }
        });
        client.create().creatingParentsIfNeeded().withMode(CreateMode.EPHEMERAL).forPath(path+"/ali","haha".getBytes());
        client.create().creatingParentsIfNeeded().withMode(CreateMode.EPHEMERAL).forPath(path+"/ala","haha".getBytes());
        client.create().creatingParentsIfNeeded().withMode(CreateMode.EPHEMERAL).forPath(path+"/alf","haha".getBytes());

        Thread.sleep(Integer.MAX_VALUE);
    }
}
```

#### 4.4.2.4 Master选举

在分布式系统中，经常会碰到这样的场景，对于一个复杂的任务，仅仅需要从集群中选举出一台进行处理即可，诸如此类的分布式问题，我们称之为Master选举，借助Zk，我们可以比较方便的实现Master选举的功能，其大体思路非常简单，集群中的多个机器同时去创建一个节点，那么只有一个进程会创建进程，那么就该机器来执行处理。Curator也是基于这个思路，但是它将节点创建、事件监听和自动选举过程都进行了封装，用户只需要调用简单的API即可实现Master选举。

```java
package org.foo.demo;

import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.cache.PathChildrenCache;
import org.apache.curator.framework.recipes.leader.LeaderSelector;
import org.apache.curator.framework.recipes.leader.LeaderSelectorListener;
import org.apache.curator.framework.state.ConnectionState;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.apache.zookeeper.CreateMode;

public class TestSelectorMaster {
    static String path = "/zk-master";
    public static void main(String[] args) throws Exception {
        ExponentialBackoffRetry exponentialBackoffRetry = new ExponentialBackoffRetry(1000, 5);
        CuratorFramework client = CuratorFrameworkFactory.builder()
                .connectString("127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183")
                .canBeReadOnly(true)
                .connectionTimeoutMs(60000) //连接创建超时时间
                .retryPolicy(exponentialBackoffRetry) //重试策略  默认有四种实现，也可以自定义实现重试策略
                .sessionTimeoutMs(15000)//会话超时时间
                .build();
        client.start();
        LeaderSelector leaderSelector = new LeaderSelector(client, path, new LeaderSelectorListener() {
            @Override
            public void takeLeadership(CuratorFramework client) throws Exception {
                System.out.println("成为Master角色");
                Thread.sleep(4000); //执行业务逻辑
                System.out.println("完成Master操作，释放Master权利");
            }

            @Override
            public void stateChanged(CuratorFramework client, ConnectionState newState) {
            }
        });
        leaderSelector.autoRequeue();
        leaderSelector.start();
        Thread.sleep(Integer.MAX_VALUE);


    }
}

```

#### 4.4.2.5 分布式锁

```java
package org.foo.demo;

import jdk.internal.org.objectweb.asm.tree.FieldInsnNode;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.locks.InterProcessMutex;
import org.apache.curator.retry.ExponentialBackoffRetry;

import javax.xml.crypto.Data;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.logging.SimpleFormatter;

public class TestLock { 
    static String path = "/zk-master";
    public static void main(String[] args) throws Exception {
        ExponentialBackoffRetry exponentialBackoffRetry = new ExponentialBackoffRetry(1000, 5);
        CuratorFramework client = CuratorFrameworkFactory.builder()
                .connectString("127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183")
                .canBeReadOnly(true)
                .connectionTimeoutMs(60000) //连接创建超时时间
                .retryPolicy(exponentialBackoffRetry) //重试策略  默认有四种实现，也可以自定义实现重试策略
                .sessionTimeoutMs(15000)//会话超时时间
                .build();
        client.start();
        final InterProcessMutex lock = new InterProcessMutex(client,path);
        try {
            for (int i = 0; i < 30 ; i++) {
             
                new Thread(() -> {
                    try {
                        lock.acquire();
                        
                        SimpleDateFormat simpleDateFormat = new SimpleDateFormat("HH:mm:ss|SSS");
                        String format = simpleDateFormat.format(new Date());
                        System.out.println(format);
                    } catch (Exception e) {
                        e.printStackTrace();
                    }finally {
                        try {
                            lock.release();
                        } catch (Exception e) {
                            e.printStackTrace();
                        }
                    }


                }).start();
            }
            
        }catch (Exception e){
            
        }finally {
            lock.release();
        }
        Thread.sleep(Integer.MAX_VALUE);
        
    }
        
}

```

#### 4.4.2.6 分布式计数器

分布式计数器典型场景是统计系统的在线人数，基于ZK的分布式计数器的实现思路也很简单。指定一个ZK数据节点作为计数器，多个应用实例在分布式锁的控制下，通过更新该数据节点内容来实现计数功能。**DistributedAtomicInteger**

```java
package org.foo.demo;

import jdk.internal.org.objectweb.asm.tree.FieldInsnNode;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.atomic.DistributedAtomicInteger;
import org.apache.curator.framework.recipes.locks.InterProcessMutex;
import org.apache.curator.retry.ExponentialBackoffRetry;

import javax.xml.crypto.Data;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.logging.SimpleFormatter;

public class TestLock {
    static String path = "/zk-master";
    public static void main(String[] args) throws Exception {
        ExponentialBackoffRetry exponentialBackoffRetry = new ExponentialBackoffRetry(1000, 5);
        CuratorFramework client = CuratorFrameworkFactory.builder()
                .connectString("127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183")
                .canBeReadOnly(true)
                .connectionTimeoutMs(60000) //连接创建超时时间
                .retryPolicy(exponentialBackoffRetry) //重试策略  默认有四种实现，也可以自定义实现重试策略
                .sessionTimeoutMs(15000)//会话超时时间
                .build();
        client.start();
        final InterProcessMutex lock = new InterProcessMutex(client,path);
        final DistributedAtomicInteger atomicInteger = new DistributedAtomicInteger(client,path+"/count",exponentialBackoffRetry);
        try {
            for (int i = 0; i < 30 ; i++) {

                new Thread(() -> {
                    try {
                        lock.acquire();
                        atomicInteger.increment();
                        SimpleDateFormat simpleDateFormat = new SimpleDateFormat("HH:mm:ss|SSS");
                        String format = simpleDateFormat.format(new Date());
                        System.out.println(format);
                    } catch (Exception e) {
                        e.printStackTrace();
                    }finally {
                        try {
                            lock.release();
                        } catch (Exception e) {
                            e.printStackTrace();
                        }
                    }


                }).start();
            }

        }catch (Exception e){

        }finally {
            lock.release();
        }
        Thread.sleep(Integer.MAX_VALUE);

    }

}

```

#### 4.4.2.7 分布式Barrier

```java
package org.foo.demo;

import jdk.internal.org.objectweb.asm.tree.FieldInsnNode;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.atomic.DistributedAtomicInteger;
import org.apache.curator.framework.recipes.barriers.DistributedBarrier;
import org.apache.curator.framework.recipes.locks.InterProcessMutex;
import org.apache.curator.retry.ExponentialBackoffRetry;

import javax.xml.crypto.Data;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.logging.SimpleFormatter;

public class TestLock {
    static String path = "/zk-master";
    static DistributedBarrier distributedBarrier;

    public static void main(String[] args) throws Exception {

            for (int i = 0; i < 30 ; i++) {
               new Thread(new Runnable() {
                   @Override
                   public void run() {
                       ExponentialBackoffRetry exponentialBackoffRetry = new ExponentialBackoffRetry(1000, 5);
                       CuratorFramework client = CuratorFrameworkFactory.builder()
                               .connectString("127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183")
                               .canBeReadOnly(true)
                               .connectionTimeoutMs(60000) //连接创建超时时间
                               .retryPolicy(exponentialBackoffRetry) //重试策略  默认有四种实现，也可以自定义实现重试策略
                               .sessionTimeoutMs(15000)//会话超时时间
                               .build();
                       client.start();
                       distributedBarrier = new DistributedBarrier(client, path);
                       try {
                           distributedBarrier.setBarrier();
                       } catch (Exception e) {
                           e.printStackTrace();
                       }
                       try {
                           distributedBarrier.waitOnBarrier();
                       } catch (Exception e) {
                           e.printStackTrace();
                       }
                       SimpleDateFormat simpleDateFormat = new SimpleDateFormat("HH:mm:ss|SSS");
                       String format = simpleDateFormat.format(new Date());
                       System.out.println(format);
                   }
               }).start();
            }
            Thread.sleep(10000);

        distributedBarrier.removeBarrier();
        Thread.sleep(Integer.MAX_VALUE);

    }

}

```

