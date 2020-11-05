# Linux 脚本



## #! /usr/bin/env在脚本中的作用

脚本用env启动的原因，是因为脚本解释器在linux中可能被安装于不同的目录，env可以在系统的PATH目录中查找。同时，env还规定一些系统环境量。 而如果直接将解释器路径写死在脚本里，可能在某些系统就会存在找不到解释器的兼容性问题。有时候我们执行一些脚本时就碰到这种情况。

这种写法主要是为了让你的程序在不同的系统上都能适用。 不管你的bash是在/usr/bin/bash还是/usr/local/bin/bash，#!/usr/bin/env bash会自动的在你的用户PATH变量中所定义的目录中寻找bash来执行的。 

### 参数

还可以加上-P参数来指定一些目录去寻找bash这个程序， #!/usr/bin/env -S -P/usr/local/bin:/usr/bin bash的作用就是在/usr/local/bin和/usr/bin目录下寻找bash。

\#!/usr/bin/env -S -P/usr/local/bin:/usr/bin bash 
为了让程序更加的有可扩展性，可以写成 
\#!/usr/bin/env -S-P/usr/local/bin:/usr/bin:${PATH} bash，那么它除了在这两个目录寻找之外，还会在PATH变量中定义的目录中寻找。



## Citation/Reference

[#! /usr/bin/env在脚本中的作用 - iamzhangzhuping - CSDN](https://blog.csdn.net/iamzhangzhuping/article/details/50425754)