##  ApplicationContext 与 ResourceLoader
![](https://github.com/ALittleFang/java-reader/blob/master/Java面试/AbstractApplicationContext.png)
 ApplicationContext 继承了 ResourcePatternResolver ，当然就间接实现了 ResourceLoader 接口。所以，任何的 ApplicationContext 实现都可以看作是一个ResourceLoader 甚至 ResourcePatternResolver 。而这就是 ApplicationContext 支持Spring内统一资源加载策略的真相。

### ResourceLoader（统一资源定位器）
![](https://github.com/ALittleFang/java-reader/blob/master/Java面试/resourceLoader.png)
ResourceLoader 接口是资源查找定位策略的统一抽象，具体的资源查找定位策略则由相应的 ResourceLoader 实现类给出。

#### 资源地址表达式
+ classpath:前缀开头，如classpath:xxx.xml
+ /开头，如/WEB_INF/xxx.xml
+ 非/开头，如WEB_INF/xxx.xml
+ **url协议**，如file:/D:/xxx.xml

#### ResourceLoader实现类
##### DefaultResourceLoader（默认实现类）
资源查找处理逻辑如下：
1. 首先检查资源路径是否以 classpath: 前缀打头，如果是，则尝试构造 ClassPathResource 类型资源并返回。
2. 否则,
	- 尝试通过URL资源路径来定位资源，如果没有抛出 MalformedURLException，有则会构造 UrlResource 类型的资源并返回;
	- 如果还是无法根据资源路径定位指定的资源，则委派getResourceByPath(String)方法来定位，构造 ClassPathResource 类型的资源并返回.
	
##### FileSystemResourceLoader
继承自 DefaultResourceLoader ，但覆写了 getResourceByPath(String) 方法，使之从文件系统加载资源并以FileSystemResource 类型返回。

##### ResourcePatternResolver ——批量查找的 ResourceLoader
ResourcePatternResolver 是 ResourceLoader 的扩展， 其可以根据指定的资源路径匹配模式，每次返回多个 Resource 实例。同时还引入了一种新的协议前缀 classpath*:

## BeanFactory和FactoryBean的区别
+ **BeanFactory是IOC最基本的容器，负责生产和管理bean，它为其他具体的IOC容器提供了最基本的规范**，例如DefaultListableBeanFactory。XmlBeanFactory,ApplicationContext 等具体的容器都是实现了BeanFactory，再在其基础之上附加了其他的功能。
+ **FactoryBean是个工厂Bean，属于Spring Bean的一种，是一种特殊的Bean**。通过getBean(String BeanName)获取到的Bean对象并不是FactoryBean的实现类对象，而是这个实现类中的getObject()方法返回的对象。要想获取FactoryBean的实现类，就要getBean(&BeanName)，在BeanName之前加上&。

### FactoryBean的作用
FactoryBean 通常是用来创建比较复杂的bean，一般的bean 直接用xml配置即可，但如果一个bean的创建过程中涉及到很多其他的bean 和复杂的逻辑，用xml配置比较困难，这时可以考虑用FactoryBean。

## 基于注解的形式，是怎么实现的（答案暂定）
@Configuration、@Controller、@Service这些注解其实都是@Component的派生注解，我们看这些注解的代码会发现，都有@Component注解修饰。而spring通过metadata.hasMetaAnnotation()方法获取到这些注解包含@Component，所以都可以扫描到
# SpringMVC

### SpringleMVC的运行机制


## StringMVC与Struts2的区别
**Struts2是类级别的拦截，每次请求就会创建一个Action**，和Spring整合时Struts2的ActionBean注入作用域是原型模式prototype，然后通过setter，getter吧request数据注入到属性。Struts2中，一个Action对应一个request，response上下文，在接收参数时，可以通过属性接收，这说明属性参数是让多个方法共享的。Struts2中Action的一个方法可以对应一个url，而其类属性却被所有方法共享，这也就无法用注解或其他方式标识其所属方法了，只能设计为多例。

　　**SpringMVC是方法级别的拦截，一个方法对应一个Request上下文**，所以方法直接基本上是独立的，独享request，response数据。而每个方法同时又何一个url对应，参数的传递是直接注入到方法中的，是方法所独有的。处理结果通过ModeMap返回给框架。在Spring整合时，SpringMVC的Controller Bean默认单例模式Singleton，所以默认对所有的请求，只会创建一个Controller，有应为没有共享的属性，所以是线程安全的，如果要改变默认的作用域，需要添加@Scope注解修改。

　　**Struts2有自己的拦截Interceptor机制，SpringMVC这是用的是独立的Aop方式**，这样导致Struts2的配置文件量还是比SpringMVC大。

**Struts2采用Filter实现，SpringMVC则采用Servlet实现**。Filter在容器启动之后即初始化；服务停止以后坠毁，晚于Servlet。Servlet在是在调用时初始化，先于Filter调用，服务停止后销毁。
使用注解的话，SpringMVC基本上是零配置，而Struts需要配置很多。

## dispatchServlet的工作原理
1. DispatcherServlet继承FrameworkServlet，当客户端发送请求的时候， 会调用Servlet对应的doGet、doPost、doDelete等方法。　　
2. doGet、doPost、doDelete里面会调用processRequest方法
3. processRequest方法进一步调用doService方法
4. DispatcherServlet实现了doService方法，在doService方法中对Request参数进行处理，然后调用doDispatch方法
5. 在doDispatch方法中获取并调用处理器映射器、处理器适配器，获取并返回执行结果。doDispatch() 方法的主要过程是通过 HandlerMapping 获取 Handler，再找到用于执行它的 HandlerAdapter，执行 Handler 后得到 ModelAndView ，ModelAndView 是连接“业务逻辑层”与“视图展示层”的桥梁，接下来就要通过 ModelAndView 获得 View，再通过它的 Model 对 View 进行渲染
