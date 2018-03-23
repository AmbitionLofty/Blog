# 物联网协议之CoAP协议开发学习笔记初始篇

***
[toc]
***


@(物联网)



> 哪有什么天生如此，只是我们天天坚持。 -Zhiyuan

CoAP协议博大精深，网上资料较少，大多是外网未翻译的文章，英语水平有限，如有不足，大家... 我怎么可能有不足！这可是全网最全CoAP协议文章。(有不足大家还请尽情在下面评论) 有过对你有帮助，点赞收藏走一波啊~~~

想了下还是把文章分开写，此篇文章介绍 **CoAP是何方神圣有何神通**，感兴趣的朋友可查看我其余文章。

Let's Go !  I'm Coming CoAP ! 


1.揭开CoAP的神秘面纱
-------------
老规矩先来看最权威的定义：

 1. 维基百科：
>         Constrained Application Protocol (CoAP) is an specialized Internet Application Protocol for constrained devices, as defined in RFC 7228).
        It enables those constrained devices called "nodes" to communicate with the wider Internet using similar protocols.
        CoAP is designed for use between devices on the same constrained network (e.g., low-power, lossy networks), between devices and general nodes on the Internet, and between devices on different constrained networks both joined by an internet.
        CoAP is also being used via other mechanisms, such as SMS on mobile communication networks.

>         CoAP is a service layer protocol that is intended for use in resource-constrained internet devices, such as wireless sensor network nodes. 
         CoAP is designed to easily translate to HTTP for simplified integration with the web, while also meeting specialized requirements such as multicast support, very low overhead, and simplicity.
        Multicast, low overhead, and simplicity are extremely important for Internet of Things (IoT) and Machine-to-Machine (M2M) devices, which tend to be deeply embedded and have much less memory and power supply than traditional internet devices have. 
        Therefore, efficiency is very important. CoAP can run on most devices that support UDP or a UDP analogue.    
      
 2. 百度百科：
      >      由于物联网中的很多设备都是资源受限型的，即只有少量的内存空间和有限的计算能力，所以传统的HTTP协议应用在物联网上就显得过于庞大而不适用。
        IETF的CoRE工作组提出了一种基于REST架构的CoAP协议。
        由于无线物联网中的设备很多都是资源受限型的，这些设备只有少量的内存空间和有限的计算能力。
        为此，IETF（Intemet Engineering Task Force）的CoRE（Constrained RESTful Environment）工作组为受限节点制定相关的REST（Representational State Transfer）形式的应用层协议。
        这就是CoRE工作组正在制订的CoAP（Constrained Application Protocol）协议。

再来参考一下白话一点的 重点都划出来了：
   
**CoAP** 是 **受限制的应用协议**(Constrained Application Protocol)的代名词。
最近几年专家们预测会有更多的设备相互连接，而这些设备的数量将远超人类的数量。在这种大背景下，物联网和M2M技术应运而生。
虽然对人们而言，连接入互联网显得方便容易，但是对于那些 **微型设备 而言接入互联网**非常困难。在当前由PC机组成的世界，信息交换是通过TCP和应用层协议HTTP实现的。但是对于小型设备而言，实现TCP和HTTP协议显然是一个过分的要求。为了让小设备可以接入互联网，CoAP协议被设计出来。
**CoAP是一种应用层协议**，它**运行于UDP协议之上**而不是像HTTP那样运行于TCP之上。
**CoAP协议非常小巧，最小的数据包仅为4字节。**





2.先看看CoAP的小知识点
-------------

 - 上文一直说**CoAP是为小设备量身打造的**，但是小设备是指那些呢？

         小设备：256KB Flash 32KB RAM 20MHz主频....的设备
            
 - 能不能替代HTTP协议?
 
         不能！ HTTP协议是爸爸，嗯，没有啦。
         
 - CoAP协议有哪些特点？

 ```
        1.满足资源受限的网络需求。
        2.无状态HTTP映射，可以通过HTTP代理实现访问CoAP资源，或者在CoAP智商构建HTTP接口。
        3.使用UDP实现可靠IP单播和最大努力IP多播。
        4.异步消息交换
        5.很小的消息头载荷及解析复杂度。
        6.支持URI和内容类型(Content-type).
        7.支持代理和缓存.
        8.内建资源发现.
        9.可以使用DTLS作为安全加密层。
        10.资源消耗低，所需RAM和ROM资源均小于10KB。
        11.其双层(事务层，请求/响应层)处理方式可支持异步通信.
        12.支持观察模式。
        13.支持块传输
 ```

    
  是不是觉得CoAP还是蛮厉害的。贼啦牛掰~~~ 
  同样是物联网协议，我们来看一下MQTT和CoAP的区别，没用过 MQTT的朋友凭你的悟性自己慢慢看。
   

3.CoAP与MQTT比较区别
-------------
MQTT和CoAP都是非常有用的物联网协议，但两者有根本区别，两个协议各有特点。
 1. **MQTT** 是多个客户端通过一个**中央代理**传递消息的多对多协议。它通过让客户端发布消息、代理决定消息路由和复制来解耦生产者和消费者。虽然MQTT持久性有一些支持，但它是最好的实时通讯总线。
 2. **CoAP** 基本上是**一个在Client和Server之间传递状态信息的单对单协议**。虽然它支持观察资源，但是CoAP最适合状态转移模型，而不是单纯的基于事件。
 3. **MQTT：** Clients与Broker之间保持TCP长连接，这个在NAT环境中也不会有问题。
    
 4. **CoAP：** Clients与Server都要接收和发送UDP包。在NAT环境下使用CoAP，需要使用“隧道掘进”或者端口转发(内网穿透)，否则像LWM2M（轻量级M2M）一样，首先初始化设备到‘头端’( head-end )的连接.

 5. **MQTT**不支持带有类型或者其它帮助Clients理解的标签消息。MQTT消息可用于任意目的，但前提是所有的Clients必须知道消息格式。
**而CoAP则相反**，它**内置内容协商和发现支持**，这样允许设备彼此窥测以找到交换数据的方式。

    
 
点赞收藏走一波~~~~





    
    
    
    

    
    
