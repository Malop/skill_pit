# skill_pit
## 项目遇到的一些问题笔记

### 1、问题描述:SpringMVC中前台日期格式字段传到后台controller后，后台无法转化为Date
  原因：SpringMVC前台的日期格式字段都是以json格式传输，到后台后都是String，model无法将String转化为Date
  * 解决方案一：在对应的Controller中增加如下代码
  ```java
  @InitBinder
  Public void initBinder(WebdataBinder binder){
    DateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd");
    dateFormat.setLenient(true);
    binder.registerCustomEditor(Date.class,New CustomDateEditor(dateFormat,true));
  }
  ```
  * 解决方案二：在字段对应的model中对该字段增加相应转化注解
  ```java
  @Column(name = "ENDDATE")
  @DateTimeFormat(pattern = "yyyy-MM-dd")
  @JsonFormat(pattern = "yyyy-MM-dd")
  private Date enddate;
  ```
### 2、Spring 无法注入依赖包的实例
	原因：对于容器而言，如@Autowired@component等注解都是针对项目工程下资源告诉容器工程中的实例可以访问并加载进入容器，但第三方的包实例需用@Import注解将这些包实例数据导入


### 3、在设计全局性唯一ID时产生的问题
	描述：设计系统全局唯一性id功能时，尝试了用db2的sequence自增的方式来做(当然也可以用redis的自增)，但遇到一个问题，在id达到最大时(设置为int的最大值，或者是id=年+月+日+00000+最后三位用sequence自增获得，那在达到999时需要做回滚)，需要做id的回滚，但是在并发情况下，有可能出现id重复的情况，然后采用一个取巧的方式，增加缓冲区的方式来做回滚存取
```java 
/**
*注:该方法主要是栋神想出来的
*在不用缓冲区情况下，req1=max,req2=max+1,req1重置后为1，这时req3和req4分别获取下一个序列号，req3=2，req4=3，而同一时间req2重置后为2，req5在req2的基础上获取的序列号也为3，这样就重复了，而加了缓冲区后则会变成req1=1001,req2=2001,req3=1002,req5=2002，这样并发下req3跟req5有1000的缓冲就不会重复
**/
public static void getInt(){
	long seq = RedisUtil.incr(tableName.getName());
    //long seq = db2sequenceUtil.incr();
    //序列号达到最大需做回滚
	if(seq > Integer.MAX_VALUE){
        //并发下对不同线程增加1000的缓存处理，这样就不会出现多线程取到同一个值
		int ret = (int) ((seq)%Integer.MAX_VALUE)*1000+1;
		set(tableName,ret);
		return ret;
	}else{
		return (int)seq;
	}
}
``` 
### 4、采用Guava的Optional处理Java集合中的null值
	描述：Java集合中的null值表述意思不清晰，可表述为空，也可表述为不存在(异常)，google的guava包提供了Optional类来解决null值问题
	常用静态方法:
	1>.Optional.of(T)：获得一个Optional对象，其内部包含了一个非null的T数据类型实例，若T=null，则立刻报错。
	2>.Optional.absent()：获得一个Optional对象，其内部包含了空值
	3>.Optional.fromNullable(T)：将一个T的实例转换为Optional对象，T的实例可以不为空，也可以为空[Optional.fromNullable(null)，和Optional.absent()等价。
	a常用实例方法:
	1>. boolean isPresent()：如果Optional包含的T实例不为null，则返回true；若T实例为null，返回false
	2>. T get()：返回Optional包含的T实例，该T实例必须不为空；否则，对包含null的Optional实例调用get()会抛出一个IllegalStateException异常
	3>. T or(T)：若Optional实例中包含了传入的T的相同实例，返回Optional包含的该T实例，否则返回输入的T实例作为默认值
	4>. T orNull()：返回Optional实例中包含的非空T实例，如果Optional中包含的是空值，返回null，逆操作是fromNullable()
	5>. Set<T> asSet()：返回一个不可修改的Set，该Set中包含Optional实例中包含的所有非空存在的T实例，且在该Set中，每个T实例都是单态，如果Optional中没有非空存在的T实例，返回的将是一个空的不可修改的Set。

### 5、redis连接池性能优化
	问题描述：监控系统发现在存储数据的方法上，对redis进行了setnx()和get()操作，两个操作总共消耗约1.5ms，然后存储数据的方法平均耗时有3.74ms，多出的2ms无解
	实践：监控系统的原理是对redis的get方法做代理，从发送命令到同步读取数据完成，所以io时间不再监控范围内。用jconcle模拟线上排查，发现改方法在除了setnx跟get后，在获取连接的时候消耗了大量时间。
	解决方案:修改redis连接池配置：
	* 设置mindle=1，保证至少有一个可用的连接，
	* 设置testOnBorrow=false，testOnReturn=false，减少不必要的IO请求
	* 设置testWhileIdle=true，保证连接的可用性，定期检查空闲连接的状态
	在jedisFactory的源码可以看出在获取连接池连接的时候会发送ping指令到redis集群，浪费io不必要的时间，所以需要配置testOnBorrow与testOnResturn属性为false

### 6、ZK作为一个注册中心有什么优缺点？说一说CAP？
ZK对强一致性的要求较高
	
	
### 7、设计模式，策略模式和状态模式的区别

### 8、Spring中bean的依赖注入的实现

## 9、线程池的工作原理，线程池如何知道线程是空闲的？

### 10、mysql连接池满，无法获取链接问题
	问题描述：线上报警有台机器一直抛异常，查询日志发现是mysql数据库连接池一直获取不到，提示连接池满
	排查：先进行单台机器重启应急保证系统运行。review获取数据库连接池的地方，发现是某处代码获取数据库连接后未做资源释放，基本定位问题，准备在测试环境做复现验证，多次执行后却发现数据库连接池为增大，很疑惑。。。再review进底层封装的代码，发现【每次获取连接并不是直接向连接池获取连接，而是从自己封装的事务管理器获取连接，这就导致了：在一大段业务代码中，即使存在个别地方不关闭数据库资源，但只要存在任何一个地方正确使用了这段代码，都会最终把此连接释放】，这也就解释了为何测试环境无法复现，进一步验证：在读取数据库后，抛出异常，不断执行，发现链接确实暴增
	解决方案：修改业务代码，在每处获取数据库链接调用的地方都进行try{}catch{}finnaly{}在最后的finnaly里保证释放资源；或者修改底层源码逻辑，保证每次调用完后都会释放链接资源

### 11、页面静态化方案：缓存方案。ESI/CSI/SSI

### 12、myBatis懒加载问题
	问题描述:myBatis配置懒加载之后，在test时发现打印出来实体时懒加载并未生效，debug加断点也捕捉不到
	排查：查看源码发现，myBatis在懒加载时默认对触发的hashcode,requals,clone,toString方法就会执行懒加载，而test断点debug时，是新起一个线程去加载代码，在打印实体类时，调用了hashcode和toString方法，这时就触发了懒加载
	解决方案:配置懒加载出发方法configuration.setLazyLoadTriggerMethods(new HashSet<String>());
	还有就是用mybatis的默认debug级别日志，查看sql语句来判断懒加载是否生效
