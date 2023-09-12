---
title: "使用SSH tunnel实现免认证上网(误)"
date: 2023-09-12T21:34:24+08:00
draft: false
---

# 使用SSH tunnel实现免认证上网(误)

无意发现`Transmit`里有一台吃灰的机器, IP还是只能校内访问的教育网IP, 我想起了这是上学期用来实验的Linux服务器, 当时在这个上面写Jupyter notebook 和一键脚本来完成实验, 想想这个操作还是挺离谱的. SSH连上去, `htop`看了一下, 64GB内存配32GB存储, 很离谱啊...

写了个简单的`go helloweb`测试了一下, 外内网均无法访问, 看来只开放了SSH和实验平台的端口, 没办法, 又试了一下
```bash
ssh -L 1080:localhost:8080 user@host -p port
curl localhost:1080
```
发现可以正常收到response, 于是我有了这个想法.

## 下载gost

[gost](https://gost.run/)是常用的流量转发工具, 下载使用也很简单, 这里使用国内的github加速服务下载, 然后解压.
```bash
wget https://ghproxy.com/github.com/go-gost/gost/releases/download/v3.0.0-rc8/gost_3.0.0-rc8_linux_amd64.tar.gz
tar xvf gost_3.0.0-rc8_linux_amd64.tar.gz
```

## 启动gost

gost的使用很简单, 这里以socks5代理为例:
```bash
nohup ./gost -L socks5://:1080 &>/dev/null &
```

## 使用SSH tunnel连接服务器

这里我使用了[sshtunnel](https://github.com/elliotchance/sshtunnel)写了一个简单的软件用于快速建立tunnel:
```go
package main

import (
	"log"
	"os"

	"github.com/elliotchance/sshtunnel"
	"golang.org/x/crypto/ssh"
)

func main() {
	tunnel, _ := sshtunnel.NewSSHTunnel(
		"user@host:port",
		ssh.Password("password"),
		"localhost:3080",
		"1080",
	)
	tunnel.Log = log.New(os.Stdout, "", log.Ldate|log.Lmicroseconds)
	tunnel.Start()
}

```


然后启动, 再设置代理socks5为localhost:1080, 连接校园网, JLU.PC或有线网, 看到软件开始打印log, 说明已经成功连接. 再打开ipv6测试网站, 显示了host的ip, 但很遗憾, 没有ipv6... 这样就能免登录用上校园网了, 适合在没有任何正常登录方法的时候使用, 下载dogcom等登录软件后关闭, 请勿长期使用.