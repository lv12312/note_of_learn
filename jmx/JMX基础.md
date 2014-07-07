## JMX基础

###1、JMX基本术语

* **Manageable Resource** 可管控的资源，由Java封装的可管控的资源，如打印机、外部存储。
* **MBean** 符合JMX规范的Java类，管理应用通过访问MBean访问属性和调用操作。本书关注三种类型的MBean:标准的、动态的和模型的MBean。每种类型的MBean对于特殊的资源都有特殊的优势。
* **MBean Server** 管理MBean的Java类，是JMX管理环境的核心。MBean Server暴露注册的MBean给外部。MBean Server同样提供查询MBean的方法。
* **JMX Agent** JMX代理是一个Java进程，提供了一组管理MBean的服务，是MBean服务器的**容器**，JMX代理提供了创建MBean的关系，动态加载类和简单的监控和定时功能。JMX代理提供了多种协议的访问。
* **Protocol adapters and connectors**,协议适配器和连接器驻留在JMX 代理中，它们将代理暴露给管理应用和协议。例如SNMP 可以使用SNMP 适配器将自己映射到JMX 代理。另外，JMX 代理还可以有一个RMI 连接器，以备使用RMI 客户端进行管理。协议适配器由代理端的一个单独对象组成，而连接器则由代理端和客户端的对象一同组成。
一个代理可以有任意数量的适配器和连接器，从而可以使用工具或已存在的协议和应用来连接代理。这样不仅仅代理具有可扩展性，而且有了一种跨网络的分布式代理机制。

* **Management application**, 管理应用是任何用于连接到JMX代理接口的用户程序。
* **Notification**，通知是Java对象，由MBean和MBeanServer发送。其他MBean或者Agent可以注册一个监听器来接收通知。
* **Instrumentation**，指令是通过MBean 暴露的可管理资源的一个处理过程。指令的开发可以伴随应用的开发，或者开发者可以使用正在使用的系统的API 创建MBean 。随后你会发现，通过选择合适的MBean 类型，可以很容易的只暴露应用和资源的一部分。

###2、JMX应用架构

JMX 三层架构

> 分布层Distributed layer,包含使管理系统和 JMX 代理通信的组件
> 
> 代理层Agent layer,包括代理和 MBean 服务器
> 
>指令层 Instrumentation layer,包括代表可管理资源的 MBean


###3、HelloWorld示例

	public class HelloAgent {
    private MBeanServer mbs = null;

    public void startAgent(){
        //1、创建MBean Server 和HTML adapter
        mbs = MBeanServerFactory.createMBeanServer("HelloAgent");
        HtmlAdaptorServer adaptorServer = new HtmlAdaptorServer();
        //2、创建MBean实例
        HelloWorld hw = new HelloWorld();
        ObjectName adapterName;
        ObjectName helloWorldName;
        try {
            //3、创建ObjectName实例，注册HelloWorld MBean
            helloWorldName = new ObjectName("HelloAgent:name=helloWorld1");
            mbs.registerMBean(hw,helloWorldName);
            //4、注册并允许HTML访问的 adapter MBean,
            adapterName = new ObjectName("HelloAgent:name=htmlAdapter,port=9092");
            adaptorServer.setPort(9092);
            mbs.registerMBean(adaptorServer,adapterName);
            adaptorServer.start();

        }catch (Exception e){
            e.printStackTrace();
        }
    }

    public static void main(String[] args) {
        System.out.println("HelloAgent is running!");
        HelloAgent agent = new HelloAgent();
        agent.startAgent();//启动agent
    }
	}

从代码就能看出adapter同样也是MBean，也需要注册，接受MBeanServer的管理。

###4、唯一标识MBean
注册的MBean的时候，需要使它和其他的MBean区分开来。要做到这一点，需要新建一个`javax.management.ObjectName`实例，用来唯一标识每一个注册在它里面的MBean。
每个MBean包括两个部分：

* 域(domain)——一般是MBean想注册的那个域。使用域将MBean隔离开来。
* (name,value)对，以及提供的MBean信息，可以提供诸如名字、端口、位置等信息。

如代码中写到的：
>HelloAgent:name=helloWorld1

Domain Name为HelloAgent
key=Value List为name=helloWorld1

注意：注册ObjectName时不允许重复。

###5、使用MBean通知

JMX通知是Java对象，通过它可以从MBean和代理向那些注册了接受通知的对象发送通知。对接受事件感兴趣的对象是通知监听器，是实现了`javax.management.NotificationListener`接口的类。

JMX提供了两种机制来为MBean提供监听器以注册来接受通知：

* 实现`javax.management.NotificationBroadcaster`接口
* 继承`javax.management.NotificationBroadcasterSupport`类

