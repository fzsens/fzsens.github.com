### Zookeeper服务启动

ZooKeeper是一个典型的分布式数据一致性的解决方案，分布式应用程序可以基于它实现诸如数据发布/订阅、负债均衡、命名服务、分布式协调/通知、集群管理、Master选举、分布式锁和分布式队列等功能。

从本质上，Zookeeper类似一个分布式的存储系统，通过以Leader-Followers的模式运行的集群来实现高可用。本文主要分析单机模式的zk的启动流程。

![zk-start-diagram](http://i.imgur.com/X9BWilY.png)

#### 启动入口分析

以`Windows`下的启动例子，找到`zkServer.cmd`，主要的内容为

````shell
# zkServer.cmd

set ZOOMAIN=org.apache.zookeeper.server.quorum.QuorumPeerMain
echo on
call %JAVA% "-Dzookeeper.log.dir=%ZOO_LOG_DIR%" "-Dzookeeper.root.logger=%ZOO_LOG4J_PROP%" -cp "%CLASSPATH%" %ZOOMAIN% "%ZOOCFG%" %*
````
可以找到启动的Main类为`QuorumPeerMain`,参数为`ZOOCFG`，也就是zk的配置文件路径。

````java
    
// QuorumPeerMain

/**
 * zk 接收config配置启动
 */
public static void main(String[] args) {
    QuorumPeerMain main = new QuorumPeerMain();
    main.initializeAndRun(args);
}

/**
 * 初始化zk，并运行
 */
protected void initializeAndRun(String[] args)
    throws ConfigException, IOException
{
    QuorumPeerConfig config = new QuorumPeerConfig();
    if (args.length == 1) {
        config.parse(args[0]);
    }
    // Start and schedule the the purge task
    // 清理txnlog 和snapshot
    DatadirCleanupManager purgeMgr = new DatadirCleanupManager(config
            .getDataDir(), config.getDataLogDir(), config
            .getSnapRetainCount(), config.getPurgeInterval());
    purgeMgr.start();

    if (args.length == 1 && config.servers.size() > 0) {
        runFromConfig(config);
    } else {
        // 单机模式
        ZooKeeperServerMain.main(args);
    }
}
````

可以看出，`QuorumPeerMain`中，完成了`DatadirCleanupManager`的创建和运行，并将单机启动的工作交给`ZooKeeperServerMain`进行处理。

#### 清理snapshot和TxnLog

`DatadirCleanupManager`的主要工作是定期清理snapshot和snapshot对应的TxnLog，可以通过`zoo.cfg`中的`autopurge.purgeInterval`和`autopurge.snapRetainCount`来调整执行频率和需要保留的snapshot数量。

>TxnLog记录了zk接收到的事务请求，zk的事务指的是会影响zk服务器状态的操作，包括节点的创建、节点数据的更新、回话的创建和关闭、节点quota的配置等，每一个事务操作都会被记录到TxnLog中，并分配一个zxid，zxid是全序并且唯一的。整个zk实际上也是一个状态机，如果将所有的TxnLog在两台全新的zk上执行，得到的状态是完全一致的。为了提高zk性能，在运行时，会将所有的数据存储在内存中。但是每一次重启后内存中的数据就会消失。zk会在数据恢复节点从TxnLog中奖所有的事务节点执行一遍。但是显然随着TxnLog的增长这样的效率会变得越来越差，因此zk会每隔一段时间或者在处理一定数量的事务之后，将内存中的数据序列化后写入到磁盘中作为恢复快照。这样重启恢复数据的时候，只需要将snapshot先反序列化之后读入磁盘，再将部分TxnLog进行重放，即可快速恢复数据。

````java
//DatadirCleanupManager

public void start() {
    if (PurgeTaskStatus.STARTED == purgeTaskStatus) {
        LOG.warn("Purge task is already running.");
        return;
    }
    // Don't schedule the purge task with zero or negative purge interval.
    if (purgeInterval <= 0) {
        LOG.info("Purge task is not scheduled.");
        return;
    }

    timer = new Timer("PurgeTask", true);
    TimerTask task = new PurgeTask(dataLogDir, snapDir, snapRetainCount);
    timer.scheduleAtFixedRate(task, 0, TimeUnit.HOURS.toMillis(purgeInterval));

    purgeTaskStatus = PurgeTaskStatus.STARTED;
}
````

`DatadirCleanupManager`启动之后，会启动一个定时器，通过`PurgeTask`将可以清理的snapshot和对应的TxnLog清理掉。

````java
//PureTxnLog

public static void purge(File dataDir, File snapDir, int num) throws IOException {
    if (num < 3) {
        throw new IllegalArgumentException(COUNT_ERR_MSG);
    }

    FileTxnSnapLog txnLog = new FileTxnSnapLog(dataDir, snapDir);

    List<File> snaps = txnLog.findNRecentSnapshots(num);
    retainNRecentSnapshots(txnLog, snaps);
}

// VisibleForTesting
static void retainNRecentSnapshots(FileTxnSnapLog txnLog, List<File> snaps) {
    // found any valid recent snapshots?
    if (snaps.size() == 0)
        return;
    File snapShot = snaps.get(snaps.size() -1);
    final long leastZxidToBeRetain = Util.getZxidFromName(
            snapShot.getName(), PREFIX_SNAPSHOT);

    class MyFileFilter implements FileFilter{
        private final String prefix;
        MyFileFilter(String prefix){
            this.prefix=prefix;
        }
        public boolean accept(File f){
            if(!f.getName().startsWith(prefix + "."))
                return false;
            long fZxid = Util.getZxidFromName(f.getName(), prefix);
            // 根据文件的末尾zxid标记判断是否删除
            if (fZxid >= leastZxidToBeRetain) {
                return false;
            }
            return true;
        }
    }
    // add all non-excluded log files
    List<File> files = new ArrayList<File>(Arrays.asList(txnLog
            .getDataDir().listFiles(new MyFileFilter(PREFIX_LOG))));
    // add all non-excluded snapshot files to the deletion list
    files.addAll(Arrays.asList(txnLog.getSnapDir().listFiles(
            new MyFileFilter(PREFIX_SNAPSHOT))));
    // remove the old files
    for(File f: files)
    {
        System.out.println("Removing file: "+
            DateFormat.getDateTimeInstance().format(f.lastModified())+
            "\t"+f.getPath());
        if(!f.delete()){
            System.err.println("Failed to remove "+f.getPath());
        }
    }

}
````
`PureTask`通过`PurgeTxnLog`进行文件的清理。因此snapshot和txnlog的后缀都是全序的zxid，因此可以通过zxid进行判断和过滤。

#### 启动和初始化工作

````java
//ZooKeeperServerMain

/**
 * 单机启动
 */
protected void initializeAndRun(String[] args)
    throws ConfigException, IOException
{
    try {
        //通过JMX来设置log4j的配置
        ManagedUtil.registerLog4jMBeans();
    } catch (JMException e) {
        LOG.warn("Unable to register log4j JMX control", e);
    }

    // zoo.cfg 作为初始化条件
    ServerConfig config = new ServerConfig();
    if (args.length == 1) {
        config.parse(args[0]);
    } else {
        config.parse(args);
    }

    runFromConfig(config);
}

/**
 * 开始启动
 * Run from a ServerConfig.
 * @param config ServerConfig to use.
 * @throws IOException
 */
public void runFromConfig(ServerConfig config) throws IOException {
    LOG.info("Starting server");
    FileTxnSnapLog txnLog = null;
    try {

        ZooKeeperServer zkServer = new ZooKeeperServer();

        // 创建txn log文件
        txnLog = new FileTxnSnapLog(new File(config.dataLogDir), new File(
                config.dataDir));
        zkServer.setTxnLogFactory(txnLog);
		//心跳频率
        zkServer.setTickTime(config.tickTime);
		// session超时
        zkServer.setMinSessionTimeout(config.minSessionTimeout);
        zkServer.setMaxSessionTimeout(config.maxSessionTimeout);
        // Zk 默认的通信使用过的原生的NIO 可以通过环境变量设置为Netty
        cnxnFactory = ServerCnxnFactory.createFactory();
        cnxnFactory.configure(config.getClientPortAddress(),
                config.getMaxClientCnxns());
		// zk服务
        cnxnFactory.startup(zkServer);
        cnxnFactory.join();
        if (zkServer.isRunning()) {
            zkServer.shutdown();
        }
    } catch (InterruptedException e) {
        // warn, but generally this is ok
        LOG.warn("Server interrupted", e);
    } finally {
        if (txnLog != null) {
            txnLog.close();
        }
    }
}
````

zk使用log4j和slf4j作为日志组件，可以通过JMX进行运行时监控和管理。

snapshot和txnlog是zk文件管理的核心，`FileTxnSnapLog`封装了对snapshot和txnlog的操作接口。在整个zk运行过程中，都是通过这个类来对snapshot和txnlog进行处理，包括了txn的写入、删除、以及zk恢复阶段的核心操作。

zk在启动之后，通过网络对外提供服务，zk提供了两个网络通信框架：分别是Netty3和NIO，默认使用NIO，可以通过`zookeeper.serverCnxnFactory`系统属性进行修改，实际上Netty也是基于NIO实现。实际的实现类为`NIOServerCnxnFactory`。

````java
/**
 * 启动 zkserver
 */
@Override
public void startup(ZooKeeperServer zks) throws IOException,
        InterruptedException {
    // 设置线程状态为启动，并开始NIO监听客户端请求
    start();
    // 设置Zookeeper Server
    setZooKeeperServer(zks);
    // 加载恢复数据
    zks.startdata();
    // 处理启动
    zks.startup();
}

//监听zk client的请求
public void run() {
    while (!ss.socket().isClosed()) {
        try {
            // 1s
            selector.select(1000);
            Set<SelectionKey> selected;
            synchronized (this) {
                selected = selector.selectedKeys();
            }
            ArrayList<SelectionKey> selectedList = new ArrayList<SelectionKey>(
                    selected);
            Collections.shuffle(selectedList);
            for (SelectionKey k : selectedList) {
                if ((k.readyOps() & SelectionKey.OP_ACCEPT) != 0) {
                    //连接
                    SocketChannel sc = ((ServerSocketChannel) k
                            .channel()).accept();
                    //client ip
                    InetAddress ia = sc.socket().getInetAddress();
                    int cnxncount = getClientCnxnCount(ia);
					// 限制客户端连接数
                    if (maxClientCnxns > 0 && cnxncount >= maxClientCnxns) {
                        LOG.warn("Too many connections from " + ia
                                + " - max is " + maxClientCnxns);
                        sc.close();
                    } else {
                        LOG.info("Accepted socket connection from "
                                + sc.socket().getRemoteSocketAddress());
                        sc.configureBlocking(false);
						//Channel READ
                        SelectionKey sk = sc.register(selector,
                                SelectionKey.OP_READ);
                        //将cnxn作为存储附件
                        NIOServerCnxn cnxn = createConnection(sc, sk);
                        sk.attach(cnxn);
                        addCnxn(cnxn);
                    }
                } else if ((k.readyOps() & (SelectionKey.OP_READ | SelectionKey.OP_WRITE)) != 0) {
                    // 读写请求
                    NIOServerCnxn c = (NIOServerCnxn) k.attachment();
                    // 处理请求
                    c.doIO(k);
                } else {
                    if (LOG.isDebugEnabled()) {
                        LOG.debug("Unexpected ops in select "
                                + k.readyOps());
                    }
                }
            }
            selected.clear();
        } catch (RuntimeException e) {
            LOG.warn("Ignoring unexpected runtime exception", e);
        } catch (Exception e) {
            LOG.warn("Ignoring exception", e);
        }
    }
    closeAll();
    LOG.info("NIOServerCnxn factory exited run method");
}
````

`NIOServerCnxnFactory`创建了一个NIO Server，创建连接之后，将NIO通信封装为`NIOServerCnxn`作为附件存放在`SelectionKey`中，并在客户端请求到达之后取出进行后续的请求处理。后续会专门有来讨论这个流程。

#### 数据恢复

数据恢复是zk启动服务的关键步骤，通过先前创建的`FileTxnSnapLog`对象中包含的snapshot和txnlog重新构建内存DataTree。

````java
// ZooKeeperServer
/**
 * 初始化数据
 *
 * @throws IOException
 * @throws InterruptedException
 */
public void startdata()
        throws IOException, InterruptedException {
    //check to see if zkDb is not null
    if (zkDb == null) {
		// 根据snapshot创建zbdb
        zkDb = new ZKDatabase(this.txnLogFactory);
    }
    if (!zkDb.isInitialized()) {
        loadData();
    }
}


````

