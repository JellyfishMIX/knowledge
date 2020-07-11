# 常用SQL



##  查询指定步长

从起始时间2020-05-20 21:03:00查询间隔1秒到结束时间2020-05-20 21:04:00的所有时间
即：从起始时间2020-05-20 21:03:00以1秒为步长的到到结束时间2020-05-20 21:04:00的所有时间

```
select * from wx_interchange.file_info
where (datediff('2020-05-20 21:03:00', wx_interchange.file_info.creation_time)%1) = 0
  and wx_interchange.file_info.creation_time >= '2020-05-20 21:03:00'
  and wx_interchange.file_info.creation_time <= '2020-05-20 21:04:00';
```