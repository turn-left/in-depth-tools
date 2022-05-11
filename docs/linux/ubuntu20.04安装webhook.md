# ubuntu 20.04 安装webhook

转载于 https://www.jianshu.com/p/bb2351d9dd35

### webhook

[https://github.com/adnanh/webhook](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fadnanh%2Fwebhook)

### 安装

```bash
sudo apt-get install webhook
```



```
ubuntu@VM-100-15-ubuntu:~$ sudo apt-get install webhook
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following NEW packages will be installed:
  webhook
0 upgraded, 1 newly installed, 0 to remove and 104 not upgraded.
Need to get 1,825 kB of archives.
After this operation, 6,318 kB of additional disk space will be used.
Get:1 http://mirrors.tencentyun.com/ubuntu focal/universe amd64 webhook amd64 2.6.9-1 [1,825 kB]
Fetched 1,825 kB in 1s (2,482 kB/s)
Selecting previously unselected package webhook.
(Reading database ... 77181 files and directories currently installed.)
Preparing to unpack .../webhook_2.6.9-1_amd64.deb ...
Unpacking webhook (2.6.9-1) ...
Setting up webhook (2.6.9-1) ...
Created symlink /etc/systemd/system/multi-user.target.wants/webhook.service → /lib/systemd/system/webhook.service.
```

**帮助文件**

```
ubuntu@VM-100-15-ubuntu:~$ webhook -h
Usage of webhook:
  -cert string
        path to the HTTPS certificate pem file (default "cert.pem")
  -header value
        response header to return, specified in format name=value, use multiple times to set multiple headers
  -hooks value
        path to the json file containing defined hooks the webhook should serve, use multiple times to load from different files
  -hotreload
        watch hooks file for changes and reload them automatically
  -ip string
        ip the webhook should serve hooks on (default "0.0.0.0")
  -key string
        path to the HTTPS certificate private key pem file (default "key.pem")
  -nopanic
        do not panic if hooks cannot be loaded when webhook is not running in verbose mode
  -port int
        port the webhook should serve hooks on (default 9000)
  -secure
        use HTTPS instead of HTTP
  -template
        parse hooks file as a Go template
  -urlprefix string
        url prefix to use for served hooks (protocol://yourserver:port/PREFIX/:hook-id) (default "hooks")
  -verbose
        show verbose output
  -version
        display webhook version and quit
```

### 配置

创建hooks.json。如果访问该endpoint，则触发执行脚本/home/ubuntu/redeploy.sh。该范例没有传入参数。

```json
[
  {
     "id": "redeploy-webhook",
     "execute-command": "/home/ubuntu/redeploy.sh",
     "command-working-directory": "/home/ubuntu"
  }
]
```

创建**redeploy.sh**。该脚本创建一个文件。

```bash
#!/bin/bash

touch `date +%Y%m%H%M%S`
```

### 运行

**启动服务**

```bash
ubuntu@VM-100-15-ubuntu:~$ webhook -hooks hooks.json -verbose
[webhook] 2020/11/06 09:49:49 version 2.6.9 starting
[webhook] 2020/11/06 09:49:49 setting up os signal watcher
[webhook] 2020/11/06 09:49:49 attempting to load hooks from hooks.json
[webhook] 2020/11/06 09:49:49 found 1 hook(s) in file
[webhook] 2020/11/06 09:49:49   loaded: redeploy-webhook
[webhook] 2020/11/06 09:49:49 serving hooks on http://0.0.0.0:9000/hooks/{id}
[webhook] 2020/11/06 09:49:49 os signal watcher ready
```

### 测试

在另外一台服务器上访问地址`http://10.68.100.15:9000/hooks/redeploy-webhook`触发webhook执行脚本

```bash
curl http://10.68.100.15:9000/hooks/redeploy-webhook
```

在webhook server上查看结果

```bash
[webhook] 2020/11/06 09:50:28 Started GET /hooks/redeploy-webhook
[webhook] 2020/11/06 09:50:28 [6c5b21] incoming HTTP request from 10.68.0.7:40658
[webhook] 2020/11/06 09:50:28 [6c5b21] redeploy-webhook got matched
[webhook] 2020/11/06 09:50:28 [6c5b21] redeploy-webhook hook triggered successfully
[webhook] 2020/11/06 09:50:28 Completed 200 OK in 3.327051ms
[webhook] 2020/11/06 09:50:28 [6c5b21] executing /home/ubuntu/redeploy.sh (/home/ubuntu/redeploy.sh) with arguments ["/home/ubuntu/redeploy.sh"] and environment [] using /home/ubuntu as cwd
[webhook] 2020/11/06 09:50:28 [6c5b21] command output: 
[webhook] 2020/11/06 09:50:28 [6c5b21] finished handling redeploy-webhook
```

成功的创建了文件

```bash
ubuntu@VM-100-15-ubuntu:~$ ls
202011095028  hooks.json  redeploy.sh
```