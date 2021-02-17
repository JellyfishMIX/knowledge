# 使用js，计算两个日期差，天，小时，分钟，毫秒



```javascript
const date1 = new Date('2017-04-25');
const date2 = new Date('2017-06-25');
const s1 = date1.getTime();
const s2 = date2.getTime();
const total= (s2 - s1);
// 计算整数天数*1000毫秒
const date3 = parseInt(total / (24 * 60 * 60));

// 计算出相差天数
const day =Math.floor(date3/(24*3600*1000));

// 计算出小时数
const leave1=date3%(24*3600*1000);    //计算天数后剩余的毫秒数
const hours=Math.floor(leave1/(3600*1000));

// 计算相差分钟数
const leave2=leave1%(3600*1000);        //计算小时数后剩余的毫秒数
const minutes=Math.floor(leave2/(60*1000));

// 计算相差秒数
const leave3=leave2%(60*1000);      //计算分钟数后剩余的毫秒数
const seconds=Math.round(leave3/1000);
alert('相差 "+days+"天 "+hours+"小时 "+minutes+" 分钟"+seconds+" 秒');
```