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
