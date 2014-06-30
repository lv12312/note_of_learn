##ZooKeeper源码浅析

###1、脚本

先看启动脚本，和windows的启动脚本相比，linux下的显然比较强大。

windows：

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

* 1. 解析zoo.cfg配置文件填充QuorumPeerConfig类
* 2. 创建DatadirCleanupManager类，定期清理历史任务，默认情况下不清理。
* 3. 如果没有配置配置文件或者config.servers.size()=0的情况下，启动单机版的ZooKeeper, 否则创建QuorumPeer类，启动该QuorumPeer线程，然后等待。