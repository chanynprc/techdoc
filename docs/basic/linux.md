## Linux Basic

<!--### Shell命令-->

### 监控工具

#### vmstat

用于监控进程、内存、内存交换空间、磁盘IO、系统事件、CPU

```bash
# 常用命令，宽模式显示，每隔1秒刷新一次
vmstat -w 1
```

字段含义：

| procs:<br/>有关进程的信息                                    | memory:<br/>内存使用情况，单位通常为KB                       | swap:<br/>交换分区的使用情况                                 | io:<br/>输入/输出统计                             | system:<br/>系统事件                                         | cpu:<br/>CPU使用情况的摘要                                   |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| r:<br/>等待运行的进程数<br/>b:<br/>处于不可中断睡眠状态的进程数 | swpd:<br/>使用的虚拟内存大小<br/>free:<br/>空闲的物理内存大小<br/>buff:<br/>作为文件缓存的内存大小<br/>cache:<br/>作为缓存的内存大小（包括页缓存和Slab内存） | si:<br/>从磁盘交换进内存的数据量<br/> so:<br/>从内存交换到磁盘的数据量 | bi:<br/>每秒读取的块数<br/>bo:<br/>每秒写入的块数 | in:<br/>每秒中断次数，包括时钟中断<br/>cs:<br/>每秒上下文切换次数 | us:<br/>用户态时间百分比<br/>sy:<br/>系统态时间百分比<br/>id:<br/>空闲时间百分比<br/>wa:<br/>等待I/O的时间百分比<br/>st:<br/>被偷走的时间百分比（在虚拟化环境中） |

#### dstat

### 用户管理

#### 添加用户

``` bash
useradd -d /home/username -m username
passwd username
```

其中，`-d`指定该用户的根目录，`-m`表示创建该用户根目录

#### 删除用户

``` bash
userdel username
```

#### CentOS给普通用户加sudo权限

使用root修改/etc/sudoers文件，在`root    ALL=(ALL)       ALL`后添加：

``` bash
root    ALL=(ALL)       ALL
newuser     ALL=(ALL)       ALL
```

### 网络配置

#### 建立SSH信任关系

（1）在A机器生成公私钥对

```bash
ssh_keygen
```

生成过程中会询问公私钥文件位置，默认为`~/.ssh`文件夹下的`id_rsa`和`id_rsa.pub`，然后询问是否需要密码，不输入表示不需要密码。

（2）将刚刚生成的公钥文件拷贝到B机器上，并且添加到B机器的SSH信任公钥列表中

```bash
cat id_rsa.pub >>  ~/.ssh/authorized_keys
```

这样就建立了A机器访问B机器的信任关系，这个信任是有方向的，若要建立B机器访问A机器的信任关系，需要逆向重复上述步骤。

建立好SSH信任关系后，就可以免密（如果第一步没设置密码）SSH登录或SCP拷贝文件了。

### 磁盘/存储管理

#### 查看硬盘剩余空间和文件夹占用空间大小

（1）查看硬盘剩余空间

```shell
df -lh
```

（2）查看某文件夹占用空间大小

```shell
du -sh <folder>
du -sh *
```

#### 查看磁盘挂载情况

```shell
# 树状图
lsblk

# 详细信息
fdisk -l
```

#### 挂载磁盘

```shell
# 创建分区
fdisk /dev/vdc
> n (n: add a new partition)
> w (w: write table to disk and exit)

# 做Raid0
mdadm -Cv /dev/md0 -l0 -n16 /dev/vd[bcdefghijklmnopq]1

# 格式化
mkfs.ext4 /dev/md0

# 挂载
mkdir /disk1
mount /dev/md0 /disk1

# 持久化（1、未验证。2、如果是挂载一个已有磁盘，是否直接执行此步骤就可以？）
echo `blkid /dev/vdc | awk '{print $2}' | sed 's/\"//g'` /disk1 ext4 defaults 0 0 >> /etc/fstab
```

#### 挂载ISO文件

```bash
cd /mnt
mkdir iso
mount -o loop -t iso9660 isofile.iso /mnt/iso
```

#### VirtualBox挂载共享文件夹

首先需要安装VirtualBox Guest Additions，安装过程中可能需要kernal-devel。如果需要创建软连接，需要额外的命令允许。

```bash
# 查找VirtualBox Guest Additions光盘
ls -l /dev | grep cdrom

# 挂载
mount /dev/cdrom /mnt/

# 升级内核（如有必要）
yum update kernel -y
yum install kernel-headers kernel-devel gcc make -y

# 安装（可能失败，需安装kernel-devel）
sh VBoxLinuxAdditions.run

# 允许创建软链接（需重启VirtualBox）
VBoxManage setextradata <vm_name> VBoxInternal2/SharedFoldersEnableSymlinksCreate/<share_name> 1
```

挂载共享文件夹

```bash
mount -t vboxsf -o uid=1000,gid=1000 workspaces /home/cyn/workspaces
```

其中uid和gid分别为普通用户的User ID和Group ID。

### 文件管理

#### 查找包含字符串的文件

（1）当前目录下的所有文件中的字符串

```bash
grep -r "zh_CN" ./
```

（2）当前目录下的以pre开头的文件中的字符串

```bash
grep -r "zh_CN" ./pre*
```

#### 查找文件

```bash
find . -name filename
find /home/user/downloads -iname filename
```

`-name`指定要查找的文件名，可模糊匹配，如：可利用file*来模糊匹配以file开头的文件。

`-iname`指定要查找的文件名，但忽略大小写。

#### 查看文件的编码格式

在vim中可以查看文件的编码格式：

```
:set fileencoding
```

### 软件管理

#### 查看已安装的包

```bash
rpm -qa | grep apr
```

#### 升级yum源

```bash
# http://dl.fedoraproject.org/pub/epel/7/aarch64/
wget https://dl.fedoraproject.org/pub/epel/7/aarch64/Packages/e/epel-release-7-12.noarch.rpm
rpm -Uvh epel-release-7-12.noarch.rpm
```

### 系统管理

#### Linux软件包操作

Ubuntu：

```bash
# 列出已安装的包
dpkg -l | grep postgres
```

#### 修改系统时间

（1）修改系统时间

```bash
date -s 14:36:53
date -s 08/28/2008
```

（2）同步系统时间到硬件时间

```bash
hwclock -w
hwclock --systohc
```

（3）同步硬件时间到系统时间

```bash
hwclock --hctosys
```

### 清除SVN未被版本控制的文件

``` bash
svn st | grep '^?' | awk '{print $2}' | xargs rm -rf
```

1. 第一个命令执行svn status
2. 第二个命令查找?开头的行，没有加入版本控制的文件或目录开头显示?号
3. 第三个命令获得第二个参数，是带路径的文件或目录名
4. 第四个命令删除它

### 引用

[1] http://www.cnblogs.com/sunleecn/archive/2011/11/01/2232210.html

[2] http://blog.csdn.net/caz28/article/details/50246951

[3] http://www.cnblogs.com/daizhuacai/archive/2013/01/17/2865132.html 

[4] http://blog.csdn.net/nfer_zhuang/article/details/42646849
