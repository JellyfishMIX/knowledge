# git-commonly-used-commands



## git本地公钥位置

```bash
cat ~/.ssh/id_rsa.pub
```



## git创建本地仓库

```bash
git init
```



## 远程仓库

### git添加关联的远程仓库地址

```bash
git remote add origin git@github.com:jellyfishmix/learngit.git
```

origin是远程仓库的名字，可以自行替换。

### git查看此本地仓库已关联的远程仓库地址

```bash
git remote -v 
```

通常每个关联的远程仓库，有两个地址记录：

```bash
origin git@github.com:jellyfishmix/learngit.git (fetch)
origin git@github.com:jellyfishmix/learngit.git (push)
```

一个用于fetch，一个用于push。

### git删除此本地仓库已关联的远程仓库地址

```bash
git remote remove origin
```

origin为本地仓库想要删除关联的远程仓库地址，可以按实际需要替换。



## 添加文件追踪

```bash
// 对当前文件夹下所有文件添加追踪
git add .
// 对指定文件添加追踪
git add ./<filename>
```



## 提交到本地

```bash
git commit -m "第一次提交"
```



## 取消文件跟踪

对所有文件取消跟踪：

```bash
// 不删除本地文件
git rm -r --cached .
// 删除本地文件
git rm -r --f .
```

对某个文件取消跟踪：

```bash
// 删除readme1.txt的跟踪，并保留在本地。
git rm --cached readme1.txt
// 删除readme1.txt的跟踪，并且删除本地文件。
git rm --f readme1.txt
```



## 撤销最近的一次commit（尚未push）

```bash
git reset --soft HEAD^
```

这样就成功撤销了最近一次的commit。

仅是撤回commit操作，所写的代码仍然保留。



HEAD^的意思是上一个版本，也可以写成`HEAD~1`。

如果进行了2次commit，想都撤回，可以使用`HEAD~2`。



几个参数：

- mixed（默认）

  意思是：不删除工作空间改动代码，撤销commit，并且撤销`git add .`。

  操作`git reset --mixed HEAD^`和`git reset HEAD^`效果是一样的。

- soft  

  不删除工作空间改动代码，撤销commit，不撤销`git add . `。

- hard

  删除工作空间改动代码，撤销commit，撤销`git add . `。

  完成这个操作后，就恢复到了上一次的commit状态。



如果只想修改commit的注释：

```bash
git commit --amend
```

此时会进入默认vim编辑器，修改注释完毕后保存就好了。



## 切换分支

查看远程分支

```bash
git branch -a 
```

```
~/mxnet$ git branch -a
* master
  remotes/origin/HEAD -> origin/master
  remotes/origin/master
  remotes/origin/nnvm
  remotes/origin/piiswrong-patch-1
  remotes/origin/v0.9rc1
```

查看本地分支

```bash
git branch
```

```
~/mxnet$ git branch
* master
```

切换本地分支

```bash
git checkout master
```

```
＃ 切换为develop分支
$ git checkout develop
Switched to branch 'develop'
Your branch is up-to-date with 'origin/develop'.
```



## Git工作流程

1. 对代码进行修改。
2. 完成了某项功能，提交（commit，只是提交到本地代码库），1-2可以反复进行，直到觉得可以推送到服务器上时，执行3。
3. 拉取（pull，或者用获取 fetch 然后再手动合并 merge）。
4. 如果存在冲突，解决冲突。
5. 推送（push），将数据提交到服务器上的代码库。



## Github开源项目贡献代码流程

1. 登录 [https://github.com](https://github.com/)。
2. cFork `git@github.com:gpake/qiniu-wxapp-sdk.git`。
3. 创建您的特性分支 (git checkout -b new-feature)。
4. 提交您的改动 (git commit -am 'Added some features or fixed a bug')。
5. 将您的改动记录提交到远程 git 仓库 (git push origin new-feature)。
6. 然后到 github 网站的该 git 远程仓库的 new-feature 分支下发起 Pull Request。



## git 修改用户名和邮箱

用户名和邮箱地址是本地git客户端的一个变量，不随git库而改变。

每次commit都会用用户名和邮箱纪录。

### 查看用户名和地址

```css
git config user.name
git config user.email
```

### 修改用户名和地址

```csharp
git config --global user.name "your name"
git config --global user.email "your email"
```



## 引用

[git使用情景2：commit之后，想撤销commit](https://blog.csdn.net/w958796636/article/details/53611133)