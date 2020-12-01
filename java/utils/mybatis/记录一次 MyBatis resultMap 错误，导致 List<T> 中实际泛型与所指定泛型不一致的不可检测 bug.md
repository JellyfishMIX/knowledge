## 记录一次 MyBatis resultMap 错误，导致 List\<T\> 中实际泛型与所指定泛型不一致的不可检测 bug



## 发现问题

近期写项目时，遇到了一处离奇的bug：

List\<T\> 中实际泛型与所指定泛型不一致

![IMG_8008(20201122-083453)](https://image-hosting.jellyfishmix.com/20201122084528.JPG)

如图所示，我传入的是 `List<YunfuDTO> listB`，数组中存储的元素，数据类型应为 `YunfuDTO` 但是运行时数组中存储的元素，数据类型是 `TaijiDTO`。这让我百思不得其解。



##分析问题

博主把目光转向了方法的调用处：

```java
// mysql 数据
List<YunfuDTO> existedYunfuDeviceModuleDTO = deviceModuleDao.queryYunfuAll();

// 取交集
List<YunfuDTO> intersectionYunfuDeviceModuleDTO = generateIntersectionYunfuDeviceModuleList(synYunfuDTOList, existedYunfuDeviceModuleDTO);
```

`existedYunfuDeviceModuleDTO` 即是出现问题的 listB，于是想到了溯源，追寻这个变量的产生。

这个变量是通过 mybatis 产生的。我们来看 mybatis: 

dao.java

```java
/**
 * 查询所有设备模块列表
 *
 * @return 设备模块列表
 */
List<YunfuDTO> queryYunfuAll();
```



mapper.xml

```xml
<resultMap id="TaijiDTOMap" type="com.skycomm.devsyn.inner.dto.TaijiDTO">
    <result property="moduleNo" column="module_no" jdbcType="VARCHAR"/>
    <result property="status" column="status" jdbcType="INTEGER"/>
    <result property="lastUpdateTime" column="last_update_time" jdbcType="TIMESTAMP"/>
</resultMap>

<select id="queryYunfuAll" resultMap="TaijiDTOMap">
    select module_no, status, last_update_time
    from device_module;
</select>
```

在 mybatis mapper 中，博主发现了问题原因，`resultMap="TaijiDTOMap"`。这样实际返回的 `List<T>`，元素的数据类型是 `TaijiDTO`，而不是在源码中显式指定的 `YunfuDTO`。

此处错误很隐蔽，编译器还无法检测出这个错误，顺利通过编译，只有运行时才会报错：`xxAClass cannot be cast to xxBClass`。

![Screen Shot 2020-11-22 at 9.08.19 AM](https://image-hosting.jellyfishmix.com/20201122090839.png)

截图为证，编译通过，运行报错。



## 解决问题

修改 `resultMap="TaijiDTOMap"` 为 `resultMap="YunfuDTOMap"`（当然 YunfuDTOMap 这个 resultMap 还需要编写 xml） 。即在 mapper.xml 中的 `resultMap` ，和 dao 中返回的 `List<T>` 的 `T` 数据类型保持一致。



## 问题总结

此处错误很隐蔽，编译器还无法检测出这个错误，顺利通过编译，只有运行时才会报错：`xxAClass cannot be cast to xxBClass`。