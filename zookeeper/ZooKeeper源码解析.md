##ZooKeeper源码浅析之一

###1、脚本

先看启动脚本：

windows版本：

zkEvn.cmd脚本

	REM %~dp0%  表示当前执行脚本的路径，d=drive,p=path

	set ZOOCFGDIR=%~dp0%..\conf
	set ZOO_LOG_DIR=%~dp0%..
	set ZOO_LOG4J_PROP=INFO,CONSOLE

	REM for sanity sake assume Java 1.6
	REM see: http://java.sun.com/javase/6/docs/technotes/tools/windows/java.html

	REM add the zoocfg dir to classpath
	set CLASSPATH=%ZOOCFGDIR%

	REM make it work in the release
	SET CLASSPATH=%~dp0..\*;%~dp0..\lib\*;%CLASSPATH%

	REM make it work for developers
	SET CLASSPATH=%~dp0..\build\classes;%~dp0..\build\lib\*;%CLASSPATH%

	set ZOOCFG=%ZOOCFGDIR%\zoo.cfg

zkServer.cmd脚本

	setlocal
	call "%~dp0zkEnv.cmd"

	set ZOOMAIN=org.apache.zookeeper.server.quorum.QuorumPeerMain
	echo on
	java "-Dzookeeper.log.dir=%ZOO_LOG_DIR%" "-Dzookeeper.root.logger=%ZOO_LOG4J_PROP%" -cp "%CLASSPATH%" %ZOOMAIN% "%ZOOCFG%" %*

	endlocal


从启动脚本上看，ZooKeeper的启动是执行QuorumPeerMain类的main方法。

###2、代码启动
启动过程的主方法如下：

```Java
   public static void main(String[] args) {
        QuorumPeerMain main = new QuorumPeerMain();
        try {
            //初始化+执行
            main.initializeAndRun(args);
        } catch (IllegalArgumentException e) {
            LOG.error("Invalid arguments, exiting abnormally", e);
            LOG.info(USAGE);
            System.err.println(USAGE);
            System.exit(2);
        } catch (ConfigException e) {
            LOG.error("Invalid config, exiting abnormally", e);
            System.err.println("Invalid config, exiting abnormally");
            System.exit(2);
        } catch (Exception e) {
            LOG.error("Unexpected exception, exiting abnormally", e);
            System.exit(1);
        }
        LOG.info("Exiting normally");
        System.exit(0);
    }
    
    protected void initializeAndRun(String[] args) throws ConfigException,
			IOException {
		/** 读取文件，然后填充配置对象 */
		QuorumPeerConfig config = new QuorumPeerConfig();
		if (args.length == 1) {
			config.parse(args[0]);
		}

		//启动定时清理任务
		DatadirCleanupManager purgeMgr = new DatadirCleanupManager(
				config.getDataDir(), config.getDataLogDir(),
				config.getSnapRetainCount(), config.getPurgeInterval());
		purgeMgr.start();

		if (args.length == 1 && config.servers.size() > 0) {
			//配置存在，且配置的服务器大于0
			runFromConfig(config);
		} else {
			//配置长度小于0在或者配置的服务器为0，走单机模式
			LOG.warn("Either no config or no quorum defined in config, running "
					+ " in standalone mode");
			// there is only server in the quorum -- run as standalone
			ZooKeeperServerMain.main(args);
		}
	}
    ...
```

* 1. 解析zoo.cfg配置文件填充QuorumPeerConfig对象
* 2. 创建DatadirCleanupManager对象，定期清理历史任务，默认情况下不清理。
* 3. 如果没有配置配置文件或者config.servers.size()=0的情况下，启动单机版的ZooKeeper, 否则创建QuorumPeer(法定成员)对象，启动QuorumPeer线程，然后join该线程。

其中第3步中：
```Java
	ServerCnxnFactory cnxnFactory = ServerCnxnFactory.createFactory();
			cnxnFactory.configure(config.getClientPortAddress(),
					config.getMaxClientCnxns());
	//提供一个服务连接事务处理器(Server Connection Transaction)处理客户端请求
	//可以通过启动参数来指定使用哪种事务处理器参数为——zookeeper.serverCnxnFactory
	//默认使用NIOServerCnxnFactory，还提供了NettyServerCnxnFactory备选
	//默认客户端连接数为60
	//NIOServerCnxnFactory.java
	...
	public void configure(InetSocketAddress addr, int maxcc) throws IOException {
		configureSaslLogin();//启动之前配置登陆的认证协议，Simple Authentication and Security Layer(SASL)

		//配置注册线程，NIO方式
		...
	}
	...

```
QuorumPeer类：管理法定成员协议，服务器有4种状态：

1. 寻找状态(LOOKING)；每台服务器将会开始选举Leader, 初始化时都假定自身是Leader。
2. 跟随者状态(FOLLOWING)；该服务器将会从Leader同步数据，复制事务。
3. 领导者状态(LEADING)；该服务器处理请求，然后将其派发给Follower，主要的Follower必须要在接受该请求前记录日志。
4. 观察者状态(OBSERVING)(后面会讲到)。

该类会设置一个数据报的Socket来接收来自目前Leader的响应，响应由以下形式构成：

* int xid;
* long myid;
* long leader_id;
* long leader_zxid;

Leader请求只由xid构成。
```Java
	///QuorumPeerMain.java
	...
	quorumPeer.start();
	...

	QuorumPeer.java
	...
	@Override
	public synchronized void start() {
		loadDataBase();//载入snapshot数据
		cnxnFactory.start();//客户端连接开始接受请求
		startLeaderElection();//开始执行Leader选举，已经过时，可以不用关注了
		super.start();
	}


	...
	@Override
	public void run() {
		setName("QuorumPeer" + "[myid=" + getId() + "]"
				+ cnxnFactory.getLocalAddress());

		//1、注册MBean，代码省略...
		
		//2、主循环，根据Peer的状态来决定要做的事
		/*
			 * Main loop
			 */
			while (running) {
				switch(getPeerStart()){
					case LOOKING://(1) 查找状态，进行Leader选举
					case OBSERVING://(2) 观察状态
					case FOLLOWING://(3) 跟随者状态
					case LEADING://(4) 领导者状态
				}
			}

```
