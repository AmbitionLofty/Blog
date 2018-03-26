# Spring boot内嵌容器tomcat配置

***
[toc]
***


@(SpringBoot)

## 默认容器

SpringBoot默认`web程序`启用 tomcat`内嵌容器tomcat`，监听`8080端口`,`servletPath`默认为 `/` 通过需要用到的就是端口、上下文路径的修改，在spring boot中其修改方法及其简单；


```profile

在资源文件中配置：   
server.port=9090 
server.contextPath=/SZY
```

## 自定义tomcat

在实际的项目中简单的配置tomcat端口肯定无法满足大家的需求，因此需要`自定义tomcat配置`信息来灵活的控制tomcat。

### 以定义默认编码为例


```java
package com.lkl.springboot.container.tomcat;

import org.springframework.boot.context.embedded.EmbeddedServletContainerFactory;
import org.springframework.boot.context.embedded.tomcat.TomcatEmbeddedServletContainerFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * tomcat 配置
 * @author zhiyuan
 * @version $Id: TomcatConfig.java, v 0.1 
 */
@Configuration
public class TomcatConfig {

    @Bean
    public EmbeddedServletContainerFactory servletContainer() {
        TomcatEmbeddedServletContainerFactory tomcat = new TomcatEmbeddedServletContainerFactory();
        tomcat.setUriEncoding("UTF-8");
        return tomcat;
    }

}
```


构建`EmbeddedServletContainerFactory`的`bean`，获取到`TomcatEmbeddedServletContainerFactory`实例以后可以对tomcat进行设置，例如这里设置编码为`UTF-8`



### 内嵌Tomcat SSL

配置资源文件:
```profile
server.port=8443
server.ssl.enabled=true
server.ssl.keyAlias=springboot
server.ssl.keyPassword=123456
server.ssl.keyStore=/Users/shizhiyuan/software/ca1/keystore
```

- `server.ssl.enabled` 启动tomcat ssl配置
- `server.ssl.keyAlias` 别名
- `server.ssl.keyPassword` 密码
- `server.ssl.keyStore` 位置


### 多端口监听配置

前面启动ssl后只能走https,不能通过http进行访问，如果要监听多端口，可采用编码形式实现。

1. 注销前面ssl配置，设置配置 `server.port=9090`

2. 修改`TomcatConfig.java`

```java
package com.lkl.springboot.container.tomcat;

import java.io.File;

import org.apache.catalina.connector.Connector;
import org.apache.coyote.http11.Http11NioProtocol;
import org.springframework.boot.context.embedded.EmbeddedServletContainerFactory;
import org.springframework.boot.context.embedded.tomcat.TomcatEmbeddedServletContainerFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * tomcat 配置
 * @author shizhiyuan
 * @version $Id: TomcatConfig.java, v 0.1 
 */
@Configuration
public class TomcatConfig {

    @Bean
    public EmbeddedServletContainerFactory servletContainer() {
        TomcatEmbeddedServletContainerFactory tomcat = new TomcatEmbeddedServletContainerFactory();
        tomcat.setUriEncoding("UTF-8");
        tomcat.addAdditionalTomcatConnectors(createSslConnector());
        return tomcat;
    }

    private Connector createSslConnector() {
        Connector connector = new Connector("org.apache.coyote.http11.Http11NioProtocol");
        Http11NioProtocol protocol = (Http11NioProtocol) connector.getProtocolHandler();
        try {
            File truststore = new File("/Users/liaokailin/software/ca1/keystore");
            connector.setScheme("https");
            protocol.setSSLEnabled(true);
            connector.setSecure(true);
            connector.setPort(8443);
            protocol.setKeystoreFile(truststore.getAbsolutePath());
            protocol.setKeystorePass("123456");
            protocol.setKeyAlias("springboot");
            return connector;
        } catch (Exception ex) {
            throw new IllegalStateException("cant access keystore: [" + "keystore" + "]  ", ex);
        }
    }
}

```

通过`addAdditionalTomcatConnectors`方法添加多个监听连接;此时可以通过`http 9090`端口，`https 8443`端口。



*** 

OVER