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


