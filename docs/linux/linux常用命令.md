## linux常用命令

### 防火墙操作

1.查看firewall服务状态

```bash
systemctl status firewalld
```

2.查看firewall的状态

```bash
firewall-cmd --state
```

3.查看哪些程序正在使用互联网

```bash
firewall-cmd --permanent --list-services ssh dhcpv6-client
```

4.开启、关闭、重启firewalld.service服务

```bash
# 开启
service firewalld start

# 关闭
service firewalld stop

# 重启
service firewalld restart
```

5.查看防火墙规则

```bash
firewall-cmd --list-all
```

6.查询、开放、关闭端口

```bash
# 查询端口是否放开
firewall-cmd --query-port=8080/tcp

# 开放80端口
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --permanent --add-port=8080-8085/tcp

# 移除端口
firewall-cmd --permanent --remove-port=8080/tcp

# 查看防火墙开放的端口
firewall-cmd --permanent --list-ports
```

7.重启防火墙（修改配置后）

firewall-cmd --reload