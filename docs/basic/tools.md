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

