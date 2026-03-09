## P0面试优先级（2026）
- `P0-1` 进程排障：`ps`/`top`/`pidstat`/`jstack` 联合定位。
- `P0-2` 网络排障：`ss`/`netstat`/`tcpdump`/`curl`。
- `P0-3` 资源监控：CPU、内存、IO、负载与上下文切换。
- `P0-4` 日志与文件：`tail`/`grep`/`awk`/`sed` 常用组合。
- `P0-5` 服务管理：`systemctl` 启停、开机自启、状态诊断。

## P0常见误区修正（本页已修订）
- `load average` 不是 CPU 使用率，需结合 CPU 核数判断。
- 只看 `top` 不够，要结合 `iostat/vmstat/sar` 做归因。
- 线上排障要优先无损命令，避免高开销全量抓取。

# 常见命令
## 进程类

> Service ( Centos6)

```
service服务名start
service服务名stop
service服务名restart
service服务名reload
service服务名status

#查看服务的方法 /etc/init.d/ 服务名
#通过 chkconfig 命令设置自启动
#查看服务 chkconfig --list | grep xxx

chkconfig --level 5 服务名 on

```

> Systemctl ( Centos7 )

```
systemctl start 服务名(xxx.service)
systemctl restart 服务名(xxxx.service)
systemctl stop 服务名(xxxx.service)
systemctl reload 服务名(xxxx.service)
systemctl status 服务名(xxxx.service)

#查看服务的方法 /usr/lib/systemd/system
#查看服务的命令

systemctl list-unit-files
systemctl --type service

#通过systemctl命令设置自启动

systemctl enable service_name
systemctl disable service_name
```