## Perf Basic

### perf

**record**

```bash
perf record -e cpu-clock -g -p 5432
```

- -e：监控指标
- -g：记录函数调用关系
- -p：监控的进程pid

**report**

```bash
perf report -i perf.data
```

- -i：指定输入文件，由perf record输出

**top**

```
perf top
```

### schbench

URL：https://git.kernel.org/pub/scm/linux/kernel/git/mason/schbench.git/

**安装**

```bash
wget https://git.kernel.org/pub/scm/linux/kernel/git/mason/schbench.git/snapshot/schbench-1.0.tar.gz
```



### 火焰图



