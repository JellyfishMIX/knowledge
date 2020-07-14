# git-problems



## 解决Git中fatal: refusing to merge unrelated histories

### 发现问题

在使用Git创建项目的时候，在两个分支合并的时候，出现了下面的这个错误。

```bash
~/SpringSpace/newframe on  master ⌚ 11:35:56
$ git merge origin/druid
fatal: refusing to merge unrelated histories
```

问题的关键在于：fatal: refusing to merge unrelated histories 
可能会在 `git pull` 或者 `git push` 中都有可能会遇到，这是因为两个分支没有取得关系。

### 解决问题

在操作命令后面加 `--allow-unrelated-histories` 
例如：

```bash
git merge master --allow-unrelated-histories
```

如果是 `git pull` 或者 `git push` 报 `fatal: refusing to merge unrelated histories`，同理：

```bash
git pull origin master --allow-unrelated-histories
```

问题解决！



## 引用/参考

[解决Git中fatal: refusing to merge unrelated histories - 向小凯同学学习 - CSDN](https://blog.csdn.net/wd2014610/article/details/80854807)