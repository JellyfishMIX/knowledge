# css超出一定宽度用省略号代替



超出一行，用省略号：

```css
{
    // 超出的文本隐藏
    overflow:hidden;
    // 溢出用省略号显示
    text-overflow:ellipsis;
    // 溢出不换行
    white-space:nowrap;
}
```

超出两行或者多行，用省略号：

```css
{
    overflow:hidden; 
	text-overflow:ellipsis;
    // 将对象作为弹性伸缩盒子模型显示
	display:-webkit-box; 
    // 从上到下垂直排列子元素（设置伸缩盒子的子元素排列方式）
	-webkit-box-orient:vertical;
    // 这个属性不是css的规范属性，需要组合上面两个属性，表示显示的行数。
	-webkit-line-clamp:2;
}
```



## 转自

[css超出一定宽度用省略号代替 - 姜-无忧 - CSDN](https://blog.csdn.net/xiasohuai/article/details/81836351)