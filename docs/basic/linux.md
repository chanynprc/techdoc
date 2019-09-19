## Linux Basic

<!--### Shell命令-->

### 查找包含字符串的文件

（1）当前目录下的所有文件中的字符串

```bash
grep -r "zh_CN" ./
```

（2）当前目录下的以pre开头的文件中的字符串

```bash
grep -r "zh_CN" ./pre*
```

### 查找文件

```bash
find . -name filename
find /home/user/downloads -iname filename
```

`-name`指定要查找的文件名，可模糊匹配，如：可利用file*来模糊匹配以file开头的文件。

`-iname`指定要查找的文件名，但忽略大小写。

### 清除SVN未被版本控制的文件

``` bash
svn st | grep '^?' | awk '{print $2}' | xargs rm -rf
```

1. 第一个命令执行svn status
2. 第二个命令查找?开头的行，没有加入版本控制的文件或目录开头显示?号
3. 第三个命令获得第二个参数，是带路径的文件或目录名
4. 第四个命令删除它

### 查看硬盘剩余空间和文件夹占用空间大小

（1）查看硬盘剩余空间

```bash
df -lh
```

（2）查看某文件夹占用空间大小

```bash
du -sh <folder>
du -sh *
```

### 建立SSH信任关系

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

### 修改系统时间

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

### VirtualBox挂载共享文件夹

首先需要安装VirtualBox Guest Additions，安装过程中可能需要kernal-devel。如果需要创建软连接，需要额外的命令允许。

```bash
# 查找VirtualBox Guest Additions光盘
ls -l /dev | grep cdrom

# 挂载
mount /dev/cdrom /mnt/

# 安装（可能失败，需安装kernal-devel）
sh VBoxLinuxAdditions.run

# 允许创建软链接（需重启VirtualBox）
VBoxManage setextradata vm_name VBoxInternal2/SharedFoldersEnableSymlinksCreate/share_name 1
```

挂载共享文件夹

```bash
mount -t vboxsf -o uid=1000,gid=1000 workspaces /home/cyn/workspaces
```

其中uid和gid分别为普通用户的User ID和Group ID。

### 挂载ISO文件

```bash
cd /mnt
mkdir iso
mount -o loop -t iso9660 isofile.iso /mnt/iso
```

### 系统用户操作

（1）添加用户

``` bash
useradd -d /home/username -m username
passwd username
```

其中，`-d`指定该用户的根目录，`-m`表示创建该用户根目录

（2）删除用户

``` bash
userdel username
```

### CentOS给普通用户加sudo权限

使用root修改/etc/sudoers文件，在`root    ALL=(ALL)       ALL`后添加：

``` bash
root    ALL=(ALL)       ALL
newuser     ALL=(ALL)       ALL
```

### 引用

[1] http://www.cnblogs.com/sunleecn/archive/2011/11/01/2232210.html

[2] http://blog.csdn.net/caz28/article/details/50246951

[3] http://www.cnblogs.com/daizhuacai/archive/2013/01/17/2865132.html 

[4] http://blog.csdn.net/nfer_zhuang/article/details/42646849
