# Linux - VNC的安装、配置与使用

转载于 https://www.cnblogs.com/anliven/p/6847762.html

## VNC简介

VNC（Virtual Network Computing）是一套实现远程网络操控的软件。
网络操控技术是指由一部计算机（主控端）去控制另一部计算机（被控端），而且当主控端在操作时，就如同亲自在被控端前操作一样，可以执行被控端的应用程序，及使用被控端的系统资源。

VNC软件主要由两个部分组成： VNC Server及VNC Viewer Client。

- 用户需先将VNC Server安装在被控端的计算机上，才能在主控端执行VNC Viewer Client控制被控端。
- VNC Server与VNC Viewer Client支持多种操作系统，如Unix系列、Windows及MacOS，因此可将VNC Server及VNC Viewer Client分别安装在不同的操作系统中进行控制。
- 如果目前操作的主控端计算机没有安装VNC Viewer Client，也可以通过一般的网页浏览器来访问，进而控制被控端。

整个VNC运行的工作流程

1. VNC客户端通过浏览器或VNC Viewer Client连接至VNC Server。
2. VNC Server传送一个对话窗口至客户端，要求输入连接密码，以及存取的VNC Server显示装置。
3. 在客户端输入联机密码后，VNC Server验证客户端是否具有存取权限。
4. 通过VNC Server的验证后，客户端将立即要求VNC Server显示桌面环境。
5. VNC Server将获得的桌面环境利用VNC通信协议送至客户端，并且允许客户端控制VNC Server的桌面环境及输入装置。



## Free VNC tools

TigerVNC

- Homepage：<http://tigervnc.org/>
- Download：<https://bintray.com/tigervnc/stable/tigervnc/>

TightVNC

- Homepage：<http://www.tightvnc.com/>
- Download：<https://www.tightvnc.com/download.php>

RealVNC的VNC Viewer Client

- <https://www.realvnc.com/en/connect/download/viewer/>



## 示例 - 在CentOS7.5中安装和配置VNC Server

### 1-确认系统环境

```
[root@localhost ~]# uname -a
Linux localhost.localdomain 3.10.0-957.el7.x86_64 #1 SMP Thu Nov 8 23:39:32 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
[root@localhost ~]# cat /etc/system-release
CentOS Linux release 7.5.1804 (Core) 
```

### 2-安装VNC Server

确认是否安装了VNC Server，如果没有安装，那么找到对应的rpm包进行安装。

```
yum -y install tigervnc-server tigervnc  # 安装VNC Server及Viewer
```

安装完成：

```
[root@localhost ~]# rpm -qa |grep vnc
tigervnc-license-1.8.0-5.el7.noarch
tigervnc-1.8.0-13.el7.x86_64
tigervnc-icons-1.8.0-13.el7.noarch
tigervnc-server-minimal-1.8.0-5.el7.x86_64
gtk-vnc2-0.7.0-3.el7.x86_64
tigervnc-server-1.8.0-13.el7.x86_64
gvnc-0.7.0-3.el7.x86_64
[root@localhost ~]# 
```

### 3-配置VNC Server

VNC Server支持多种配置：

- 单用户单界面配置（一个用户访问，使用一个界面）
- 多用户单界面配置（多个用户访问，使用同一个界面）
- 多用户多界面配置（多个用户访问，使用各自的界面）

#### 创建和修改配置文件（单用户单界面）

```
[root@localhost ~]# cp /usr/lib/systemd/system/vncserver@.service /etc/systemd/system/vncserver@.service
[root@localhost ~]# ll /etc/systemd/system/vncserver@.service
-rw-r--r-- 1 root root 1828 Jun 21 11:08 /etc/systemd/system/vncserver@.service
[root@localhost ~]# 
[root@localhost ~]# vim /etc/systemd/system/vncserver@.service
[root@localhost ~]# cat /etc/systemd/system/vncserver@.service  |grep -v "#"


[Unit]
Description=Remote desktop service (VNC)
After=syslog.target network.target

[Service]
Type=forking
User=root

ExecStartPre=/bin/sh -c '/usr/bin/vncserver -kill %i > /dev/null 2>&1 || :'
ExecStart=/usr/sbin/runuser -l Anliven -c "/usr/bin/vncserver %i -geometry 1280x1024"
PIDFile=/home/Anliven/.vnc/%H%i.pid
ExecStop=/bin/sh -c '/usr/bin/vncserver -kill %i > /dev/null 2>&1 || :'

[Install]
WantedBy=multi-user.target
[root@localhost ~]# 
[root@localhost ~]# systemctl daemon-reload
[root@localhost ~]# 
```

注意

- 启动服务的用户设置为root，这样VNC Client访问时可以看到菜单栏(Menu bar)
- 用户使用的是Anliven，联结时将登陆到Anliven的界面

本例只介绍单用户单界面配置。
针对多用户单界面方式（多用户同时访问），只需要将上面“vncserver@:1.service”改为“vncserver@:2.service”，并配置用户名、分辨率等参数，然后再完成以下步骤即可。
如果想配置为多用户多界面，请参考**[官方信息](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/system_administrators_guide/ch-tigervnc#masthead)！**

设置用户密码，这里定义在配置文件里的用户是Anliven。
注意：此步骤需要切换用户。

```
[root@localhost ~]# su - Anliven
Last login: Fri Jun 21 11:25:07 CST 2019 on pts/0
[Anliven@localhost ~]$ vncpasswd
Password:
Verify:
Would you like to enter a view-only password (y/n)? n  
A view-only password is not used
[Anliven@localhost ~]$ 
```

#### 启动VNC服务并设置为自动启动

```
[root@localhost ~]# systemctl start vncserver@:1.service
[root@localhost ~]# systemctl enable vncserver@:1.service
Created symlink from /etc/systemd/system/multi-user.target.wants/vncserver@:1.service to /etc/systemd/system/vncserver@.service.
[root@localhost ~]# 
```

#### 查看VNC服务状态

```
[root@localhost ~]# systemctl status vncserver@:1.service -l
● vncserver@:1.service - Remote desktop service (VNC)
   Loaded: loaded (/etc/systemd/system/vncserver@.service; enabled; vendor preset: disabled)
   Active: active (running) since Fri 2019-06-21 13:15:55 CST; 18min ago
 Main PID: 6556 (Xvnc)
   CGroup: /system.slice/system-vncserver.slice/vncserver@:1.service
           ‣ 6556 /usr/bin/Xvnc :1 -auth /home/Anliven/.Xauthority -desktop localhost.localdomain:1 (Anliven) -fp catalogue:/etc/X11/fontpath.d -geometry 1280x1024 -pn -rfbauth /home/Anliven/.vnc/passwd -rfbport 5901 -rfbwait 30000

Jun 21 13:15:51 localhost.localdomain systemd[1]: Starting Remote desktop service (VNC)...
Jun 21 13:15:55 localhost.localdomain systemd[1]: Started Remote desktop service (VNC).
[root@localhost ~]# 
```

#### 设置防火墙

方式1：关闭防火墙并设置为不自动启动

```
systemctl stop firewalld && systemctl disable firewalld
```

判断firewalld是否启动：

```
[root@localhost ~]# firewall-cmd --state
not running
```

方式2：在防火墙中打开VNC服务

```
$ firewall-cmd --permanent --add-service vnc-server
success
$ firewall-cmd --reload
```

### 5-其他信息

用户配置文件

```
[Anliven@localhost ~]$ ll -a .vnc
total 28
drwxrwxr-x   2 Anliven Anliven  120 Jun 21 13:15 .
drwx------. 20 Anliven Anliven 4096 Jun 21 13:44 ..
-rw-r--r--   1 Anliven Anliven  332 Jun 21 13:15 config
-rw-rw-r--   1 Anliven Anliven 4619 Jun 21 13:41 localhost.localdomain:1.log
-rw-rw-r--   1 Anliven Anliven    5 Jun 21 13:15 localhost.localdomain:1.pid
-rw-------   1 Anliven Anliven    8 Jun 21 11:26 passwd
-rwxr-xr-x   1 Anliven Anliven  112 Jun 21 13:15 xstartup
[Anliven@localhost ~]$ 
```

一些命令

```
[Anliven@localhost ~]$ vncserver -h

usage: vncserver [:<number>] [-name <desktop-name>] [-depth <depth>]
                 [-geometry <width>x<height>]
                 [-pixelformat rgbNNN|bgrNNN]
                 [-fp <font-path>]
                 [-cc <visual>]
                 [-fg]
                 [-autokill]
                 [-noxstartup]
                 [-xstartup <file>]
                 <Xvnc-options>...

       vncserver -kill <X-display>

       vncserver -list

[Anliven@localhost ~]$ 
[Anliven@localhost ~]$ vncpasswd -h
usage: vncpasswd [file]
       vncpasswd -f
[Anliven@localhost ~]$ 
```

停止VNC服务进程

```
[Anliven@localhost ~]$ vncserver -list

TigerVNC server sessions:

X DISPLAY #	PROCESS ID
:1		6556
[Anliven@localhost ~]$ 
[Anliven@localhost ~]$ vncserver -kill :1
Killing Xvnc process ID 6556
[Anliven@localhost ~]$ 
[Anliven@localhost ~]$ vncserver -list

TigerVNC server sessions:

X DISPLAY #	PROCESS ID
[Anliven@localhost ~]$ 
```



## 使用VNC Viewer Client进行远程连接

- 在Linux下，运行vncviewer命令即可，服务器地址的写法形如192.168.1.11:1
- 在Windows下，运行windows版本的vncviewer客户端程序即可，用法与linux下相近。
- 用浏览器也能够进行远程连接，但可能会涉及较为繁琐的配置，一般不使用此方式。

### 确认vncserver是否运行

```
[Anliven@localhost ~]$ vncserver -list

TigerVNC server sessions:

X DISPLAY #	PROCESS ID
:1		1390
[Anliven@localhost ~]$ 
```

### 使用Tiger VNC Viewer来连接

Download - Windows-64bit: <https://bintray.com/tigervnc/stable/download_file?file_path=vncviewer64-1.9.0.exe>
![img](https://img2018.cnblogs.com/blog/819128/201906/819128-20190624224445217-481611814.png)
![img](https://img2018.cnblogs.com/blog/819128/201906/819128-20190624224450954-126820465.png)

[](https://www.cnblogs.com/anliven/p/6847762.html#_labelTop)

## 问题处理

### 问题-1

#### 问题现象：启动vncserver服务报错

```
[root@localhost ~]# systemctl start vncserver@:1.service
Job for vncserver@:1.service failed because the control process exited with error code. See "systemctl status vncserver@:1.service" and "journalctl -xe" for details.
[root@localhost ~]# 
[root@localhost ~]# systemctl status vncserver@:1.service
● vncserver@:1.service - Remote desktop service (VNC)
   Loaded: loaded (/etc/systemd/system/vncserver@.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Fri 2019-06-21 13:07:28 CST; 2min 14s ago
  Process: 6104 ExecStart=/usr/sbin/runuser -l Anliven -c /usr/bin/vncserver %i -geomotry 1280x1024 (code=exited, status=255)
  Process: 6095 ExecStartPre=/bin/sh -c /usr/bin/vncserver -kill %i > /dev/null 2>&1 || : (code=exited, status=0/SUCCESS)

Jun 21 13:07:28 localhost.localdomain runuser[6104]: must supply to access the server (default=)
Jun 21 13:07:28 localhost.localdomain runuser[6104]: rfbauth        - Alias for PasswordFile
Jun 21 13:07:28 localhost.localdomain runuser[6104]: PasswordFile   - Password file for VNC authentication (default=)
Jun 21 13:07:28 localhost.localdomain runuser[6104]: SecurityTypes  - Specify which security scheme to use (None, VncAuth, Plain,
Jun 21 13:07:28 localhost.localdomain runuser[6104]: TLSNone, TLSVnc, TLSPlain, X509None, X509Vnc, X509Plain)
Jun 21 13:07:28 localhost.localdomain runuser[6104]: (default=TLSVnc,VncAuth)
Jun 21 13:07:28 localhost.localdomain runuser[6104]: (EE)
Jun 21 13:07:28 localhost.localdomain runuser[6104]: Fatal server error:
Jun 21 13:07:28 localhost.localdomain runuser[6104]: (EE) Unrecognized option: -geomotry
Jun 21 13:07:28 localhost.localdomain runuser[6104]: (EE)
[root@localhost ~]# 
```

#### 问题处理

根据报错信息“Unrecognized option: -geomotry”，确认为配置参数错误，更改为正确配置“-geometry”即可。
删除相关目录文件：`rm -rf /tmp/.X11-unix/`。

### 问题-2

#### 问题现象

Tiger VNC Viewer连接CentOS7.5系统的VNC Server时,提示信息:

```
Authentication Required
Authentication is required to create a color profile.
```

#### 问题处理

在`/etc/polkit-1/rules.d/`添加对应的rules文件.

```
[Anliven@localhost ~]$ su - root
Password: 
Last login: Mon Jun 24 10:36:25 CST 2019 on pts/0
[root@localhost ~]#
[root@localhost ~]# cd /etc/polkit-1/rules.d/
[root@localhost rules.d]# ll
total 12
-rw-r--r--. 1 root root 974 Jun 10  2014 49-polkit-pkla-compat.rules
-rw-r--r--. 1 root root 326 Apr 30  2013 50-default.rules
-rw-r--r--  1 root root 547 Jun 24 10:50 gnome-vnc.rules
[root@localhost rules.d]# cat gnome-vnc.rules 
polkit.addRule(function(action, subject) {
   if ((action.id == "org.freedesktop.color-manager.create-device" ||
        action.id == "org.freedesktop.color-manager.create-profile" ||
        action.id == "org.freedesktop.color-manager.delete-device" ||
        action.id == "org.freedesktop.color-manager.delete-profile" ||
        action.id == "org.freedesktop.color-manager.modify-device" ||
        action.id == "org.freedesktop.color-manager.modify-profile") &&
       subject.isInGroup("Anliven")) {
      return polkit.Result.YES;
   }
});
[root@localhost rules.d]# 
```

### 问题处理参考

- [https://wiki.archlinux.org/index.php/TigerVNC#"Authentication_is_required_to_create_a_color_managed_device"_dialog_when_launching_GNOME_3](https://wiki.archlinux.org/index.php/TigerVNC#%22Authentication_is_required_to_create_a_color_managed_device%22_dialog_when_launching_GNOME_3)
- <https://unix.stackexchange.com/questions/417906/authentication-is-required-to-create-a-color-profile>
- <https://c-nergy.be/blog/?p=12073>



## 示例：在RedHat Linux 5企业版中配置VNCSREVER

### 确认及安装 VNC Server

首先确认服务器是否配置了VNCSERVER：

```
[root@localhost: ~]#rpm -qa |grep vnc 
vnc-4.1.2-14.el5
vnc-server-4.1.2-14.el5
```

返回VNCSEVER服务器端版本说明已经安装了 VNCSERVER。
如果没有安装VNCSEVER，那么找到安装包进行安装。

### 配置 VNCSERVER

第一次启动VNCSERVER提示输入密码，这里分为管理员账户及普通账户，启动方式略有所不同。
管理员：

```
[root@localhost /]#   VNC Server 
You will require a password to access your desktops. 
Password: 123456            #输入vnc 连接密码 
Verify: 123456                 #确认vnc密码 
xauth: creating new authority file /root/.Xauthority 
New ‘localhost.localdomain:1 (root)’ desktop is localhost.localdomain:1 
Creating default startup script /root/.vnc/xstartup 
Starting applications specified in /root/.vnc/xstartup 
Log file is /root/.vnc/localhost.localdomain:1.log 
```

普通用户：

```
[root@localhost /]#su ceboy    #ceboy 是用户名 
[ceboy@localhost /]$   VNC Server 
You will require a password to access your desktops. 
Password: 123456            #输入vnc 连接密码 
Verify: 123456                 #确认vnc密码 
xauth: creating new authority file /home/ceboy/.Xauthority 
New ‘localhost.localdomain:2 (ceboy)’ desktop is localhost.localdomain:2 
Creating default startup script /home/ceboy/.vnc/xstartup 
Starting applications specified in /home/ceboy/.vnc/xstartup 
Log file is /home/ceboy/.vnc/localhost.localdomain:2.log 
```

注意：

- 每个用户都可以启动自己的 VNCSERVER远程桌面，同时每个用户可以启动多个 VNCSERVER远程桌面，它们用ip加端口号：ip:1、ip:2、ip:3 来标识区分，使用同一端口会使另外登录的用户自动退出。
- VNCSERVER的大部分配置文件及日志文件都在用户home目录下.vnc目录下。

用户可以自定义启动号码如：

```
[ceboy@localhost /]$   VNC Server :2        #注意:2前面一定要有空格。 
A   VNC Server is already running as :2 
```

### 相关桌面配置

#### gnome桌面

RedHat Linux支持两种图形模式：KDE模式和gnome模式。
通过`ps -A`命令列出所有当前运行的程序，以KDE或者gnome字段来判断
如果是gnome桌面，那么需要修改`/root/.vnc/xstartup`的配置文件。

```
[root@localhost .vnc]# vi xstartup 
#!/bin/sh 
# Uncomment the following two lines for normal desktop: 
# unset SESSION_MANAGER        #将此行的注释去掉 
# exec /etc/X11/xinit/xinitrc        #将此行的注释去掉 
[ -x /etc/vnc/xstartup ] && exec /etc/vnc/xstartup 
[ -r $HOME/.Xresources ] && xrdb $HOME/.Xresources 
xsetroot -solid grey 
vncconfig -iconic & 
xterm -geometry 80×24+10+10 -ls -title “$VNCDESKTOP Desktop” & 
gnome-session gnome           #添加这一句是连接时使用gnome 桌面环境 
twm & 
```

设置修改完毕最好是重启一次，否则设置不会生效。
采用的方法是杀死 VNCSERVER进程再重运行 VNCSERVER。

```
[root@localhost .vnc]#  VNC Server -kill :1      #1是启动  VNC Server时的端口号 
[root@localhost .vnc]#  VNC Server :1           #重启，注意:1前面一定要有空格。
```

#### 设置用户信息及分辨率

修改配置文件`/etc/sysconfig/VNC Servers`

```
[root@localhost: ~]#vi /etc/sysconfig/VNC Servers 
# The   VNCSERVERS variable is a list of display:user pairs. 
# 
# Uncomment the lines below to start a   VNC Server on display :2 
# as my ‘myusername’ (adjust this to your own). You will also 
# need to set a VNC password; run ‘man vncpasswd’ to see how 
# to do that. 
# 
# DO NOT RUN THIS SERVICE if your local area network is 
# untrusted! For a secure way of using VNC, see 
# <URL:http://www.uk.research.att.com/archive/vnc/sshvnc.html >. 
# Use “-nolisten tcp” to prevent X connections to your   VNC Server via TCP. 
# Use “-nohttpd” to prevent web-based VNC clients connecting. 
# Use “-localhost” to prevent remote VNC clients connecting except when 
# doing so through a secure tunnel. See the “-via” option in the 
# `man vncviewer’ manual page. 
VNCSERVERS=”1:root 2:ceboy”           #此处已添加两个用户root和cebo。 
VNCSERVERARGS[1]=”-geometry 800×600 -nolisten tcp -nohttpd -localhost”  VNCSERVERARGS[2]=”-geometry 1024×768 -nolisten tcp -nohttpd -localhost” 
```

注意：
上面是分别设置的root和ceboy两个用户的分辨率，注意是用端口号区分的。
也可通过命令行临时修改分辨率及色深，这种方式重启后就会丢失，命令如下：

```
[root@localhost: ~]#  VNC Server -geometry 800×600        #设置  VNC Server的分辨率    
[root@localhost: ~]#  VNC Server -depth 16           #设置  VNC Server的色深 
```

### 客户端连接

访问方式

- 在Linux下，运行vncviewer命令即可，服务器地址的写法形如192.168.1.11:1
- 在Windows下，运行windows版本的vncviewer客户端程序即可，用法与linux下相近。
- 用浏览器也能够进行远程连接，但可能会涉及较为繁琐的配置，一般不使用此方式。

### 常用操作

- 修改密码：运行vncpasswd即可。
- 停止 VNC Server：命令类似`VNC Server -kill :1` ，注意VNC Server只能由启动它的用户来关闭。
- 稳定性设置：VNC Server默认在多个客户机连接同一个VNC Server的显示端口时，会关闭VNC Server端口旧连接，而为新连接服务，可通过-dontdisconnect拒绝新连接请求而保持旧的连接。
- 同一个显示可以连接多个客户机：`VNC Server -alwaysshared`
- 重启服务：`service VNC Server restart`
- 随机启动VNCSERVER：使用VNC连接登录到RedHat Linux图形界面，点击“系统”——“管理”——“服务器设置”——“服务”，在“后台服务”中找到 VNCSERVER后勾选它，点击保存即可。



## VNC相关资源

- [Red Hat Training-CHAPTER 13. TIGERVNC](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/system_administrators_guide/ch-tigervnc#masthead)