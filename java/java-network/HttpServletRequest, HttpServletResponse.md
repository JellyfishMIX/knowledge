## HttpServletRequest的方法

request.getRequestURL() 返回全路径

request.getRequestURI() 返回除去host（域名或者ip）部分的路径

request.getContextPath() 返回工程名部分，如果工程映射为/，此处返回则为空

request.getServletPath() 返回除去host和工程名部分的路径

 

例如：


request.getRequestURL() http://localhost:8080/jqueryLearn/resources/request.jsp 
request.getRequestURI() /jqueryLearn/resources/request.jsp
request.getContextPath()/jqueryLearn 
request.getServletPath()/resources/request.jsp 

注： resources为WebContext下的目录名 
      jqueryLearn 为工程名



## 引用/参考

[request.getRequestURL()和request.getRequestURI()的区别 - gavid0124 - CSDN](https://blog.csdn.net/Gavid0124/article/details/45390999)