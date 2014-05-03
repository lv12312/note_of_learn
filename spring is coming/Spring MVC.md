##SpringMVC原理分析##

###1. 过程概述###

谈到web，可以先看看标准的Java Web项目是如何的？
首先看一个标准的JavaWeb项目的web.xml配置，
	
	<?xml version="1.0" encoding="UTF-8"?>
	<web-app version="2.4"
         xmlns="http://java.sun.com/xml/ns/j2ee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://java.sun.com/xml/ns/j2ee http://java.sun.com/xml/ns/j2ee/web-app_2_4.xsd">
	
	<!--配置上文参数-->
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath*:spring/applicationContext*.xml</param-value>
    </context-param>

	<!--过滤器-->
    <filter>
        <filter-name>compositeFilter</filter-name>
        <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
		<init-param>
			<param-name>targetFilterLifecycle</param-name>
			<param-value>true</param-value>
		</init-param>
    </filter>
    <filter-mapping>
        <filter-name>compositeFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>

	<!--监听器-->	
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>


	<!--这里才是真正标准的Servlet配置-->
	<servlet>
        <servlet-name>ShoppingServlet</servlet-name>
        <servlet-class>com.myTest.ShoppingServlet</servlet-class>
	</servlet>
	<servlet-mapping>
        <servlet-name>ShoppingServlet</servlet-name>
        <url-pattern>/shop/ShoppingServlet</url-pattern>
	</servlet-mapping>

    <error-page>
        <error-code>401</error-code>
        <location>/common/401.jsp</location>
    </error-page>
    <error-page>
        <error-code>403</error-code>
        <location>/common/403.jsp</location>
    </error-page>
    <error-page>
        <error-code>500</error-code>
        <location>/common/500.jsp</location>
    </error-page>
    <error-page>
        <exception-type>java.lang.Throwable</exception-type>
        <location>/common/500.jsp</location>
    </error-page>
	</web-app>


讲述SpringMVC肯定不会少了`org.springframework.web.servlet.DispatcherServlet`
根据源代码讲述的，这个类是HTTP请求handler/controller中心调度器，将Web请求派发给注册好的处理器进行处理。Servlet是非常灵活的：可以在各种流程下面使用，通过安装合适的适配器。提供了以下区别于其他请求驱动的MVC框架的诸多功能：

![SpringMVC图片](https://raw.githubusercontent.com/lv12312/note_of_learn/master/spring%20is%20coming/springmvc.JPG)

* 基于JavaBean配置机制
* 可以使用任何基于 `HandlerMapping` 的实现，默认的是 `BeanNameUrlHandlerMapping` 和 `DefaultAnnotationHandlerMapping`，HandlerMapping对象可以作为Bean在Servlet应用上下文中注册，实现了HandlerMapping接口
* 等等。。。


* （1）整个过程从客户端发出一个HTTP请求，然后通过Apache或者Nginx对地址的解析，然后再通过Web容器解析，如果匹配DispatcherServlet的请求映射路径(在web.xml指定)，Web容器就将该请求转交给DispatcherServlet处理。
* （2）DispatcherServlet接收到这个请求后，根据请求的信息(包括URL、HTTP方法、请求报文头、请求参数、Cookie等)以及HandlerMapping找到处理器，Spring任何一个Object都可以成为请求处理器。
* （3）当 DispatcherServlet根据 HandlerMapping得到对应当前请求的Handler后，通过HandlerAdapter对Handler进行封装，再以统一的适配器接口调用Handler。HandlerAdapter是Spring MVC的框架级接口，适配器，统一的接口对各种Handler方法进行调用。
* （4）处理完业务逻辑后返回一个ModelAndView给DispatcherServlet，包含了视图逻辑名称和真实视图对象的解析工作。
* （5）ModelAndView中包含的是“逻辑视图名”而非真正的视图对象，DispatcherServlet借由ViewResolver完成逻辑视图名到真实视图对象的解析工作。
* （6）得到真实的视图对象View后，DispatcherServlet就是用这个View对象对ModelAndView中的模型数据进行视图渲染。
* （7）最终客户端得到的响应消息，可能是一个普通的HTML页面，也可能是一个XML或者JSON字符串等等。

（未完待续）
