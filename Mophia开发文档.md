## Morphia 开发文档（中文版） ##

###1、简介###
Morphia作为MongoDB轻量级ORM框架，提供了类型安全的、支持运行期校验的Query API。Morphia使用注解来操作MongoDB。如果有JPA的经验，Morphia会让你更爽的。
###2、特性###
* Lifecycle Method/Event支持
* 能和依赖注入框架配合使用，如Guice，Spring
* 丰富的扩展点（注解，转换器，映射行为，日志等等）
* 不会存储null/空值(默认)
* GWT支持(实体都是POJO)--(GWT忽略注解）
* 允许原生转换的高级映射器，`void toObject(DBOjbect)` 或者 `DBObject fromObject(Object)`
###3、使用###
建议使用Maven

	<dependency>
			<groupId>org.mongodb.morphia</groupId>
			<artifactId>morphia</artifactId>
			<version>${morphia.version}</version>
	</dependency>

代码示例：

	@Entity("employees")
	class Employee {
	//自动生成，如果没有设置的话
  	@Id ObjectId id;

	//自动持久的类型
  	String firstName, lastName;

	//只有非空的值会被存储
  	Long salary = null;

 	//默认为@Embedded
  	Address address;

	//未原子载入的引用也可被保存
  	Key<Employee> manager;

	//引用被存储，也可自动载入，但是不会存入对象，只是引用他们，需要自己控制如何存储
  	@Reference List<Employee> underlings = new ArrayList<Employee>();

  	//在一个二进制属性中存储
  	@Serialized EncryptedReviews;

  	//字段被重命名
  	@Property("started") Date startDate;
 	@Property("left") Date endDate;

  	//字段可以通过使用索引来提高性能
  	@Indexed boolean active = false;

  	//字段可以被载入，但是不会保存
  	@NotSaved string readButNotStored;

  	//可忽略的字段，不会被存储和载入
  	@Transient int notStored;

  	//not @Transient, will be ignored by Serialization/GWT for example.
	//没有标记@Transient, 将会在序列化的时候忽略
  	transient boolean stored = true;
	
	//生命周期方法(Lifecycle方法) -- 载入之前/之后，持久之前/之后...
  	@PostLoad void postLoad(DBObject dbObj) { ... }
	}

	...

	Datastore ds = ...; //如：new Morphia(new Mongo()).createDatastore("hr")
	morphia.map(Employee.class);

	ds.save(new Employee("Mister", "GOD", null, 0));

	// get an employee without a manager
	Employee boss = ds.find(Employee.class).field("manager").equal(null).get();

	Key<Employee> scottsKey =
  	ds.save(new Employee("Scott", "Hernandez", ds.getKey(boss), 150**1000));

	//add Scott as an employee of his manager
	UpdateResults<Employee> res =
  	ds.update(
    boss,
    ds.createUpdateOperations(Employee.class).add("underlings", scottsKey)
  	);

	// get Scott's boss; the same as the one above.
	Employee scottsBoss =
  	ds.find(Employee.class).filter("underlings", scottsKey).get();

	for (Employee e : ds.find(Employee.class, "manager", boss))
   	print(e);

###4、初始化Morphia/Mongo###

	Morphia morphia = new Morphia();
	// mongo应该是单例的
	DataStore dataStore = morphia.createDatastore(mongo, dbName);
	// 在调用Morphia的map方法之前应该需要map这个class
	morphia.map(ProductEntity.class);
	// 创建索引
	dataStore.ensureIndexes();
	// 给@Entity注解过的实体创建capped Collections
	ds.ensureCaps();

###5、保存###

	ProductEntity insertEntity = new ProductEntity();
    datastore.save(insertEntity);


###6、查询###
	
	//查询首条记录
	ProductEntity single = datastore.find(ProductEntity.class).get();
    System.out.println(single);

	
###7、Datastore接口###
`Datastore`接口提供了类型安全的方法，用于在MongoDB中访问或者存储Java对象，提供了get/find/save/delete方法。

####7.1 Get方法####

Get方法返回和@Id相匹配的实例。这里只是将id值作为的条件的find()方法包装一下。每次都是返回实体，如果没有找到的话就是null。

	ProductEntity getEntity = datastore.get(insertEntity);

####7.2 Find方法####

Find方法是Query（createQuery()）轻量的包装器。作为包装器将会返回Query, 同样支持Iterable<T>和QueryResults接口。

	Datastore ds = ...

	//use in a loop
	for(Hotel hotel : ds.find(Hotel.class, "stars >", 3)) {
  	 print(hotel);
	}

	//get back as a list
	List<Hotel> hotels = ds.find(Hotel.class, "stars >", 3).asList();

	//sort the results
	List<Hotel> hotels = ds.find(Hotel.class, "stars >", 3).sort("-stars").asList();

	//get the first matching hotel, by querying with a limit(1)
	Hotel gsHotel = ds.find(Hotel.class, "name", "Grand Sierra").get();

	//same as
	Hotel gsHotel = ds.find(Hotel.class, "name =", "Grand Sierra").get();

可以使用“<,>,=,==,!=,<>,<=,>=,in,nin,all,size,exits”.
####7.3 Save方法####
使用很简单：

	ProductEntity insertEntity = new ProductEntity("Apple", "INST", "Hello", 45, new Reviews(
            "Lemon", "20140407"));
    datastore.save(insertEntity);
	insertEntity.getId();//能够直接fill这个entity

####7.4 Delete方法####
这个就不用解释了。

	Datastore ds = ...
	ds.delete(Hotel.class, "Grand Sierra Resort");
	//使用查询
	ds.delete(ds.createQuery(Hotel.class).filter("pendingDelete", true));
####7.5 FindAndDelete####
某些时候需要类似原子操作，查询之后就删除掉到的值(在返回的的时候)：

	Datastore ds = ...
	Hotel grandSierra = ds.findAndDelete(Hotel.class, "Grand Sierra Resort");

####7.6 Update方法####

过多的不解释，看代码：

	@Entity
	class User {
   		@Id
   		private ObjectId id;
   		private long lastLogin;
   		//... other members

   		private Query<User> queryToFindMe() {
      		return datastore.createQuery(User.class).field(Mapper.ID_KEY).equal(id);
   		}

   		public void loggedIn() {
      		long now = System.currentTimeMillis();
      		UpdateOperations<User> ops = datastore.createUpdateOperations(User.class).set("lastLogin", now);
      		ds.update(queryToFindMe(), ops);
      		lastLogin = now;
   		}
	}


####7.7 建立索引和Capped集合####


方法将会在Mophia中注册实体的的时候调用。该方法会异步创建索引和capped集合。另外一个方式就是在应用启动的时候执行，还可以通过管理接口，或者使用部署脚本。在应用设计的时候就可以将使用索引特性用上。
Note:如果索引已经建立了，该操作将不会执行。

	Morphia m = ...
	Datastore ds = ...

	m.map(MyEntity.class);
	ds.ensureIndexes(); //creates all defined with @Indexed
	ds.ensureCaps(); //creates all collections for @Entity(cap=@CappedAt(...))

详细的文档见：
[Morphia WIKI](https://github.com/mongodb/morphia/wiki)