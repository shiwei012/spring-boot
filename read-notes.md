# spring boot 源码阅读
### 0X00
多年阅读源码半途而废的我又开始阅读 spring boot 的源码，这次阅读的版本是v1.5.2.RELEASE
其实阅读源码没什么困难的，主要先要会使用它，然后通过 git tag 找到相应版本，然后想用它，
中间可以借助优秀的ide，比如：idea，开启调试模式定位相应的函数调用链。
这个人给我的启发很大，其实没有什么需要畏惧的，这是一件很快乐的事
https://raw.githubusercontent.com/wangshunping/read_requests/
### 0X01
spring boot 已经为我们写好了一个main入口函数，位于SpringApplication类中
需要完成两个关键性步骤，类的初始化initialize和调用run函数即可启动spring boot
初始化做了两件重要的事情
setInitializers 
    以ApplicationContextInitializer这个接口为key查找相关的实现类，最后有6个实例
setListeners
    以ApplicationListener这个接口为key查找相关的实现类，最后有10个实例
以上两个函数通过读取spring-boot模块spring-boot-autoconfigure模块resource中的META-INF/spring.factories配置文件，
查找到实现类以后都使用classloader通过反射的方式实例化所有对象

spring boot包中的对象实例化函数 一个轮子get
```java
private <T> Collection<? extends T> getSpringFactoriesInstances(Class<T> type,
			Class<?>[] parameterTypes, Object... args) {
    ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
    // Use names and ensure unique to protect against duplicates
    Set<String> names = new LinkedHashSet<String>(
            SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
            classLoader, args, names);
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}
```
当然上面写的只是重要的部分，其中还有什么web环境的检测之类的，都是细节，通过函数名去查找自己再看看就好了
阅读源码我选择了一个比较慢的速度，对于自己没看懂的地方，通过google关键字和自己运行都弄懂了，其实代码就和
自己写的差不多，只是做了很多封装。
通过拆解可以发现很多轮子，以后自己写代码需要用到就不用自己去google找轮子了
### 0X02
现在就可以直接看run函数了

```java
/**
	 * Run the Spring application, creating and refreshing a new
	 * {@link ApplicationContext}.
	 * @param args the application arguments (usually passed from a Java main method)
	 * @return a running {@link ApplicationContext}
	 */
	public ConfigurableApplicationContext run(String... args) {
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		FailureAnalyzers analyzers = null;
		configureHeadlessProperty();
		SpringApplicationRunListeners listeners = getRunListeners(args);
		listeners.starting();
		try {
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(
					args);
			ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);
			Banner printedBanner = printBanner(environment);
			context = createApplicationContext();
			analyzers = new FailureAnalyzers(context);
			prepareContext(context, environment, listeners, applicationArguments,
					printedBanner);
			refreshContext(context);
			afterRefresh(context, applicationArguments);
			listeners.finished(context, null);
			stopWatch.stop();
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass)
						.logStarted(getApplicationLog(), stopWatch);
			}
			return context;
		}
		catch (Throwable ex) {
			handleRunFailure(context, listeners, analyzers, ex);
			throw new IllegalStateException(ex);
		}
	}
```

一开始就有一个轮子
StopWatch 停表
Simple stop watch, allowing for timing of a number of tasks,
exposing total running time and running time for each named task.
感觉像是百米跑步用的秒表

configureHeadlessProperty
这玩意就是配置Headless 模式，就是告诉系统我是服务器不需要显示器键盘鼠标，或许可以提高性能吧

listeners.starting();
这行代码很重要（我觉得应该划起来，考试要考~）
这个函数实现了SpringApplicationRunListener接口，这是外国人写的代码吗？直译就是跑监听者的接口。好吧，其实他就一个类实现了他
EventPublishingRunListener事件发布跑监听者类，很溜有木有？看到这里我就想源码就是在一层一层的封装，类越来越多的过程
starting其实调用了这么一段代码
```java
this.initialMulticaster
				.multicastEvent(new ApplicationStartedEvent(this.application, this.args));
```
广播事件，multicast是多播很准确的翻译（毕竟6级420）
这个人广播了一个ApplicationStartedEvent的事件，其实通过看这个类的所有父类你会觉得很坑
首先，
ApplicationStartedEvent这个类调用了自己的父类ApplicationStartingEvent
```java
super(application, args);
```
然后，
ApplicationStartingEvent类和自己的儿子做了一样的事情调用了这个父类
SpringApplicationEvent
然后，
SpringApplicationEvent这个类做的事情多了一点，application参数传给了父类，args自己写了一个getter setter
然后，
这个人ApplicationEvent调用了自己的父类，加了一个timestamp
然后，
最终的祖先EventObject，因为是祖先自己接收的是Object source，呵呵。。。
这就是event的整个继承，很溜，唯一的解释就是他们是通过类名表示自己是什么事件的，所以边看边思考是一个好习惯，看了3分懂了7分！
SimpleApplicationEventMulticaster.multicastEvent是怎么广播的？
这个类和类名一样做得真的很简单，也是很多任务执行框架的典范，比如说我写的，哈哈
可以设置线程池，设置错误处理函数，如果你设置了他就会调，没有设置他就不会调，spring boot启动过程我是没就到有设置线程池的

现在就是找到之前设置的10个监听者谁注册了ApplicationStartedEvent事件
调用的ApplicationListener这个接口的方法onApplicationEvent。
我决定先用调试模式一个一个进去每个类的事件都是什么（其实应该有文档写了的），这个方法最重要的是在运行代码的下一行打上断点（教训get）

首先，不需要看十个，因为有个这个函数很有用
getApplicationListeners，可以根据传入的eventtype路由到相关的类去处理，我的天，路由器就是这样。。
解析之后只有一下四个：
class: ConfigFileApplicationListener events: ApplicationEnvironmentPreparedEvent, ApplicationPreparedEvent
class: LoggingApplicationListener events: ApplicationStartingEvent, ApplicationEnvironmentPreparedEvent, ApplicationPreparedEvent, ContextClosedEvent, ApplicationFailedEvent
class: DelegatingApplicationListener events: ApplicationEnvironmentPreparedEvent 
这个类有点特殊因为他自己还会进行一次广播
```java
List<ApplicationListener<ApplicationEvent>> delegates = getListeners(
                ((ApplicationEnvironmentPreparedEvent) event).getEnvironment());
if (delegates.isEmpty()) {
    return;
}
this.multicaster = new SimpleApplicationEventMulticaster();
for (ApplicationListener<ApplicationEvent> listener : delegates) {
    this.multicaster.addApplicationListener(listener);
}
```
也就是说event自己实现的时候可能会有不同的参数，其实你想想自己怎么写任务执行器的时候就知道了，我一般会用map
有点受到脚本语言的影响，java一般喜欢用java bean
class: LiquibaseServiceLocatorApplicationListener events: ApplicationEnvironmentPreparedEvent, ApplicationPreparedEvent
这个类也很奇怪的实现方式
```java
ClassUtils.isPresent("liquibase.servicelocator.ServiceLocator", null)
```
ClassUtils是spring utils的轮子，liquibase.servicelocator.ServiceLocator玩意是个什么鬼


### 0X03
这边看完了listener.staring()
下面这个非常重要，启动tomcat容器，也就是说自己程序以后想装个b搞个web容器，这段代码可以来一发
```java
private ConfigurableEnvironment getOrCreateEnvironment() {
		if (this.environment != null) {
			return this.environment;
		}
		if (this.webEnvironment) {
			return new StandardServletEnvironment();
		}
		return new StandardEnvironment();
	}
```
this.webEnvironment是等于true的因为前面initialize做了一件这样的事deduceWebEnvironment()演绎web环境（这才是我认识的外国人写的代码嘛）
怎么演绎的，额。。。就是这样
```java
private static final String[] WEB_ENVIRONMENT_CLASSES = { "javax.servlet.Servlet",
			"org.springframework.web.context.ConfigurableWebApplicationContext" };
for (String className : WEB_ENVIRONMENT_CLASSES) {
    if (!ClassUtils.isPresent(className, null)) {
        return false;
    }
}
```
看见这两个类了吗？就是看这两个类我能不能找到，哇~，这个起名字说明是很讲究的，后来我写的类比如执行任务的我叫TaskGreedy任务贪婪者，顿时很高大上

好的，下面是environmentPrepared这个event了
```java
@Override
	public void environmentPrepared(ConfigurableEnvironment environment) {
		this.initialMulticaster.multicastEvent(new ApplicationEnvironmentPreparedEvent(
				this.application, this.args, environment));
	}
```
这里就可以总结一下大师都是怎么写代码的了，SpringApplication里面的成员变量基本是整个生命周期需要用的，event可以理解为task，task的参数
会把所有的成员变量都传过去，咦？和我很像啊！！所以整个spring boot框架添加代码也是非常容易的因为这和你写web controller差不多，只要在spring.factories
中去添加你关注的event自己再写一个handler就好了，异常简单

嗯，先看一下这边这个event的名字ApplicationEnvironmentPreparedEvent，注册这个event的类有7个。。
0 = {ConfigFileApplicationListener@1278} 
1 = {AnsiOutputApplicationListener@2084} 
2 = {LoggingApplicationListener@2289} 
3 = {BackgroundPreinitializer@2290} 
4 = {ClasspathLoggingApplicationListener@2291} 
5 = {DelegatingApplicationListener@2292} 
6 = {FileEncodingApplicationListener@2293} 
第0个 ConfigFileApplicationListener这个类之前就看过了他会handle这个event
这里需要用到这个接口EnvironmentPostProcessor，记住spring boot，他是打死都不会用new的，同样使用spring.factories文件中配置的类名去找handler实例化
这里面处理方法是找到自己的兄弟们一起来处理。。。于是有了下面这些。。
第一个是他自己ConfigFileApplicationListener
这里面又发现一个轮子，就是resolvePlaceholder，比如${spring.application.json:${SPRING_APPLICATION_JSON:}}这个东西他一个通过get属性拼接成你需要的value
最后没处理什么因为上面这个属性是空的
第二个是CloudFoundryVcapEnvironmentPostProcessor这个是处理云平台的，6了，不过之前实例化的tomcat并不是云，看来云环境可以传递args参数设置，以后google一下
后面两个也差不多，反正配置参数都是空没做什么事，估计如果在application.properties里面写了配置会触发相关代码的
第1个 AnsiOutputApplicationListener
```java
@Override
	public void onApplicationEvent(ApplicationEnvironmentPreparedEvent event) {
		RelaxedPropertyResolver resolver = new RelaxedPropertyResolver(
				event.getEnvironment(), "spring.output.ansi.");
		if (resolver.containsProperty("enabled")) {
			String enabled = resolver.getProperty("enabled");
			AnsiOutput.setEnabled(Enum.valueOf(Enabled.class, enabled.toUpperCase()));
		}

		if (resolver.containsProperty("console-available")) {
			AnsiOutput.setConsoleAvailable(
					resolver.getProperty("console-available", Boolean.class));
		}
	}
```
很明显这个类就handler这一个event，应该是自己添加properties然后自己处理的典范
第2个 LoggingApplicationListener 这个我只想说可以读这个类文档，写得很明白，其实还是读取属性触发配置





class: ConfigFileApplicationListener events: ApplicationEnvironmentPreparedEvent, ApplicationPreparedEvent
class: ConfigFileApplicationListener events: ApplicationEnvironmentPreparedEvent, ApplicationPreparedEvent
class: ConfigFileApplicationListener events: ApplicationEnvironmentPreparedEvent, ApplicationPreparedEvent
class: ConfigFileApplicationListener events: ApplicationEnvironmentPreparedEvent, ApplicationPreparedEvent
class: ConfigFileApplicationListener events: ApplicationEnvironmentPreparedEvent, ApplicationPreparedEvent
class: ConfigFileApplicationListener events: ApplicationEnvironmentPreparedEvent, ApplicationPreparedEvent


