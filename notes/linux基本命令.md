# linux基本命令



## 防火墙操作（centos7）



检查防火墙：

```
systemctl status firewalld
```

查看端口：

```javascript
firewall-cmd --list-ports
```

开启端口（服务器重启后有效）：

```javascript
firewall-cmd --permanent --zone=public --add-port=8080/tcp 
```

重启防火墙（开启端口后一定要重启防火墙！）：

```javascript
systemctl reload firewalld
```

关闭端口（服务器重启后有效）：

```
firewall-cmd --permanent --remove-port=3000/tcp
```

关闭防火墙：

```
systemctl  stop   firewalld.service
```

开启防火墙：

```
systemctl  start   firewalld.service
```

禁止开机启动启动防火墙：

```
systemctl   disable   firewalld.service
```

## 显示端口，杀掉进程

显示 tcp，udp 的端口和进程：

```
netstat -tunlp
netstat -tunlp | grep 端口号
```

杀掉对应的进程：

```
kill -9 PID
```