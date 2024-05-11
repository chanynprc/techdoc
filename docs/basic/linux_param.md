## Linux系统参数

### tcp keepalive

在Linux系统中，net.ipv4.tcp_keepalive系列参数是与TCP保活（keepalive）机制相关的一组内核参数。TCP保活机制用于检测并保持活跃的TCP连接，特别是在无数据传输的长期连接情况下。同时，也有助于识别已经停止响应的对端（服务器或客户端），并在必要时终止连接。修改keepalive参数可以对于需要维持长期连接的应用有帮助，如数据库连接、网络共享等，并且能防止连接在出现网络问题时一直挂起。

| 参数                          | 参数含义                                                     |
| ----------------------------- | ------------------------------------------------------------ |
| net.ipv4.tcp_keepalive_time   | 当TCP连接在多少秒后没有任何数据传输时，才发送TCP keepalive探测包<br/>单位：秒<br/>默认值：7200（7200秒，2小时） |
| net.ipv4.tcp_keepalive_intvl  | 在前一个TCP keepalive探测未被应答时，系统是在多少秒后再次发送探测包<br/>单位：秒<br/>默认值：75（75秒） |
| net.ipv4.tcp_keepalive_probes | 在认定TCP连接为“死链接”之前，发送多少个TCP keepalive探测包。如果对端在此期间仍未应答，则系统认为连接已断开<br/>单位：次<br/>默认值：9（9次） |

![](/techdoc/docs/basic/images/tcpkeepalive1.png)

在正常情况下，当TCP连接在tcp_keepalive_time后没有任何数据传输时，会发送keepalive探测包，并且一直发送下去。

![](/techdoc/docs/basic/images/tcpkeepalive2.png)

在出现网络异常情况下，当TCP连接在tcp_keepalive_time秒后没有任何数据传输时，会发送第1个keepalive探测包，这时候对端无应答，系统是在tcp_keepalive_intvl秒后再次发送探测包，直到发送tcp_keepalive_probes次，如果对端一直无应答，则系统认为连接已断开。

```bash
# 临时修改
sysctl -w net.ipv4.tcp_keepalive_time=7200
sysctl -w net.ipv4.tcp_keepalive_intvl=75
sysctl -w net.ipv4.tcp_keepalive_probes=9

# 永久修改
vim /etc/sysctl.conf
sysctl -p /etc/sysctl.conf
```



