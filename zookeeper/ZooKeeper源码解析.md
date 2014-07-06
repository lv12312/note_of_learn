##ZooKeeper源码浅析

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

启动过程如下：

* 1. 解析zoo.cfg配置文件填充QuorumPeerConfig对象
* 2. 创建DatadirCleanupManager对象，定期清理历史任务，默认情况下不清理。
* 3. 如果没有配置配置文件或者config.servers.size()=0的情况下，启动单机版的ZooKeeper, 否则创建QuorumPeer(法定成员)对象，启动QuorumPeer线程，然后join该线程。

其中第3步中：
	
	...
	ServerCnxnFactory cnxnFactory = ServerCnxnFactory.createFactory();
			cnxnFactory.configure(config.getClientPortAddress(),
					config.getMaxClientCnxns());
	//提供一个服务连接事务处理器(Server Connection Transaction)
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


QuorumPeer类：管理法定成员协议，服务器有4种状态：

1. 寻找状态(LOOKING)；进行Leader选举时的状态。
2. 跟随者状态(FOLLOWING)；
3. 领导者状态(LEADING)；
4. 观察者状态(OBSERVING)。

