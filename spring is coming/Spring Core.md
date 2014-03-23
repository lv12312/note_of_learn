## Spring Core IoC容器中装配Bean

####1. XML方式的配置

通常的情况下都是使用XML来作为Spring的配置文件，也是项目中非常常见的，一下是个示例的XML文件：

	<?xml version="1.0" encoding="UTF-8"?>
	<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:p="http://www.springframework.org/schema/p"
	xsi:schemaLocation="http://www.springframework.org/schema/beans 
	http://www.springframework.org/schema/beans/spring-beans.xsd">

	<bean id="car" class="com.lemon.spring.test.Car"/>
	</beans>
使用Schema来描述的，通常包含名称空间和名称空间对应的schema文件，如上面所示，具体的xsd代表什么含义见文档
####2. Bean的基本配置
#####2.1 Bean的装配
	<bean id="car" class="com.lemon.spring.test.Car"/>
这就是最基本Bean,由bean id和类名构成。Bean的名称要满足XML的规范，Bean名称可以定义多个名称，但是通常情况下没有人会傻逼到取很多和各种奇怪的名字给自己挖坑。老老实实的按照类名的小写来作为Id，这样统一化了命名，也为后面的自动装配的命名养成了良好的命名习惯。

对于Bean的属性而言，最佳命名方式是第一个单词小写，如果碰到一些大写的缩略语，都要弄成小写的；比如USA，LV，一律写成usa,lv，这样做的目的就是为了防止在bean setter方法的时候造成不符合bean规范，还有boolean类型的变量，最好也是用setter和getter，不是用is来修饰。

#####2.2 构造方法注入和属性注入的比较
* 选择构造方法注入的理由：无需增加Bean中过多的setter方法，更好的封装变量，有些属性不希望外部改动
* 选择属性注入的理由：
	* 如果属性过多，构造方法会无比的庞大，而且增加一个属性之后，构造方法需要做相应的修改
	* 灵活性不够，如果希望参数拥有一个null值的话，这个属性就是有必要的
	* 需要用到继承的时候，这下就比较麻烦一点
	* 如果使用构造方法时，容易出现循环依赖的问题


#####2.3 FactoryBean的使用
在很多场景里面使用了FactoryBean,该接口中定义了三个方法：

* `T getObject()`:返回由FactoryBean创建的Bean实例，如果isSingleton()返回true，则该实例会放入Spring容器的单实例缓存池中。
* `boolean isSingleton()`：确定Bean的作用域是singleton还是prototype。
* `Class<?> getObjectType<>`:返回FactoryBean创建Bean的类型。 

#####2.4 Spring容器内部工作机制
`refresh()`方法启动

			// 1.准备上下文
			prepareRefresh();
			// 2.告诉子类刷新内部Bean工厂
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
			// 3.准备在上下文中使用Bean工厂
			prepareBeanFactory(beanFactory);

			try {
				// 4.允许提前运行子类中的上下文的bean工厂
				postProcessBeanFactory(beanFactory);

				// 5、调用作为Bean注册在上下文中的工厂后处理器
				invokeBeanFactoryPostProcessors(beanFactory);

				// 6、注册拦截bean生成的bean后处理器
				registerBeanPostProcessors(beanFactory);

				// 7、初始化上下文的消息源：初始化容器的国际化信息资源；
				initMessageSource();

				//8、初始化应用上下文事件广播器
				initApplicationEventMulticaster();

				// 9、初始化其他的特定上下文子类的特殊Bean
				onRefresh();

				// 10. 检查并注册事件监听器
				registerListeners();

				// 11.初始化所有的单例Bean（不包括懒加载的），初始化bean之后放入缓存中
				finishBeanFactoryInitialization(beanFactory);

				// 12.发布上下文刷新事件：创建上下文刷新事件，时间广播器负责将这些事件广播到每个注册的事件监听器中
				finishRefresh();
				}

				catch (BeansException ex) {
				// 抛出异常，销毁已经创建的单例来避免悬挂资源
				destroyBeans();

				// 重置active标记
				cancelRefresh(ex);
				throw ex;
			}

Spring从加载配置文件到创建出一个完整的Bean的作业流程以及参与的角色：

	Spring-config.xml-->Resource-->BeanDefinitionRegistry-->PropertyEditorRegistr
											|                             |
											|BeanFactoryProcessor处理后    |
											|                            \|/
										   \|/              PropertyEditorRegistry
								BeanDefinitionRegistry                     |
											|        |                     |
				 InstantiationStrategy实例化 |        BeanWrapper设置属性--->|
										    |          |设置属性            |
  										   \|/        \|/                 \|/
								   Bean实例(未设置属性)------------->Bean实例(已设置属性)
																		 |
											   BeanPostProcessor对Bean加工|
																		\|/
																Bean实例（准备完毕）