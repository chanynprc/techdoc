## Tools Basic

### Git

从源代码安装git

```bash
yum remove git

wget https://mirrors.edge.kernel.org/pub/software/scm/git/git-2.42.0.tar.gz --no-check-certificate
tar zxvf git-2.42.0.tar.gz
cd git-2.42.0

# yum install expat-devel
make -j prefix=/usr/local/git all
make -j prefix=/usr/local/git install

ln -s /usr/local/git/bin/git /usr/local/bin/

git --version
```

同步远程Tag（先删除所有Tag，再进行同步）

```bash
git tag -l | xargs git tag -d
git fetch <repo_set_name>
```

拉取远程Tag

```bash
git fetch origin --tags
```

列出本地Tag

```bash
git tag
```

从Tag新建分支

```bash
git checkout -b new-branch-name tags/v1.0.0
```

查看本地分支和追踪情况

```bash
git remote show origin
```

删除已被远程删除的分支

```bash
git remote prune origin
```

修改最后一次提交的作者信息

```bash
git commit --amend --reset-author
```

### SVN

清除SVN未被版本控制的文件

``` bash
svn st | grep '^?' | awk '{print $2}' | xargs rm -rf
```

1. 第一个命令执行svn status
2. 第二个命令查找?开头的行，没有加入版本控制的文件或目录开头显示?号
3. 第三个命令获得第二个参数，是带路径的文件或目录名
4. 第四个命令删除它



