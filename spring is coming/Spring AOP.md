## Spring AOP ##

###1. AOP基础 ###

**AOP**是**Aspect Oriented Programming**的简称，编程语言的终极目的就是能够更自然的、更灵活的方式模拟现实世界。AOP做为OOP的一个很好的补充。按照软件重构的思想，代码重复书写对于程序员来讲是一件非常痛苦的事情，这样的代码需要提取出来，比如：打印日志、性能检测、权限校验等。

AOP有点类似**“雁过拔毛”**的感觉

AOP术语

* Joinpoint（连接点）：一个类或一段程序拥有具有边界性质的特定点，这些代码中的特定点就称之为“连接点”。Spring只支持方法的连接点，只有在方法调用前、调用后，方法抛出异常的时候织入增强。
* PointCut(切点）：每个类可能有多个连接点，但是如何定位到连接点呢？按照数据库的概念来理解再合适不过了，连接点表示数据库记录，切点表示查询条件。
* Advice(增强)：增强是织入到目标类连接点上的一段程序代码。
* Target(目标对象)：增强逻辑的织入目标类。
* Introduction(引介)：引介是一种特殊的增强，为类添加一些属性和方法。可以让业务类动态添加一些接口实现。听起来非常棒吧？
* Weaving(织入)：织入是将增强添加到目标类连接点上的过程，像个织布机。有三种织入方式：
	* 编译期织入，需要使用特殊的Java编译器。
	* 类装载期织入，需要使用特殊的类装载器。
	* 动态代理织入，在运行期为目标类添加增强生成子类的方式。

Spring采用动态代理，AspectJ采用编译器织入和类装载期织入。

* Proxy(代理)：一个类被AOP织入增强，就产生了一个结果类，它是融合了原类和增强类的代理类。
* Aspect(切面)：切面由切点和增强(引介)组成，它既包括了横切逻辑的定义，也包括了连接点的定义。


###2.创建增强类###

####2.1增强类型####
* 前置增强 `org.springframework.aop.BeforeAdvice` 接口代表前置增强，Spring只支持方法级别的强制,`MethodBeforeAdvice`是目前可用的前置增强。
* 后置增强 `org.springframework.aop.AfterReturningAdvice`后置增强接口。
* 环绕增强 `org.aopalliance.intercept.MethodInterceptor`,表示在目标方法前后实施增强。
* 异常抛出增强 `org.springframework.aop.ThrowsAdvice`,抛出异常的时候增强
* 引介增强 `org.springframework.aop.IntroductionInterceptor` 代表引介增强，表示在目标类中添加一些新的方法和属性。

这里注意一点：
ProxyFactory实现有`JdkDynamicAopProxy`和`Cglib2AopProxy`

* 对于 *JdkDynamicAopProxy* 创建代理时速度快，但是由于使用反射机制，运行慢
* 对于 *Cglib2AopProxy* 创建代理时速度慢，但是由于使用Cglib字节码机制，运行速度快


####2.2创建切面####

增强被织入到目标类的所有方法中，假设我们希望有选择地织入到目标类某些特定方法中，就需要使用切点进行目标连接点的定位。Spring通过 `org.springframework.aop.Pointcut` 接口来描述连接点，通过ClassFilter和MethodMatcher构成的，通过ClassFilter来定位到某些类的能力，通过MethodMatcher来定位到方法。

切点有六种：

* 静态方法切点，默认匹配所有的类 `org.springframework.aop.support.StaticMethodMatcherPointcut` 
* 动态方法切点 `org.springframework.aop.support.DynamicMethodMatcherPointcut` 动态方法切点的抽象基类，默认情况匹配所有的类。
* 注解切点 `org.springframework.aop.support.annotation.AnnotationMatchingPointcut
` JDK5.0注解标签定义的切点
* 表达式切点 `org.springframework.aop.support.ExpressionPointcut` 
* 流程切点 `org.springframework.aop.support.ControlFlowPointcut`
* 复合切点 `org.springframework.aop.support.ComposablePointcut`

切面：一个切面同时包括横切代码和连接点信息

####2.2 自动创建代理####
ProxyFactoryBean创建织入切面的代理，每一个都要配置，太繁琐了，如果开发大型系统的话，这样显然是不科学的。Spring提供了自动动态代理的能力。
常见的就是

#####BeanNameAutoProxyCreator#####
通过代码注释可以知是通过Bean的名称来定位到Bean,然后设置拦截器，这样就自动拦截了。

自动代理创建器通过name列表来定位bean,XML配置示例

	<bean id="xxxService" class="com.xxx.xxx.XxxService" />
	<!-- 拦截器 -->
	<bean class="org.springframework.aop.framework.autoproxy.BeanNameAutoProxyCreator">
		<property name="interceptorNames">
			<list>
				<value>xxxInterceptor0</value>
				<value>xxxInterceptor1</value>
			</list>
		</property>
		<property name="beanNames">
			<value>xxxService</value>
		</property>
	</bean>
	<bean id="xxxInterceptor0" class="com.xxx.XxxInterceptor0"/>
	<bean id="xxxInterceptor1" class="com.xxx.XxxInterceptor1">
	</bean>


#####DefaultAdvisorAutoProxyCreator#####

Advisor是切点和增强的结合体，本身包含了足够的信息：要织入什么？织入在哪里？
这个作用是可以扫描容器中的Advisor，并请将Advisor自动织入到匹配目标Bean中，即为匹配目标Bean自动创建代理。