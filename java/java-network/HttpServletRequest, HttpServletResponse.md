# HttpServletRequest, HttpServletResponse



## HttpServletRequest的方法

- request.getRequestURL() 返回全路径
  - e.g. `http://localhost:8080/jqueryLearn/resources/request.jsp `

- request.getRequestURI() 返回除去host（域名或者ip）部分的路径
  - e.g. `/jqueryLearn/resources/request.jsp`

- request.getContextPath() 返回工程名部分，如果工程映射为/，此处返回则为空
  - e.g.  `/jqueryLearn`

- request.getServletPath() 返回除去host和工程名部分的路径
  - e.g. `/resources/request.jsp `

注：

- resources为WebContext下的目录名 

- jqueryLearn 为工程名



## httpUrlConnection 的 setDoOutput 与 setDoInput

- `httpUrlConnection.setDoOutput(true);` 以后就可以使用 `connection.getOutputStream().write();`
- `httpUrlConnection.setDoInput(true);` 以后就可以使用 `connection.getInputStream().read();`

get请求用不到 `connection.getOutputStream()`，因为参数直接追加在地址后面，因此默认是 `false`。

post请求（比如：文件上传）需要往服务区传输大量的数据，这些数据是放在http的body里面的，因此需要在建立连接以后，往服务端写数据。因为总是使用 `connection.getInputStream()` 获取服务端的响应，因此默认值是 `true`。

API说的很清楚，可以看一下Java API文档：

```java
// URL 连接可用于输入和/或输出。如果打算使用 URL 连接进行输入，则将 DoInput 标志设置为 true；如果不打算使用，则设置为 false。默认值为 true。
public void setDoInput(boolean doinput)将此 URLConnection 的 doInput 字段的值设置为指定的值。  
// URL 连接可用于输入和/或输出。如果打算使用 URL 连接进行输出，则将 DoOutput 标志设置为 true；如果不打算使用，则设置为 false。默认值为 false。  
public void setDoOutput(boolean dooutput)将此 URLConnection 的 doOutput 字段的值设置为指定的值。  
```



## 引用/参考

[request.getRequestURL()和request.getRequestURI()的区别 - gavid0124 - CSDN](https://blog.csdn.net/Gavid0124/article/details/45390999)

[关于 httpUrlConnection 的 setDoOutput 与 setDoInput - -droidcoffee- - CSDN](https://blog.csdn.net/ID19870510/article/details/7031086)

