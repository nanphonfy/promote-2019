[TOC]

### zookeeper由来
>①各节点数据一致性；  
②怎么保证任务只在一个节点执行；  
③若其中一个节点挂了，其他节点如何发现并接替；  
④存在共享资源，互斥性、安全性。

>分布式系统的很多难题，都是由缺少协调机制造成。分布式协调做的较好的：Google Chubby、Apache Zookeeper。  
Google Chubby是一个分布式锁服务，解决分布式协作、Master选举等与分布式锁服务相关问题。  
Zookeeper也类似，在雅虎内部很多系统都需依赖一个系统来分布式协调，但谷歌的Chubby不开源，故雅虎基于Chubby的思想开发了zookeeper并捐赠给Apache。  
zookeeper架构，可用来解决task多实例执行问题，各服务先去zookeeper上注册节点，然后获得权限后再来访问task。

### zookeeper设计猜想
>主要解决分布式环境的服务协调问题，若实现一个zookeeper中间件需做啥？  
>- ①防单点故障：做集群，且集群若要满足高性能要求，还得是一个高性能、高可用集群。高性能意味该集群能够分担客户端的请求流量，高可用意味集群中某节点宕机后，不影响整个集群的数据和继续提供服务的可能性。    
结论：该中间件需考虑集群,且该集群还需分摊客户端的请求流量；  
>- ②要满足高性能，每个节点都能接到请求，且每个节点数据都必须保持一致。实现各节点数据一致性，要有一个 leader节点负责协调和数据同步操作。若集群无leader节点，每个节点都可接收所有请求，那该集群的数据同步复杂度很大。  
结论：该集群涉及数据同步，会存在leader节点；  
>- ③如何选举leader节点，leader挂了后，如何恢复？  
结论：zookeeper基于paxos理论衍生出ZAB协议；    
>- ④leader节点如何和其他节点保证数据一致性，且要求强一致。  
分布式系统中，每个节点虽都能知道自己的事务操作过程是成功/失败，却无法直接获取其他分布式节点的操作结果。故当一个事务操作涉及到跨节点时，就需用到分布式事务。  
分布式事务的数据一致性协议：2PC协议和3PC协议。

>基于如上猜想，基本知道zookeeper为何要用到zab理论做选举、为何要做集群、为何要用到分布式事务实现数据一致性。

#### 2PC提交
>Two Phase Commitment Protocol，当一个事务需跨越多个分布式节点时，为保持ACID特性，需引入一个协调者（TM）来统一调度所有分布式节点的执行逻辑，分布式节点被称为AP。TM负责调度AP的行为，最终决定AP是否提交事务；整个事务分为两个阶段提交，故称2PC。

>TM（或许是个进程）通常是由TPM（transaction processing monitor）提供。  
角色：①AP:application应用节点；②RM:resource manager资源管理器，一般是数据库文件系统之类；③TM:transaction manager事务管理器、事务协调者，负责解释AP发起的事务指令，调度协调参与者所有RM的协调，确保事务正常完成。
##### 阶段一：提交事务请求（投票）
>①事务询问：协调者向所有参与者发送事务内容，询问是否可执行事务提交，等待各参与者响应；  
②执行事务：各参与者执行事务，将Undo和Redo信息记录到事务日志，尽量把提交过程中所有消耗时间的操作和
准备都提前完成确保后面100%成功提交事务；  
③各参与者向协调者反馈事务询问的响应：若各参与者成功执行事务，则反馈给协调者yes的响应，表示事务可执行；若参与者没成功执行事务，则反馈给协调者no的响应，表示事务不可执行。该阶段类似协调者组织各参与者对一次事务操作的投票表态过程，因此2pc协议的第一个阶段称为“投票阶段”，即各参与者投票表明是否需继续执行事务提交。
##### 阶段二：执行事务提交
>该阶段，协调者根据各参与者的反馈情况决定最终的事务提交操作，包含两种可能:执行事务、中断事务。

### 集群
>zookeeper中，客户端会随机连接到zookeeper集群中的一个节点。若是读请求，直接从当前节点读取数据；若是写请求，会被转发给leader提交事务(广播)，超半数节点写入成功，则被提交（类2PC事务）所有事务请求必须由一个全局唯一的服务器协调处理，该服务器即leader，其他服务器即follower。   
leader服务器把客户端的请求转化成一个事务proposal（提议），并把该proposal分发给集群中所有follower，之后leader需等待所有follower的反馈，超半数follower正确反馈，leader会再次向所有follower发送commit消息，要求各follower节点提交前面一个proposal。

#### 集群角色
- Leader角色
>整个zookeeper集群的核心，主要工作：  
>1.事务请求的唯一调度和处理者，保证集群事务处理的顺序性；  
2.集群内部各服务器的调度者。

- Follower角色
>主要职责：  
>1.处理客户端非事务请求、转发事务请求给leader；  
2.参与事务请求Proposal的投票（需半数以上服务器通过才通知leader提交数据；  
3.参与Leader选举的投票。  

- Observer角色
>zookeeper3.3开始引入的全新角色，观察者角色。
观察zookeeper集群中最新状态变化，将这些状态变化同步到observer上。工作原理与follower角色基本一致，唯一不同：observer不参与任何形式的投票，包括事务请求proposal的投票和leader选举的投票。简单说，observer只提供非事务请求服务，通常在于不影响集群事务处理能力的前提下提升集群非事务处理的能力。

- 集群组成
>zookeeper由2n+1台server组成，每个server都知道彼此存在。对于2n+1台server，只要有n+1台（大
多数）server可用，整个系统保持可用。   集群要对外提供可用的服务，必须有过半机器正常工作且彼此正常通信，基于该特性，若向搭建一个能允许F台机器down掉的集群，则要部署2*F+1台服务器构成的zookeeper集群。故3台机器构成的zookeeper集群，能够在挂掉一台机器后依然正常工作。
>5台机器的集群，允许2台机器挂掉；6台机器的集群，只能挂掉2台机器。故5台和6台在容灾能力上并无明显优势，反而增加网络通信负担。系统启动时，集群中的server会选举出一台server为Leader，其它的就作为follower（先不考虑observer角色） 。

### ZAB协议
>Zookeeper Atomic Broadcast协议，是为ZooKeeper设计的支持崩溃恢复的原子广播协议。主要依赖ZAB协议实现分布式数据一致性，基于该协议实现主备模式的系统架构，保持集群各副本间数据一致性。  
#### zab协议介绍
>包含两种基本模式：1.崩溃恢复；2.原子广播。

>①当集群启动或leader节点网络中断、崩溃（等）时，ZAB协议会进入恢复模式并选举新leader；②当leader选举出后，且集群有过半机器和该leader完成数据同步后，ZAB协议会退出恢复模式；③当集群有过半follower和leader状态同步后，会进入消息广播模式。  
此时，leader正常工作时，加入一台新服务器，会直接进入数据恢复模式，和leader数据同步。同步完后即可正常对外提供非事务请求处理。

#### 消息广播实现原理
>分布式事务的2pc和3pc协议，消息广播过程是简化版的二阶段提交：
>- 1.leader收到请求后，将消息赋予全局唯一64位自增id——zxid，通过其大小比较即可实现因果有序；  
>- 2.leader为每个follower准备一个FIFO队列（TCP协议实现，全局有序）将带有zxid的消息作为一个提案（proposal）分发给所有的follower；  
>- 3.当follower收到proposal，先把proposal写到磁盘，成功后再向leader回复ack；  
>- 4.当leader收到合法数量（超半数节点）的ACK 后，leader向这些follower发送commit命令，同时在本地执行该消息；  
>- 5.当follower收到消息的commit命令后，会提交该消息leader的投票过程，不需observer的ack，即observer不需参与投票过程，但需同步leader数据从而在处理请求时保证数据一致性。

### 崩溃恢复
>ZAB协议基于原子广播协议的消息广播过程，一旦 leader崩溃，或网络问题导致leader失去过半follower联系（可能leader和follower间产生网络分区，此时leader不再合法），会进入崩溃恢复模式。在 ZAB协议中，为保证程序正确运行，整个恢复过程结束后需选举出一个新leader。  
为使leader挂了后系统正常工作，需解决以下两
个问题：
>- 1.已被处理的消息不能丢失。当leader收到合法数量的follower的ACK后，向各follower广播COMMIT命令，同时在本地执行COMMIT并返回给客户端。   若各follower收到COMMIT前，leader挂了，导致剩下的服务器没执行该消息。  
leader对事务消息发起commit操作，在follower1执行了，follower2还没收到commit已经挂了，而客户端已收到事务处理成功的响应。故zab协议需保证所有机器都要执行该事务消息。  
>- 2.被丢弃的消息不能再出现。当leader收到消息请求生成proposal后挂了，其他follower没收到此 proposal，经恢复模式重新选了leader后，该消息被跳过。此时之前挂的leader重启并注册成follower，保留被跳过的proposal状态，与整个系统状态是不一致的，需将其删除。  

>ZAB协议需满足以上两情况，要设计leader选举算法：确保已被leader提交的事务proposal能够提交、 同时丢弃已被跳过的事务proposal。  
>- 1.若leader选举算法能保证新选举出的leader拥有集群中所有机器最高编号（ZXID最大）的事务
proposal， 保证新选举的leader一定具有已提交提案。必须有超半数服务器的事务日志有该提案的 proposal， 故有合法数量的节点正常工作，必然有一个节点保存了所有被COMMIT消息的proposal 状态。  
>- 2.zxid是64位，高32位：epoch编号，每经过
一次leader选举产生新leader，epoch号+1；低32 位：消息计数器，每收到一条消息，值+1，新leader选举后值重置为0。——好处：老leader挂了后重启，不会被选举为leader，因此zxid肯定小于当前新leader。当老leader作为follower接入新leader后，新leader会让它将所有拥有旧epoch号的未被COMMIT的proposal清除。  
#### ZXID
>即事务id，为保证事务顺序的一致性，zookeeper 采用递增事务id号（zxid）标识事务。所有提议（proposal）都在被提出时加上zxid。zxid是64位数字，高32位是epoch（ZAB协议通过epoch编号区分leader周期变化策略）用来标识leader关系是否改变，每次一个leader被选出，都会有一个新epoch=（原epoch+1），标识当前属于那个leader的统治时期。低32位用于递增计数。
>>epoch：可理解为当前集群所处年代或周期，每个leader像皇帝有年号，每次改朝换代后，都会在前一个年代加1。旧leader崩溃恢复后，也无人听它，follower只听从当前年代的leader命令。

- epoch变化实验
>1.启动一个zookeeper集群；  
2.在/tmp/zookeeper/VERSION-2路径下的currentEpoch文件，显示的是当前的epoch；  
3.把leader停机，观看currentEpoch的变化。随着每次选举新leader，epoch都会发生变化。  

### leader选举
>leader选举分两个过程：启动时的leader选举、 leader崩溃时的选举。  
>服务器启动时的leader选举，每个节点启动时状态都是LOOKING（观望状态），至少需两台机器 。以3台为例，集群初始化阶段，当有一台服务器 server1启动时，它本身无法进行和完成leader选举，当第二台服务器server2启动时，此时可相互通信，都试图找到leader，于是进入leader选举过程：
>①每个server投票一次，初始情况server1、2都投自己为leader，每次投票包含所推举服务器的myid和ZXID、epoch，eg.server1投票为(1,0)，server2投票为(2,0)，然后将该投票发给集群中其他机器；  
②接受各服务器的投票。每个服务器收到投票后，先判断投票的有效性（eg.是否本轮投票epoch、是否LOOKING状态服务器）；    
③处理投票。针对每个投票，服务器都需将别的投票和自己的投票PK，规则如下：
>>i.优先检查ZXID，比较大的优先作为leader；  
ii.ZXID同，则比较myid，较大的作为leader；
eg.server1投票：(1, 0)，server2投票：(2,0)，比较两者ZXID，均为 0，再比较 myid，server2较大。于是更新自己的投票为(2, 0)， 然后重新投票。  
server2不需更新自己的投票，只需再向集群发出上一次投票信息即可。  

>④统计投票。每次投票后，服务器会统计投票信息，判断是否已过半机器收到相同投票信息，对于server1、2而言，都统计出集群中已有两台机器收到(2,0)投票信息，则认为已选出leader；  
⑤改变服务器状态。一旦确定leader，每个服务器会更新状态：follower->FOLLOWING、leader->LEADING。

#### 运行过程中的leader选举
>当集群的leader宕机或不可用时，集群将无法对外提供服务，进入新一轮选举，运行期间leader选举和启动时期基本过程一致。  
①变更状态。leader挂后，余下的非Observer都会将自己的服务器状态变更为LOOKING，然后进入leader选举过程；  
②每个server发出一个投票，运行期间每个服务器的ZXID可能不同。eg.server1的ZXID=123， Server3的ZXID=122，第一轮投票，server1、3都会投自己，产生投票(1,123)、(3, 122)，将投票发送给集群所有机器，接收来自各服务器的投票。与启动时同；  
③处理投票。与启动时同，此时server1成为leader；  
④统计投票。与启动时同；  
⑤改变服务器状态。与启动时同。  


### leader选举源码分析

#### 入口QuorumPeerMain
```
// org.apache.zookeeper.server.quorum.QuorumPeerMain
public static void main(String[] args) {
    QuorumPeerMain main = new QuorumPeerMain();
    try {
        main.initializeAndRun(args);
        ......
        
protected void initializeAndRun(String[] args) throws ConfigException, IOException {
    QuorumPeerConfig config = new QuorumPeerConfig();
    if (args.length == 1) {
        config.parse(args[0]);
    }

    // Start and schedule the the purge task
    DatadirCleanupManager purgeMgr = new DatadirCleanupManager(config.getDataDir(), config.getDataLogDir(),config.getSnapRetainCount(),config.getPurgeInterval());
    purgeMgr.start();

    //判断是standalone或集群模式
    if (args.length == 1 && config.servers.size() > 0) {
        runFromConfig(config);
    } else {
        // there is only server in the quorum -- run as standalone
        ZooKeeperServerMain.main(args);
    }
}

public void runFromConfig(QuorumPeerConfig config) throws IOException {
  try {
      ManagedUtil.registerLog4jMBeans();
  } catch (JMException e) {
      LOG.warn("Unable to register log4j JMX control", e);
  }

  LOG.info("Starting quorum peer");
  try {
      // 为客户端提供读写的server，2181端口的访问功能
      ServerCnxnFactory cnxnFactory = ServerCnxnFactory.createFactory();
      cnxnFactory.configure(config.getClientPortAddress(), config.getMaxClientCnxns());

      // ZK的逻辑主线程，负责选举、投票
      quorumPeer = getQuorumPeer();
      quorumPeer.setQuorumPeers(config.getServers());
      quorumPeer.setTxnFactory(new FileTxnSnapLog(new File(config.getDataLogDir()), new File(config.getDataDir())));
      quorumPeer.setElectionType(config.getElectionAlg());
      quorumPeer.setMyid(config.getServerId());
      quorumPeer.setTickTime(config.getTickTime());
      quorumPeer.setInitLimit(config.getInitLimit());
      quorumPeer.setSyncLimit(config.getSyncLimit());
      quorumPeer.setQuorumListenOnAllIPs(config.getQuorumListenOnAllIPs());
      quorumPeer.setCnxnFactory(cnxnFactory);
      quorumPeer.setQuorumVerifier(config.getQuorumVerifier());
      quorumPeer.setClientPortAddress(config.getClientPortAddress());
      quorumPeer.setMinSessionTimeout(config.getMinSessionTimeout());
      quorumPeer.setMaxSessionTimeout(config.getMaxSessionTimeout());
      quorumPeer.setZKDatabase(new ZKDatabase(quorumPeer.getTxnFactory()));
      quorumPeer.setLearnerType(config.getPeerType());
      quorumPeer.setSyncEnabled(config.getSyncEnabled());

      // sets quorum sasl authentication configurations
      quorumPeer.setQuorumSaslEnabled(config.quorumEnableSasl);
      if (quorumPeer.isQuorumSaslAuthEnabled()) {
          quorumPeer.setQuorumServerSaslRequired(config.quorumServerRequireSasl);
          quorumPeer.setQuorumLearnerSaslRequired(config.quorumLearnerRequireSasl);
          quorumPeer.setQuorumServicePrincipal(config.quorumServicePrincipal);
          quorumPeer.setQuorumServerLoginContext(config.quorumServerLoginContext);
          quorumPeer.setQuorumLearnerLoginContext(config.quorumLearnerLoginContext);
      }
      quorumPeer.setQuorumCnxnThreadsSize(config.quorumCnxnThreadsSize);
      quorumPeer.initialize();
      // 启动主线程，QuorumPeer重写了Thread.start方法
      quorumPeer.start();
      quorumPeer.join();
  } catch (InterruptedException e) {
      // warn, but generally this is ok
      LOG.warn("Quorum Peer interrupted", e);
  }
}
```
#### 调用QuorumPeer的start方法
```java 
// org.apache.zookeeper.server.quorum.QuorumPeer
public synchronized void start() {
    // 恢复DB
    loadDataBase();
    cnxnFactory.start();
    // 选举初始化
    startLeaderElection();
    super.start();
}

/**主要从本地文件中恢复数据,获取最新zxid**/
private void loadDataBase() {
    File updating = new File(getTxnFactory().getSnapDir(), UPDATING_EPOCH_FILENAME);
    try {
        // 从本地文件恢复db
        zkDb.loadDataBase();
    
        // load the epochs
        // 从最新zxid恢复epoch变量、zxid64位、前32位是epoch值，后32位是zxid
        long lastProcessedZxid = zkDb.getDataTree().lastProcessedZxid;
        long epochOfZxid = ZxidUtils.getEpochFromZxid(lastProcessedZxid);
        try {
            // 从文件中读取当前epoch
            currentEpoch = readLongFromFile(CURRENT_EPOCH_FILENAME);
            if (epochOfZxid > currentEpoch && updating.exists()) {
                if (!updating.delete()) {
                    throw new IOException("Failed to delete " + updating.toString());
                }
            }
        } catch (FileNotFoundException e) {
            currentEpoch = epochOfZxid;
            LOG.info(CURRENT_EPOCH_FILENAME+ " not found! Creating with a reasonable default of {}. This should only happen when you are upgrading your installation",currentEpoch);
            writeLongToFile(CURRENT_EPOCH_FILENAME, currentEpoch);
        }
        ......
```
#### 初始化LeaderElection
```java 
// org.apache.zookeeper.server.quorum.QuorumPeer
synchronized public void startLeaderElection() {
	try {
        // 投票给自己
		currentVote = new Vote(myid, getLastLoggedZxid(), getCurrentEpoch());
	} catch(IOException e) {
		RuntimeException re = new RuntimeException(e.getMessage());
		re.setStackTrace(e.getStackTrace());
		throw re;
	}
    for (QuorumServer p : getView().values()) {
        if (p.id == myid) {
            myQuorumAddr = p.addr;
            break;
        }
    }
    if (myQuorumAddr == null) {
        throw new RuntimeException("My id " + myid + " not in the peer list");
    }
    if (electionType == 0) {
        try {
            udpSocket = new DatagramSocket(myQuorumAddr.getPort());
            responder = new ResponderThread();
            responder.start();
        } catch (SocketException e) {
            throw new RuntimeException(e);
        }
    }
    // 根据配置获取选举算法
    this.electionAlg = createElectionAlgorithm(electionType);
}

protected Election createElectionAlgorithm(int electionAlgorithm){
    Election le=null;
            
    //TODO: use a factory rather than a switch
    switch (electionAlgorithm) {
    case 0:
        le = new LeaderElection(this);
        break;
    case 1:
        le = new AuthFastLeaderElection(this);
        break;
    case 2:
        le = new AuthFastLeaderElection(this, true);
        break;
    case 3:
        // leader选举IO负责类
        qcm = createCnxnManager();
        QuorumCnxManager.Listener listener = qcm.listener;
        if(listener != null){
            // 启动已绑定端口的选举线程，等待集群中其他线程
            listener.start();
            // 基于TCP的选举算法
            le = new FastLeaderElection(this, qcm);
        } else {
            LOG.error("Null listener when initializing cnx manager");
        }
        ......
```
>配置选举算法有3种zoo.cfg配置，默认fast选举。  
>继续看FastLeaderElection的初始化，主要初始化业务层的发送和接收队列。
```java 
// org.apache.zookeeper.server.quorum.FastLeaderElection
public FastLeaderElection(QuorumPeer self, QuorumCnxManager manager){
    this.stop = false;
    this.manager = manager;
    starter(self, manager);
}

private void starter(QuorumPeer self, QuorumCnxManager manager) {
    this.self = self;
    proposedLeader = -1;
    proposedZxid = -1;
    // 业务层发送队列，业务对象ToSend
    sendqueue = new LinkedBlockingQueue<ToSend>();
    // 业务层发送队列，业务对象Notification
    recvqueue = new LinkedBlockingQueue<Notification>();
    this.messenger = new Messenger(manager);
}

// org.apache.zookeeper.server.quorum.FastLeaderElection
Messenger(QuorumCnxManager manager) {
    this.ws = new WorkerSender(manager);

    Thread t = new Thread(this.ws, "WorkerSender[myid=" + self.getId() + "]");
    t.setDaemon(true);
    t.start();

    this.wr = new WorkerReceiver(manager);

    t = new Thread(this.wr, "WorkerReceiver[myid=" + self.getId() + "]");
    t.setDaemon(true);
    t.start();
}
```
>接下来调用fle.start()，即调用FastLeaderElection的start方法，该方法主要初始化发送和接收线程， 左边是FastLeaderElection的start，右边 是messager.start()。
>然后再回到QuorumPeer。FastLeaderElection 初始化完后，调用super.start()，最终运行QuorumPeer的run方法。

- 前面部分主要做JMX监控注册
```java 
// org.apache.zookeeper.server.quorum.QuorumPeer
public void run() {
    setName("QuorumPeer" + "[myid=" + getId() + "]" +
            cnxnFactory.getLocalAddress());

    LOG.debug("Starting quorum peer");
    try {
        // 此处通过JMX监控一些属性
        jmxQuorumBean = new QuorumBean(this);
        MBeanRegistry.getInstance().register(jmxQuorumBean, null);
        for (QuorumServer s : getView().values()) {
            ZKMBeanInfo p;
            if (getId() == s.id) {
                p = jmxLocalPeerBean = new LocalPeerBean(this);
                try {
                    MBeanRegistry.getInstance().register(p, jmxQuorumBean);
                } catch (Exception e) {
                    LOG.warn("Failed to register with JMX", e);
                    jmxLocalPeerBean = null;
                }
            } else {
                p = new RemotePeerBean(s);
                try {
                    MBeanRegistry.getInstance().register(p, jmxQuorumBean);
    ......
    
    while (running) {
        // 判断当前节点状态
        switch (getPeerState()) {
        // 若是LOOKING，则进入选举流程
        case LOOKING:
            LOG.info("LOOKING");

            if (Boolean.getBoolean("readonlymode.enabled")) {
                LOG.info("Attempting to start ReadOnlyZooKeeperServer");

                // Create read-only server but don't start it immediately
                final ReadOnlyZooKeeperServer roZk = new ReadOnlyZooKeeperServer(logFactory, this,
                        new ZooKeeperServer.BasicDataTreeBuilder(), this.zkDb);

                // Instead of starting roZk immediately, wait some grace
                // period before we decide we're partitioned.
                //
                // Thread is used here because otherwise it would require
                // changes in each of election strategy classes which is
                // unnecessary code coupling.
                Thread roZkMgr = new Thread() {
                    public void run() {
                        try {
                            // lower-bound grace period to 2 secs
                            sleep(Math.max(2000, tickTime));
                            if (ServerState.LOOKING.equals(getPeerState())) {
                                roZk.startup();
                            }
                        } catch (InterruptedException e) {
                            LOG.info(
                                    "Interrupted while attempting to start ReadOnlyZooKeeperServer, not started");
                        } catch (Exception e) {
                            LOG.error("FAILED to start ReadOnlyZooKeeperServer", e);
                        }
                    }
                };
                try {
                    roZkMgr.start();
                    setBCVote(null);
                    // 通过策略模式决定当前用FastLeaderElection的哪个选举算法 setCurrentVote(makeLEStrategy().lookForLeader());
                    ......
```
#### lookForLeader开始选举
```java 
// org.apache.zookeeper.server.quorum.FastLeaderElection
public Vote lookForLeader() throws InterruptedException {
    try {
        self.jmxLeaderElectionBean = new LeaderElectionBean();
        MBeanRegistry.getInstance().register(self.jmxLeaderElectionBean, self.jmxLocalPeerBean);
    } catch (Exception e) {
        LOG.warn("Failed to register with JMX", e);
        self.jmxLeaderElectionBean = null;
    }
    if (self.start_fle == 0) {
        self.start_fle = Time.currentElapsedTime();
    }
    try {
        // 收到的投票
        HashMap<Long, Vote> recvset = new HashMap<Long, Vote>();
        // 存储选举结果
        HashMap<Long, Vote> outofelection = new HashMap<Long, Vote>();

        int notTimeout = finalizeWait;

        synchronized (this) {
            // 增加逻辑时钟
            logicalclock.incrementAndGet();
            // 更新自己的zxid和epoch
            updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
        }

        LOG.info("New election. My id =  " + self.getId() +", proposed zxid=0x" + Long.toHexString(proposedZxid));
        // 发送投票，包括发送给自己
        sendNotifications();

        /*
         * Loop in which we exchange notifications until we find a leader
         */
        // 主循环，直到选举出leader
        while ((self.getPeerState() == ServerState.LOOKING) && (!stop)) {
            /*
             * Remove next notification from queue, times out after 2 times
             * the termination time
             */
            // 从IO线程里拿到投票信息（包括自己的投票）
            Notification n = recvqueue.poll(notTimeout, TimeUnit.MILLISECONDS);

            /*
             * Sends more notifications if haven't received enough.
             * Otherwise processes new notification.
             */
            if (n == null) {
                // 消息发完了，继续发送，直到选出leader
                if (manager.haveDelivered()) {
                    sendNotifications();
                } else {
                    // 消息还没投递出去，可能是其他server还没启动，尝试再连接
                    manager.connectAll();
                }

                /*
                 * Exponential backoff
                 */
                // 延长超时时间
                int tmpTimeOut = notTimeout * 2;
                notTimeout = (tmpTimeOut < maxNotificationInterval ? tmpTimeOut : maxNotificationInterval);
                LOG.info("Notification time out: " + notTimeout);
            }
            // 收到了投票信息，判断收到的消息是不是属于该集群
            else if (validVoter(n.sid) && validVoter(n.leader)) {
                /*
                 * Only proceed if the vote comes from a replica in the
                 * voting view for a replica in the voting view.
                 */
                // 判断收到消息的节点状态
                switch (n.state) {
                case LOOKING:
                    // If notification > current, replace and send messages out
                    // 判断接收到的节点epoch大于logicalclock，则表示当前是新一轮选举
                    if (n.electionEpoch > logicalclock.get()) {
                        // 更新本地logicalclock
                        logicalclock.set(n.electionEpoch);
                        // 清空接收队列
                        recvset.clear();
                        // 检查收到该消息是否可胜出，一次比较epoch、zxid、myid
                        if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, getInitId(), getInitLastLoggedZxid(),getPeerEpoch())) {
                            // 胜出后，将投票改为对方的票据
                            updateProposal(n.leader, n.zxid, n.peerEpoch);
                        }
                        // 否则，票据不变
                        else {
                            updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
                        }
                        // 继续广播消息，让其他节点知道我目前的票据
                        sendNotifications();
                    }
                    // 若收到的消息epoch小于当前节点的epoch，则忽略
                    else if (n.electionEpoch < logicalclock.get()) {
                        if (LOG.isDebugEnabled()) {
                            LOG.debug("Notification election epoch is smaller than logicalclock. n.electionEpoch = 0x"
                                            + Long.toHexString(n.electionEpoch) + ", logicalclock=0x" + Long
                                            .toHexString(logicalclock.get()));
                        }
                        break;
                    }
                    // 若epoch同，则比较zxid、myid，若胜出则更新自己的票据，广播
                    else if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, proposedLeader, proposedZxid, proposedEpoch)) {
                        updateProposal(n.leader, n.zxid, n.peerEpoch);
                        sendNotifications();
                    }

                    // 添加到本机投票集合，用来做选举终结判断
                    recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));
                    // 判断选举是否结束，默认算法：超半数同意
                    if (termPredicate(recvset,new Vote(proposedLeader, proposedZxid, logicalclock.get(), proposedEpoch))) {
                        // Verify if there is any change in the proposed leader
                        // 一直等新的notification到达，直到超时
                        while ((n = recvqueue.poll(finalizeWait, TimeUnit.MILLISECONDS)) != null) {
                            if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, proposedLeader, proposedZxid,
                                    proposedEpoch)) {
                                recvqueue.put(n);
                                break;
                            }
                        }

                        /*
                         * This predicate is true once we don't read any new
                         * relevant message from the reception queue
                         */
                        // 确定leader
                        if (n == null) {
                            // 修改状态，LEADING or FOLLOWING
                            self.setPeerState((proposedLeader == self.getId()) ? ServerState.LEADING : learningState());
                            // 返回最终投票结果
                            Vote endVote = new Vote(proposedLeader, proposedZxid, logicalclock.get(),
                                    proposedEpoch);
                            leaveInstance(endVote);
                            return endVote;
                        }
                    }
                    break;
                // 若收到选票状态非LOOKING，eg.刚加入一个已正在运行的zk集群时，OBSERVING不参与选举
                case OBSERVING:
                    LOG.debug("Notification from observer: " + n.sid);
                    break;
                // 这2种需参与选举
                case FOLLOWING:
                case LEADING:
                    /*
                     * Consider all notifications from the same epoch
                     * together.
                     */
                    // 判断epoch是否相同
                    if (n.electionEpoch == logicalclock.get()) {
                        // 加入到本机的投票集合
                        recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));
                        // 投票是否结束，若结束，再确认leader是否有效，修改自己的状态并返回投票结果
                        if (ooePredicate(recvset, outofelection, n)) {
                            self.setPeerState((n.leader == self.getId()) ? ServerState.LEADING : learningState());

                            Vote endVote = new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch);
                            leaveInstance(endVote);
                            return endVote;
                        }
                    }

                    /*
                     * Before joining an established ensemble, verify
                     * a majority is following the same leader.
                     */
                    outofelection.put(n.sid,new Vote(n.version, n.leader, n.zxid, n.electionEpoch, n.peerEpoch, n.state));

                    if (ooePredicate(outofelection, outofelection, n)) {
                        synchronized (this) {
                            logicalclock.set(n.electionEpoch);
                            self.setPeerState((n.leader == self.getId()) ? ServerState.LEADING : learningState());
                        }
                        Vote endVote = new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch);
                        leaveInstance(endVote);
                        return endVote;
                    }
                    break;
                default:
                    LOG.warn("Notification state unrecognized: {} (n.state), {} (n.sid)", n.state, n.sid);
                    break;
                }
            } else {
                if (!validVoter(n.leader)) {
                    LOG.warn("Ignoring notification for non-cluster member sid {} from sid {}", n.leader, n.sid);
                }
                if (!validVoter(n.sid)) {
                    LOG.warn("Ignoring notification for sid {} from non-quorum member sid {}", n.leader, n.sid);
                }
            }
        }
        return null;
    } finally {
        try {
            if (self.jmxLeaderElectionBean != null) {
                MBeanRegistry.getInstance().unregister(self.jmxLeaderElectionBean);
            }
        } catch (Exception e) {
            LOG.warn("Failed to unregister with JMX", e);
        }
        self.jmxLeaderElectionBean = null;
        LOG.debug("Number of connection processing threads: {}", manager.getConnectionThreadCount());
    }
}
```

- totalOrderPredicate
```java 
protected boolean totalOrderPredicate(long newId, long newZxid, long newEpoch, long curId, long curZxid, long curEpoch) {
    LOG.debug("id: " + newId + ", proposed id: " + curId + ", zxid: 0x" + Long.toHexString(newZxid) + ", proposed zxid: 0x" + Long.toHexString(curZxid));
    if(self.getQuorumVerifier().getWeight(newId) == 0){
        return false;
    }
    
    /*
     * We return true if one of the following three cases hold:
     * 1- New epoch is higher
     * 2- New epoch is the same as current epoch, but new zxid is higher
     * 3- New epoch is the same as current epoch, new zxid is the same
     *  as current zxid, but server id is higher.
     */
    // 判断epoch是否比当前大。①若是则消息中id对应的服务器就是leader；②若相等则判断zxid，zxid大的，消息中id对应的服务器就是leader；③若epoch、zxid都相等，则比较id，大的就是leader。
    return ((newEpoch > curEpoch) || 
            ((newEpoch == curEpoch) &&
            ((newZxid > curZxid) || ((newZxid == curZxid) && (newId > curId)))));
}
```
#### 消息如何广播（sendNotifications）
```java 
// org.apache.zookeeper.server.quorum.FastLeaderElection
private void sendNotifications() {
    // 循环发送
    for (QuorumServer server : self.getVotingView().values()) {
        long sid = server.id;

        // 消息实体
        ToSend notmsg = new ToSend(ToSend.mType.notification, proposedLeader, proposedZxid, logicalclock.get(),QuorumPeer.ServerState.LOOKING, sid, proposedEpoch);
        // 添加到发送队列，该队列会被workerSender消费
        sendqueue.offer(notmsg);
    }
}

// org.apache.zookeeper.server.quorum.FastLeaderElection.Messenger.WorkerSender
class WorkerSender extends ZooKeeperThread {
    volatile boolean stop;
    QuorumCnxManager manager;
    
    WorkerSender(QuorumCnxManager manager) {
        super("WorkerSender");
        this.stop = false;
        this.manager = manager;
    }
    
    public void run() {
        while (!stop) {
            try {
                // 从发送队列中获取消息实体
                ToSend m = sendqueue.poll(3000, TimeUnit.MILLISECONDS);
                if (m == null)
                    continue;
    
                process(m);
            } catch (InterruptedException e) {
                break;
            }
        }
        LOG.info("WorkerSender is down");
    }
    
    /**
     * Called by run() once there is a new message to send.
     *
     * @param m message to send
     */
    void process(ToSend m) {
        ByteBuffer requestBuffer = buildMsg(m.state.ordinal(), m.leader, m.zxid, m.electionEpoch, m.peerEpoch);
        manager.toSend(m.sid, requestBuffer);
    }
    }
    
    WorkerSender ws;
    WorkerReceiver wr;
    
    Messenger(QuorumCnxManager manager) {
    this.ws = new WorkerSender(manager);
    
    Thread t = new Thread(this.ws, "WorkerSender[myid=" + self.getId() + "]");
    t.setDaemon(true);
    t.start();
    
    this.wr = new WorkerReceiver(manager);
    
    t = new Thread(this.wr, "WorkerReceiver[myid=" + self.getId() + "]");
    t.setDaemon(true);
    t.start();
    }
    
    /**
    * Stops instances of WorkerSender and WorkerReceiver
    */
    void halt() {
    this.ws.stop = true;
    this.wr.stop = true;
    }
}

// org.apache.zookeeper.server.quorum.QuorumCnxManager
public void toSend(Long sid, ByteBuffer b) {
    /*
     * If sending message to myself, then simply enqueue it (loopback).
     */
    // 若是自己，则不走网络发送，直接添加到本地接收队列
    if (this.mySid == sid) {
         b.position(0);
         addToRecvQueue(new Message(b.duplicate(), sid));
        /*
         * Otherwise send to the corresponding thread to send.
         */
    } else {
         /*
          * Start a new connection if doesn't have one already.
          */
        // 发送给别的节点，判断之前是不是发送过
        ArrayBlockingQueue<ByteBuffer> bq = new ArrayBlockingQueue<ByteBuffer>(SEND_CAPACITY);
        // 该SEND_CAPACITY大小是1，若之前已有一个待发送，会把之前的一个删除，发送新的
        ArrayBlockingQueue<ByteBuffer> bqExisting = queueSendMap.putIfAbsent(sid, bq);
        if (bqExisting != null) {
            addToSendQueue(bqExisting, b);
        } else {
            addToSendQueue(bq, b);
        }
        // 真正的发送逻辑
        connectOne(sid);
    }
}
```
#### FastLeaderElection选举过程
>在投票过程中涉及几个类：

- FastLeaderElection
>实现了Election接口，实现各服务器间基于TCP 协议进行选举
- Notification（org.apache.zookeeper.server.quorum.AuthFastLeaderElection.Notification、org.apache.zookeeper.server.quorum.FastLeaderElection.Notification）
>内部类， 表示收到的选举投票信息（其他服务器发来的选举投票信息） ，包含被选举者id、zxid、选举周期等
- ToSend
>表示发送给其他服务器的选举投票信息，也包含了被选举者id、zxid、选举周期等
- Messenger（org.apache.zookeeper.server.quorum.FastLeaderElection.Messenger、org.apache.zookeeper.server.quorum.AuthFastLeaderElection.Messenger）
>包含了WorkerReceiver和WorkerSender两个内部类；
>>- WorkerReceiver实现Runnable 接口，选票接收器。会不断从QuorumCnxManager中获取其他服务器发来的选举消息，并将其转换成一个选票，保存到recvqueue中；
>>- WorkerSender也实现了Runnable接口，选票发送器，其会不断从sendqueue中获取待发送选票，并将其传递到底层QuorumCnxManager中。


https://blog.csdn.net/qq_16038125/article/details/80920240