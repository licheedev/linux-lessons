#  Shell命令总结
------
## **1. Top 25个Shell命令 from [commandlinefu][1]**

* **以root身份运行上一次命令**
```Shell
 $ sudo !!  
```
"!!" 获取上一次的命令，在你忘记运行命令时候特别有用

 * **在http://localhost:8000/访问当前目录树结构**
```Shell
$ python -m SimpleHTTPServer
```

* **切换到上一次工作目录**
```Shell
$ cd -
```

 * **用当前命令替换上一次运行命令**
```Shell
$ ^foo^bar
```
举个例子，输入`$lq -al` 显示 **lq: command not found**,再执行`$ ^lq^ls`,即详细的显示了当前目录

 * **更好地判断网络连通性命令**
```Shell
mtr google.com
```
 一般在windows中判断网络连通性结合使用ping和tracert，ping可判断丢包率，tracert可用来跟踪路由。


   

## **2. Top 25 Shell Tips from [Shell-fu.org][2]**
 * **智能处理：解压文件夹的Shell脚本**
```Shell
#! /bin/sh

## Handy Extract Program
## need install the software of the lists : bunzip2 unrar 
extract() {
	if [-f $1]; then 
		case $1 in 
			*.tar.bz2) tar xvjf $1 ;;
			*.tar.gz) tar xvzf $1 ;;
			*.bz2) bunzip2 $1 ;;
			*.rar) unrar x $1 ;;
			*.gz) gunzip $1 ;;
			*.tar) tar xvf $1 ;;
			*.tbz2) tar xvjf $1 ;;
			*.tgz) tar xvzf $1 ;;
			*.zip) unzip $1 ;;
			*.Z) uncompress $1 ;;
			*.7z) 7z x $1 ;;
			*) echo "'$1' cannot be extracted via >extract<" ;;
		esac
     else
		echo "'$1' is not a valid file"
     fi
}
```


  [1]: http://www.commandlinefu.com/
  [2]: http://www.shell-fu.org/index.php
