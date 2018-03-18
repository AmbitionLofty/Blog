#SpringBoot之配置文件解析
[toc]


#
#Spring Boot配置文件解析

### 前
Spring Boot使用“习惯优于配置”（项目中存在大量的配置，此外还内置了一个习惯性的配置，让你无需手动进行配置）的理念让你的项目快速运行起来。所以，我们要想把Spring Boot玩的溜，就要懂得如何开启各个功能模块的默认配置，这就需要了解Spring Boot的配置文件`application.properties`。


***

Spring Boot使用了一个全局的配置文件application.properties，放在src/main/resources目录下或者类路径的/config下。Sping Boot的全局配置文件的作用是对一些默认配置的配置值进行修改。

>如果你工程没有这个application.properties，那就在src/main/java/resources目录下新建一个.

####自定义属性

`application.properties`提供自定义属性的支持，这样我们就可以把一些常量配置在这里:

```
cn:
usr:
name: usr
address: JiNan
```

我这里使用的事`application.yml`格式的，等于`cn.usr.name = 有人物联网`

然后直接在要使用的地方通过注解`@Value(value="${config.name}")`就可以绑定到你想要的属性上面:

```java
@(SpringBoot)Value("${cn.usr.name}")
String name;


@RequestMapping("/testSend")
public String testSend() {
System.out.println(name);
return name;
}
```


有时候属性太多了，一个个绑定到属性字段上太累，官方提倡绑定一个对象的bean，这里我们建一个`ConfigBean.java`类，顶部需要使用注解`@ConfigurationProperties(prefix = "cn.usr")`来指明使用哪个


```java
/**
* @program: Learn-SpringBootRabbitmq
* @author: Rock 【shizhiyuan@usr.cn】
* @Date: 2018/3/1 0001
*/
@Component
@ConfigurationProperties(prefix = "cn.usr")
public class ConfigBean {
private String name;
private String address;

//....省略get/set
}
```

最后在使用到的地方比如：Controller中引入ConfigBean使用即可，如下：


```java
public class SenderController {

@Autowired
ConfigBean configBean;

@RequestMapping("/testSend")
public String testSend() {
return configBean.getAddress();
}

```

####参数间引用
在`application.properties`中的各个参数之间也可以直接引用来使用，就像下面的设置：

```
cn:
usr:
name: usr
address: JiNan
des: ${cn.usr.name}坐落在${cn.usr.address}

```

使用方式和上面一样不做描述了。


####使用自定义配置文件


时候我们不希望把所有配置都放在`application.yml`里面，这时候我们可以另外定义一个，这里我明取名为`test.yml`,路径跟也放在`src/main/resources`下面。


```java
@Component
@ConfigurationProperties(prefix = "cn.usr")
@PropertySource(value = "classpath:test.yml", encoding = "utf-8")
public class ConfigBean {
private String name;
private String address;

//....省略get/set
}
```


####随机值配置

配置文件中${random} 可以用来生成各种不同类型的随机值，从而简化了代码生成的麻烦，例如 生成 int 值、long 值或者 string 字符串。

```
cn:
usr:
name: usr
address: JiNan
des: ${cn.usr.name}坐落在${cn.usr.address}
number: ${random.int}
number.less.than.ten: ${random.int(10)}
number.in.range: ${random.int[1024.12323]}
bignumber: ${random.long}
uuid: ${random.uuid}
secret: ${random.value}
```


####外部配置-命令行参数配置

Spring Boot是基于`jar`包运行的，打成jar包的程序可以直接通过下面命令运行：

`java -jar xxx.jar`

举例：
修改tomcat端口号：`java -jar xxx.jar --server.port=9999`


可以看出，命令行中连续的两个减号--就是对`application.properties`中的属性值进行赋值的标识。


所以`java -jar xx.jar --server.port=9999`等价于在`application.properties`中添加属性`server.port=9090`。



如果你怕命令行有风险，可以使用`SpringApplication.setAddCommandLineProperties(false)`禁用它。



####配置文件的优先级

`application.properties`和`application.yml`文件可以放在一下四个位置：

- 外置，在相对于应用程序运行目录的/congfig子目录里。

- 外置，在应用程序运行的目录里

- 内置，在config包内

- 内置，在Classpath根目录


注意：此外，如果你在相同优先级位置同时有`application.properties`和`application.yml`，那么`application.yml`里面的属性就会覆盖`application.properties`里的属性。



####Profile-多环境配置

当应用程序需要部署到不同运行环境时，一些配置细节通常会有所不同，最简单的比如日志，生产日志会将日志级别设置为WARN或更高级别，并将日志写入日志文件，而开发的时候需要日志级别为DEBUG，日志输出到控制台即可。

如果按照以前的做法，就是每次发布的时候替换掉配置文件，这样太麻烦了，Spring Boot的Profile就给我们提供了解决方案，命令带上参数就搞定。


这里我们来模拟一下，只是简单的修改端口来测试。

在Spring Boot中多环境配置文件名需要满足`application-{profile}.properties`的格式，其中`{profile}`对应你的环境标识，比如：


```
- application-dev.properties：开发环境

- application-prod.properties：生产环境
```


当然你也可以用命令行启动的时候带上参数：

`java -jar xxx.jar --spring.prifiles.active=dev`

我给不同的环境添加不同的端口属性server.port，然后根据指定不同的spring.profiles.active来切换使用,这里就不贴代码了。

***


OVER.




