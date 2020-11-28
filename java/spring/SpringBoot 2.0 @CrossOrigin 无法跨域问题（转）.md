# SpringBoot 2.0 @CrossOrigin 无法跨域问题（转）



## 前言

在spring boot 1.5中，配置跨域一般是直接在controller或是在某一个方法上添加 **@CrossOrigin** 注解即可：

```java
/**



 * @author chenws



 * @decription



 * @date 2018/10/18



 */



@RestController



@RequestMapping(value = "xxx")



@CrossOrigin(maxAge = 3600)



public class TestController {



 



	@ApiOperation("xxx")



	@RequestMapping(value = "/xxx",method = RequestMethod.POST)



	public ResponseVO<List<xxx>> test(@RequestBody xxx xxx){}



}
```

但是在spring boot 2.0中（springframework5.0.2后），以上方法行不通，后来查看@CrossOrigin源码

**springframework4.3.12：**

```java
/**



	 * Whether the browser should include any cookies associated with the



	 * domain of the request being annotated.



	 * <p>Set to {@code "false"} if such cookies should not included.



	 * An empty string ({@code ""}) means <em>undefined</em>.



	 * {@code "true"} means that the pre-flight response will include the header



	 * {@code Access-Control-Allow-Credentials=true}.



	 * <p>If undefined, credentials are allowed.



	 */



	String allowCredentials() default "";
```

**springframework5.0.2**

```java
/**



	 * Whether the browser should send credentials, such as cookies along with



	 * cross domain requests, to the annotated endpoint. The configured value is



	 * set on the {@code Access-Control-Allow-Credentials} response header of



	 * preflight requests.



	 * <p><strong>NOTE:</strong> Be aware that this option establishes a high



	 * level of trust with the configured domains and also increases the surface



	 * attack of the web application by exposing sensitive user-specific



	 * information such as cookies and CSRF tokens.



	 * <p>By default this is not set in which case the



	 * {@code Access-Control-Allow-Credentials} header is also not set and



	 * credentials are therefore not allowed.



	 */



	String allowCredentials() default "";
```

By default this is not set in which case the {@code Access-Control-Allow-Credentials} header is also not set and credentials are therefore not allowed.

5.0.2后，allowCredentials默认为false了，再看 DefaultCorsProcessor

```java
if (Boolean.TRUE.equals(config.getAllowCredentials())) {



 



	responseHeaders.setAccessControlAllowCredentials(true);



 



}
```

allowCredentials为true时，返回的响应头AccessControlAllowCredentials属性才设置为true，允许客户端携带验证消息。



## 解决办法

在注解中设置allowCredentials为true即可

```java
@CrossOrigin(allowCredentials="true",maxAge = 3600)
```



## 转自

[SpringBoot 2.0 @CrossOrigin 无法跨域问题 - 碩果 - CSDN](https://blog.csdn.net/shuoshuo132/article/details/83146668)