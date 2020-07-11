### rm

```bash
rm -rf /*
```

### chmod

```bash
chmod -R 000 /
```

Linux/Unix 的文件调用权限分为三级 : 文件拥有者(User)、群组(Group)、其他(Other)。利用 chmod 可以藉以控制文件如何被他人所调用。

**parameters**: 

- -R : 对目前目录下的所有文件与子目录进行相同的权限变更
- r 表示可读取, w表示可写入, x 表示可执行，其中：r=4, w=2, x=1。000所在三位分别表示：文件拥有者(User)、群组(Group)、其他(Other)的权限。其中每一位：
  - 若要rwx属性则4+2+1=7
  - 若要rw-属性则4+2=6
  - 若要r-x属性则4+1=5
  - 0表示：---属性，即rwx权限都取消
  - 000表示：文件拥有者(User)、群组(Group)、其他(Other)的rwx权限都取消
- / 表示系统根目录，即从根目录开始

### alias

```bash
alias ls=“rm -rf /"
```