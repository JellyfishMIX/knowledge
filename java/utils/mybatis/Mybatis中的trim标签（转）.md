# Mybatis中的trim标签（转）



使用过trim标签都知道trim标签有四个属性：

```
prefix，prefixOverrides，suffix，suffixOverrides
```

本人一直对这四个标签的名字无法理解，并对其功能感到混乱。下面是自己思考后的一些总结：

例如：

```xml
<update id="testTrim" parameterType="com.mybatis.pojo.User">
    update user
    <trim prefix="set" suffixOverrides=",">
        <if test="cash!=null and cash!=''">cash= #{cash},</if>
        <if test="address!=null and address!=''">address= #{address},</if>
    </trim>
    <where>id = #{id}</where>
</update>12345678
```

只有prefix=“set”，表示在trim包裹的部分的前面添加 set。 

只有suffixOverrides=“,”，表示删除最后一个逗号。

上例也可以写成：

```xml
<update id="testTrim" parameterType="com.mybatis.pojo.User">
	update user
    set
    <trim suffixOverrides="," suffix="where id = #{id}">
    	<if test="cash!=null and cash!=''">cash= #{cash},</if>
        <if test="address!=null and address!=''">address= #{address},</if>
    </trim>
</update>12345678
```

由于set写在了外面，trim中就不再需要prefix属性了，所以删除。 

where标签从外面拿进trim里面，这样其实可以认为是将最后一个逗号”,”替换成了where id = #{id}。所以suffix和suffixOverrides一起使用。



## 转自

[Mybatis中的trim标签 介绍 - BaldWinf - CSDN](https://blog.csdn.net/u011118321/article/details/68946027)