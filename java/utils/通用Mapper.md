# 通用Mapper



## Maven depdency

```xml
<!--通用mapper-->
<dependency>
    <groupId>tk.mybatis</groupId>
    <artifactId>mapper-spring-boot-starter</artifactId>
    <version>2.0.3</version>
</dependency>
```



## 使用方法

```java
	/**
     * 查询redis执行记录
     *
     * @param request request
     * @return PageBean<RedisExecutesResponse>
     */
    @Override
    public PageBean<RedisExecuteResponse> queryExecutes(QueryExecutesRequest request) {
        // 开启分页
        Page page = PageHelper.startPage(request.getPageNum(), request.getPageSize());

        Example example = new Example(RedisExecute.class);
        Example.Criteria criteria = example.createCriteria();
        criteria.andEqualTo("env", request.getEnv());
        criteria.andEqualTo("dbNum", request.getDbNum());
        criteria.andEqualTo("key", request.getKey());
        criteria.andEqualTo("status", RedisExecuteStateEnum.TO_BE_EXECUTED.getCode());
        // 排序
        example.orderBy("createTime").desc();
        List<RedisExecute> executesFromQuery = executeMapper.queryExecutes(request);

        List<RedisExecuteResponse> executesResponses = new ArrayList<>();
        for (RedisExecute executeFromQuery : executesFromQuery) {
            RedisExecuteResponse executeResponse = RedisMapStruct.INSTANCE.toRedisExecuteResponse(executeFromQuery);
            executesResponses.add(executeResponse);
        }

        // 组装分页
        PageBean<RedisExecuteResponse> pageBean = new PageBean<>(
                page.getPageNum(), page.getPageSize(), NumberUtils.toInt(page.getTotal() + ""));
        pageBean.setItems(executesResponses);
        return pageBean;
    }
```



## 实体类

```java
@AllArgsConstructor
@NoArgsConstructor
@Builder
@Table(name = "`redis_execute`")
public class RedisExecute {
    /**
     * 自增ID
     */
    @Column(name = "`id`")
    @GeneratedValue(generator = "JDBC")
    private Integer id;

    /**
     * redis编号
     */
    @Column(name = "`redis_no`")
    private String redisNo;

    /**
     * 执行环境
     */
    @Column(name = "`env`")
    private String env;

    /**
     * 过期时间
     */
    @Column(name = "`ttl`")
    private Integer ttl;

    /**
     * 数据库编号
     */
    @Column(name = "`db_num`")
    private Integer dbNum;

    /**
     * 0-待执行;1-执行成功;2-执行失败;3-驳回
     */
    @Column(name = "`status`")
    private Integer status;

    /**
     * 执行人id
     */
    @Column(name = "`executor_id`")
    private String executorId;

    /**
     * 执行人名称
     */
    @Column(name = "`executor`")
    private String executor;

    /**
     * 执行时间
     */
    @Column(name = "`executor_time`")
    private Date executorTime;

    /**
     * 申请人编号
     */
    @Column(name = "`ask_id`")
    private Integer askId;

    /**
     * 申请人
     */
    @Column(name = "`asker`")
    private String asker;

    /**
     * 申请时间
     */
    @Column(name = "`ask_time`")
    private Date askTime;

    /**
     * 执行结果
     */
    @Column(name = "`msg`")
    private String msg;

    /**
     * 创建时间，自动写入
     */
    @Column(name = "`create_time`")
    private Date createTime;

    /**
     * 修改时间，自动写入
     */
    @Column(name = "`update_time`")
    private Date updateTime;

    /**
     * key
     */
    @Column(name = "`key`")
    private String key;

    /**
     * 操作的参数，以Map类型存储为json
     */
    @Column(name = "`value`")
    private String value;
}
```



## Mapper（Dao）

继承Mapper<实体类>

```java
public interface RedisExecuteMapper extends Mapper<RedisExecute> {

}
```