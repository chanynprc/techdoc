## Linux Basic

<!--### Shell命令-->

### 在目录下查找包含字符串的文件

当前目录下的所有文件中的字符串

```bash
grep -r "zh_CN" ./
```

当前目录下的以pre开头的文件中的字符串

```bash
grep -r "zh_CN" ./pre*
```

### 查找文件

```bash
find . -name filename
find /home/user/downloads -iname filename
```

```-name```指定要查找的文件名，可模糊匹配，如：可利用file*来模糊匹配以file开头的文件。

```-iname```指定要查找的文件名，但忽略大小写。

### 清除SVN未被版本控制的文件

``` bash
svn st | grep '^?' | awk '{print $2}' | xargs rm -rf
```

1. 第一个命令执行svn status
2. 第二个命令查找?开头的行，没有加入版本控制的文件或目录开头显示?号
3. 第三个命令获得第二个参数，是带路径的文件或目录名
4. 第四个命令删除它

### 系统用户操作

#### 添加用户

``` bash
useradd -d /home/username -m username
passwd username
```

```-d```指定该用户的根目录，```-m```表示创建该用户根目录

#### 删除用户

``` bash
userdel username
```

### 引用

[1] http://www.cnblogs.com/sunleecn/archive/2011/11/01/2232210.html

[2] http://blog.csdn.net/caz28/article/details/50246951

[3] http://www.cnblogs.com/daizhuacai/archive/2013/01/17/2865132.html 
