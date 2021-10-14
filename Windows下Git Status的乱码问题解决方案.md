## Windows下Git Status的乱码问题解决方案

Windows下Git Bash的乱码问题很多，不过好在终于都解决了！


### 1. 问题一：

乱码如下：

	“\344\270\212\347\”

**解决：Bash下输入如下命令**

	git config --global core.quotepath false

注：此问题Msys和Cygwin都有



### 2. 问题二：

哪都不乱码`Git Status`显示中文文件名乱码。

**解决：**

打开Git Bash，右键标题栏选择“Options”。修改Text中的Locale为“C”，Character set为“UTF-8”，关闭Git Bash。

重新打开Git Bash，问题消失。

`注：`Cygwin无此问题，但是Cygwin下面默认不会打开autoCrlf，也不知道有没有其他针对Windows需要优化的地方，所以首选还是Msys（即Git For Windows）。


### 3. 问题三：

其他奇怪问题，比如Source Tree乱码，或者使用如上设置后，Git Log什么的乱码

**解决：**

	git config --global -e

打开配置文件后，删除里面所有`i18n`设置