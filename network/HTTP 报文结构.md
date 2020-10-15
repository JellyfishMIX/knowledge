# HTTP 报文结构



## HTTP报文结构

用于 HTTP 协议交互的信息被称为 HTTP 报文。请求端（客户端）的HTTP 报文叫做请求报文，响应端（服务器端）的叫做响应报文。
HTTP 报文本身是由多行（用 CR+LF 作换行符）数据构成的字符串文本。
HTTP 报文大致可分为报文首部和报文主体两块。两者由最初出现的空行（CR+LF）来划分。通常，并不一定要有报文主体。

![img](https://image-hosting.jellyfishmix.com/20201023145120)

注：\r是回车符,\n是换行符

> 计算机还没有出现之前，有一种叫做电传打字机（Teletype Model 33）的玩意，每秒钟可以打10个字符。但是它有一个问题，就是打完一行换行的时候，要用去0.2秒，正好可以打两个字符。要是在这0.2秒里面，又有新的字符传过来，那么这个字符将丢失。 
> 于是，研制人员想了个办法解决这个问题，就是在每行后面加两个表示结束的字符。一个叫做“回车”，告诉打字机把打印头定位在左边界；另一个叫做“换行”，告诉打字机把纸向下移一行。 
> 这就是“换行”和“回车”的来历，从它们的英语名字上也可以看出一二。 后来，计算机发明了，这两个概念也就被般到了计算机上。那时，存储器很贵，一些科学家认为在每行结尾加两个字符太浪费了，加一个就可以。于是，就出现了分歧。Unix 系统里，每行结尾只有“<换行>”，即“\n”；Windows系统里面，每行结尾是“<回车><换行>”，即“ \r\n”；Mac系统里，每行结尾是“<回车>”。一个直接后果是，Unix/Mac系统下的文件在Windows里打开的话，所有文字会变成一行；而Windows里的文件在Unix/Mac下打开的话，在每行的结尾可能会多出一个^M符号。



## HTTP请求报文

![img](https://image-hosting.jellyfishmix.com/20201023145209.jpeg)

![img](https://image-hosting.jellyfishmix.com/20201023145220)



## HTTP响应报文

![img](https://image-hosting.jellyfishmix.com/20201023145419.jpeg)

![img](https://image-hosting.jellyfishmix.com/20201023145441)



## HTTP首部分类

HTTP 首部字段是由首部字段名和字段值构成的，中间用冒号“:” 分隔。
例如，在 HTTP 首部中以 Content-Type 这个字段来表示报文主体的 对象类型。Content-Type: text/html

![img](https://image-hosting.jellyfishmix.com/20201023145512)

HTTP首部字段类型

- 通用首部字段（General Header Fields）：请求报文和响应报文两方都会使用的首部。

- 请求首部字段（Request Header Fields）：从客户端向服务器端发送请求报文时使用的首部。补充了请求的附加内容、客户端信息、响应内容相关优先级等信息。

- 响应首部字段（Response Header Fields）：从服务器端向客户端返回响应报文时使用的首部。补充了响应的附加内容，也会要求客户端附加额外的内容信息。

- 实体首部字段（Entity Header Fields）：针对请求报文和响应报文的实体部分使用的首部。补充了资源内容更新时间等与实体有关的信息。



