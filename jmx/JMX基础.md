## JMX基础

###1、JMX基本术语

* Manageable Resource 被管控的资源，由Java封装的可管控的资源，如打印机、外部存储。
* MBean 符合JMX规范的Java类，管理应用通过访问MBean访问属性和调用操作。本书关注三种类型的MBean:标准的、动态的和模型的MBean。每种类型的MBean对于特殊的资源都有特殊的优势。
* MBean Server 管理MBean的Java类，是JMX管理环境的核心。MBean Server暴露注册的MBean给外部。MBean Server同样提供查询MBean的方法。
* JMX代理 JMX代理是一个Java进程，提供了一组管理MBean的服务，是MBean服务器的容器，JMX代理提供了创建MBean的关系，动态加载类和简单的监控和定时功能。JMX代理提供了多种协议的访问。