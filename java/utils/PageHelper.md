# PageHelper



## 代码

```java
/**
 * 查询redis操作执行记录
 *
 * @param execution 操作执行体
 * @param pageNum page页号
 * @param pageSize page容量
 * @return List<RedisOperationExecution>
 */
@Override
public PageBean<RedisOperationExecution> queryExecutions(RedisOperationExecution execution, int pageNum, int pageSize) {
    // 开启PageHelper分页
    Page page = PageHelper.startPage(pageNum, pageSize);

    List<RedisOperationExecution> executions = executionMapper.queryAll(execution);

    // 组装分页
    PageBean<RedisOperationExecution> pageBean = new PageBean<>(
        page.getPageNum(), page.getPageSize(), NumberUtils.toInt(page.getTotal() + ""));
    pageBean.setItems(executions);
    return pageBean;
}
```



## PageHelper.startPage(page, rows) 所放位置

PageHelper.startPage 相当于开启分页，通过拦截MySQL的方式，把你的查询语句拦截下来加limit。你放后面，语句都查完了还拦啥呢，肯定是放在查询之前。