# xml schema



## 说明

1. @author [JellyfishMIX - github](https://github.com/JellyfishMIX) / [blog.jellyfishmix.com](http://blog.jellyfishmix.com)
2. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## xml 命名空间

1. 在 w3c 的官方说法中，命名空间起到了避免元素命名冲突的作用。这里就不再举例详细说明了，如果要避免重名的冲突，在 xml 中可以使用多个不同的命名空间。

命名空间的语法:

```
xmlns:namespace-prefix="namespaceURI"
```

2. 当命名空间声明在元素的开始标签中时，所有带有相同前缀(namespace-prefix)的子元素都会与同一个命名空间相关联，前缀可以看做是命名空间的一个别名。后面 demo 中会理解这一点。



## xml schema, xsd, xml 三者间的关系

1. xml schema 是定义 xml 文件的合法构建模块，可以理解为是一种编写 xsd 或 xml 文件的语法。使用 schema 语法编写的文件后缀名可以是 .xsd 或 .xml。
2. xsd(xml schema definition) 文件，后缀名 .xsd，定义 xml 文件中可以出现哪些元素, 属性, 元素之间的关系, 顺序, 元素的数量, 元素或属性的类型和值的范围等，是对 xml 文件的一种约束方式。xsd 文件也使用 schema 语法编写。
3. 总结，xml schema 是语法，可以用来编写 xsd 或 xml 文件。xsd 文件是对 xml 文件的约束，xml 文件中导入 xsd 后就可以使用 xsd 中定义的元素, 属性等。



## xsd 文件 demo

1. 属性 targetNamespace="http://www.w3school.com.cn"，声明此 xsd 文件定义的元素的命名空间: "http://www.w3school.com.cn"。类似于 java 中的 package，定义了一个包名。

2. 属性 xmlns="http://www.w3school.com.cn"，指定默认的命名空间是 "http://www.w3school.com.cn"。类似于 java 中的 import，导入了要使用的类。

```xml
<?xml version="1.0"?>
 
<xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema"
targetNamespace="http://www.w3school.com.cn"
xmlns="http://www.w3school.com.cn"
elementFormDefault="qualified">

...
...
</xs:schema>
```

demo 中 targetNamespace 和 xmlns 一样的原因是:

1. 首先要用 targetNamespace 属性声明当前 xsd 文件的命名空间，才能被使用此 xsd 文件的地方导入。
2. 当前 xsd 文件要使用当前 xsd 文件中定义的元素，因此需要通过 xmlns 导入自己这个 xsd 的命名空间。

当前 xsd 文件想使用当前 xsd 文件自己定义的元素，需要"自己导入自己"，原因是: 

1. xml 中，使用的任何元素, 属性, 数据类型的完整名称需要包含两部分: [命名空间前缀:元素名], [命名空间前缀:属性名], [命名空间前缀:数据类型名]。即使当前 xsd 文件想使用当前 xsd 文件自己定义的元素，也需要加上命名空间前缀，因此需要把 xsd 文件自己的命名空间通过 xmlns 导入进来。
2. 对于默认的命名空间 xmlns 中的元素, 属性, 数据类型，前缀可以省略，但不表示没有，只是省略了而已。



## xsd + xml 的 demo

### xsd 文件

xsd 文件定义了一些约束。文件名 user.xsd

1. 定义了父标签 user 和子标签 name, age, phone。
2. 其中 phone 标签的 type 是在当前 xsd 文件中自定义的数据类型，想要使用这个自定义的数据类型，需要:
   1. 先使用 targetNamespace="http://www.gzn.com"，声明这个 xsd 文件的命名空间为 "http://www.gzn.com"。(可以理解为 java 声明类所在的 package)。
   2. 通过 xmlns="http://www.gzn.com"，导入自己的命名空间。(可以理解为 java import 进来了一个类)

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema"
targetNamespace="http://www.gzn.com"
xmlns="http://www.gzn.com"
elementFormDefault="qualified">
<xs:element name="user">
    <xs:complexType>
        <xs:sequence>
            <xs:element name="name" type="xs:string"/>
            <xs:element name="age" type="xs:integer"/>
            <!--这里使用了自定义数据类型-->
            <xs:element name="phone" type="phone-number"/>
        </xs:sequence>
    </xs:complexType>
</xs:element>
    
<!-- 自定义数据类型: 手机号码 -->
<xs:simpleType name="phone-number">
     <xs:restriction base="xs:string">
         <xs:pattern value="1[3|4|5|7|8][0-9]{9}"/>
     </xs:restriction>
</xs:simpleType>

</xs:schema>
```

### xml 文件

通过 xmlns="http://www.gzn.com"，引入了上述 xsd 的 xml 代码

因此可以使用父标签 user 和子标签 name, age, phone。

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<user xmlns="http://www.gzn.com"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:schemaLocation="http://www.gzn.com http://www.gzn.com user.xsd">
    <name>gzn</name>
    <age>18</age>
    <phone>13166666666</phone>
</user>
```

如果将 user.xsd 中的属性 xmlns="http://www.gzn.com" 删除掉，相当于 user.xsd 文件未导入自己的元素，`<xs:element name="phone" type="phone-number"/>` 就无法识别 phone-number 这个数据类型了，继而在 xml 中的电话号码无法得到校验，`<phone>13166666666</phone>` 就可以输入不合法的任意号码。

### 给 xmlns 导入的 xsd 命名空间加上前缀

我们把 `xmlns="http://www.gzn.com"` 改成 `xmlns:people="http://www.gzn.com"`，给导入 xsd 命名空间加上了前缀，xml 文件中使用此 xsd 命名空间中定义的元素时，也需要加上同样的前缀，`<user></user>` 变成 `<people:user></people:user>`

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<people:user xmlns:people="http://www.gzn.com"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:schemaLocation="http://www.gzn.com http://www.gzn.com user.xsd">
    <people:name>gzn</people:name>
    <people:people:age>18</people:age>
    <people:phone>13166666666</people:phone>
</people:user>
```



## xsi:schemaLocation 的作用

1. xsi:schemaLocation 属性是 namespace 为 http://www.w3.org/2001/XMLSchema-instance 里的 schemaLocation 属性。xsi 是指定的命名空间的前缀。

2. 定义了 namespace 和对应的 xsd 文件的位置关系。它的值由一个或多个 url 引用对组成(注意是成对的)，两个 url 之间以空白符分隔(空格和换行均可)。
3. 第一个 url 表示想导入的 xsd 文件的 namespace。第二个 url 定义 xsd 文件的位置，schema 解析时将从这个文件位置获取 xsd 文件，第一个 url 必须与该 xsd 文件的 targetNamespace 相匹配。
4. 说明一下"第一个 url 必须与该 xsd 文件的 targetNamespace 相匹配"，因为 xsd 文件中 targetNamespace 声明了命名空间(可以理解为 java 中声明了包名)，所以第一个 url 需要导入同样的命名空间(可以理解为 java 中按包名 import)。
5. xsi:schemaLocation 中如果 url 不是偶数个，会报错 url 必须是偶数个。因为偶数个 url 才是成对的。

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
	http://code.alibabatech.com/schema/dubbo http://code.alibabatech.com/schema/dubbo/dubbo.xsd">
</beans>
```



## elementFormDefault 和 attributeFormDefault

1. elementFormDefault="unqualified" 表示此 xsd 文件中定义的子元素，在使用时非必须指定命名空间前缀，子元素从属于顶级父元素的命名空间。
2. elementFormDefault="qualified" 表示此 xsd 文件中定义的子元素，在使用时必须指定命名空间前缀。
3. attributeFormDefault="unqualified" 表示此 xsd 文件中定义的属性，在使用时非必须指定命名空间前缀。
4. attributeFormDefault="qualified" 表示此 xsd 文件中定义的属性，在使用时必须指定命名空间前缀。

例如下面的 xsd 文件，user 是顶级元素，name 是子元素，user 是 name 的父元素。

```xml
<xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema"
	targetNamespace="http://www.gzn.com"
	xmlns="http://www.gzn.com"
	elementFormDefault="qualified"
    attributeFormDefault="unqualified">
<xs:element name="user">
    <xs:complexType>
        <xs:sequence>
            <xs:element name="name" type="xs:string"/>
            <xs:element name="age" type="xs:integer"/>
            <!--这里使用了自定义数据类型-->
            <xs:element name="phone" type="phone-number"/>
        </xs:sequence>
    </xs:complexType>
</xs:element>
    
<!-- 自定义数据类型: 手机号码 -->
<xs:simpleType name="phone-number">
     <xs:restriction base="xs:string">
         <xs:pattern value="1[3|4|5|7|8][0-9]{9}"/>
     </xs:restriction>
</xs:simpleType>

</xs:schema>
```

