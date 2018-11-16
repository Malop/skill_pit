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


	
	
	
