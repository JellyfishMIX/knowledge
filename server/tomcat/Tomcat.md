# Tomcat



## Composition

catalina 就是Tomcat服务器使用的 Apache实现的servlet容器的名字。

Tomcat的核心分为3个部分: 

1. Web容器—处理静态页面。
2. catalina — 一个servlet容器—–处理servlet。
3. 还有就是JSP容器，它就是把jsp页面翻译成一般的servlet。



## Story

### Apache、Tomcat与Catalina作为软件名字的含义与关系

如果你是从事于计算机软件相关工作的人，那你肯定经常见到Apache这个单词，也应该知道Tomcat这个服务器软件的名字，Catalina可能陌生一点，但你在配置tomcat时，一定会添加一个环境变量，然后指向tomcat的安装路径，这个环境变量的名字就叫Catalina_Home，进入tomcat安装目录，里面很多文件名字也叫Catalina。那么这三个单词作为软件的名字有什么含义、相互之间又是什么关系呢？

上世纪八十年代，当互联网开始在美国大学流行的时候，美国计算机名校伊利诺伊大学香槟分校（UIUC）的国家超级计算应用中心（National Center for Supercomputing Applications， NCSA）组织了一些研究生开始编写基于HTTP通信协议的服务器端和客户端程序。客户端端程序叫做mosaic，是第一个被普遍使用的网页浏览器，也是Netscape（网景）浏览器的前身，之后演变为Mozilla Firefox。而服务器端程序就是最早的Web服务器软件之一，名叫NCSA HTTPd，它完整地实现了HTTP协议，整个实验获得了成功。然而伊利诺伊大学香槟分校也许仅出于学术研究目的，在实验成功后开发工作就没有继续下去，研究小组也随之解散，但他们将这两个软件开源，其代码可以自由下载修改并发布。

此时的互联网对HTTP服务器软件的需求越来越大，公开源代码的NCSA HTTPd成了进一步发展的极好起点。很多研究者不断地给它添加功能、增加代码，并对不断出现的Bug打补丁。但因为缺乏规划和管理，出现了越来越多的重复劳动，随之而来的则是越多的补丁带来越多的Bug。1995年2月，为解决这种单打独斗的现象，8名志同道合的开发者决定成立一个小组，一起重写整个NCSA HTTPd程序，发布一个基于NCSA HTTPd的可靠的服务器软件。开发工作完成后，他们将软件命名为Apache，全称Apache HTTP Server。Apache本是美洲原住民印第安人一支部落的名字，这个部落因为高超的作战策略和无穷的耐性而闻名，同时也是最后一个屈服于美国政府的民族。开发小组以此寓意软件高效、可靠，同时表达了大公司迟早会参与竞争并“教化”这块最早的开源网络之地的担心。另外，因为整个软件是在NCSA HTTPd基础上打了很多补丁程序，他们也戏称它是“A Patchy Web Server”，意为一个打了很多补丁的Web服务器软件。“A Patchy”与Apache谐音，故以Apache命名一语双关。

Apache HTTP Server发布后，由于其具有坚实的稳定性、异常丰富的功能和灵活的可扩展性，得到了极大的成功。1999年6月，为有效支持Apache HTTP Server以及相关软件的发展，Apache开发小组成员们成立了一个非盈利性的Apache软件基金会（Apache Software Foundation）。大家对Apache这个名字的熟悉大概也是因为这个基金会，它支持开发了诸多享誉全球的开源软件，这些软件的名字前都会加上Apache，其中就包括Apache Tomcat。

**Tomcat的这个单词的意思是“公猫”，因为它的开发者姆斯·邓肯·戴维森希望用一种能够自己照顾自己的动物代表这个软件，于是命名为tomcat，它的Logo兼吉祥物也被设计成了一只公猫形象。**Tomcat是1999年Apache 软件基金会与Sun等其他公司一起合作的Jakarta（雅加达）项目中的一个子项目，作为服务器的容器支持基于Java语言编写的程序在服务器上运行，这样的程序被称为Servlet，因为它是运行在“Server”上的“Applet”。理论上讲这样一个容器并不是一个完整的服务器软件，因为它只能运行Java程序而不能生成HTML页面数据，也不能处理并发事务。但它集成了HTTP服务器程序，也就可以单独作为一个服务器软件来部署以处理HTTP请求，但tomcat核心技术并不在于此，所以除了用于开发过程中的调试以及那些对速度和事务处理只有很小要求的用户，很少会将Tomcat单独作为Web服务器。通常开发者会让tomcat与其他对Web服务器一起协同工作，比如Apache HTTP Server。Apache HTTP Server负责接受所有来自客户端的HTTP请求，然后将Servlets和JSP的请求转发给Tomcat来处理。Tomcat完成处理后，将响应传回给Apache，最后Apache将响应返回给客户端。于是在tomcat中运行Java程序也就是Servlet的那个模块因为体现了tomcat最核心特点而引起了大家的重视，而这个模块的名字叫做Catalina。

**Catalina是美国西海岸靠近洛杉矶22英里的一个小岛，因为其风景秀丽而著名。Servlet运行模块的最早开发者Craig McClanahan因为喜欢Catalina岛故以Catalina命名他所开这个模块，尽管他从来也没有去过那里。**另外在开发的早期阶段，Tomcat是被搭建在一个叫Avalon的服务器框架上，而Avalon则是Catalina岛上的一个小镇的名字，于是想一个与小镇名字相关联的单词也是自然而然。还有一个原因来自于Craig McClanahan养的猫，他养的猫在他写程序的时候喜欢在电脑周围闲逛。但这与Catalina有什么关系呢？我想可能是Catalina岛是个悠闲散步的好地方，猫的闲逛让Craig McClanahan想起了那里。

### 为什么Oracle中的用户名是'scott'，密码是'tiger'

**布鲁斯·斯科特（Bruce Scott）是甲骨文（当时的软件开发实验室）的第一批员工之一。**

**SCOTT模式（EMP和DEPT表格）与密码TIGER是由他创建的。**

**tiger是他的猫的名字。**

他于1984年与Umang Gupta一起创办了Gupta Technology（现在称为Centura Software），后来成为PointBase，Inc.的首席执行官和创始人.Bruce是Oracle V1，V2和V3的合着者和共同架构师。



## 引用/参考

[tomcat中catalina是什么 - 著一 - CSDN](https://blog.csdn.net/limuzi13/article/details/52805759)

[tomcat为什么把内部的jar包取名为catalina，有什么内部含意，或寄寓吗？ - 勃拉图的回答 - 知乎 ](https://www.zhihu.com/question/68213723/answer/260766297)