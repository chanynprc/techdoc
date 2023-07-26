## Tools Basic

### Git

同步远程Tag（先删除所有Tag，再进行同步）

```bash
git tag -l | xargs git tag -d
git fetch
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

