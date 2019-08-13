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
