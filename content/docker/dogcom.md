---
title: "Dogcom"
date: 2023-06-18T13:34:24+08:00
draft: false
---

# docker部署dogcom
## 起因
由于夏天的到来，宿舍的软路由出现过一次过热宕机的情况，导致宿舍的网络中断，无法正常上网，在刷了最新固件后，在本地随手验证了一下dogcom的可用性（原来的配置文件与dogcom的配置文件有较大区别，在luci-dogcom上验证时无法得到日志，于是我在本地验证时使用了docker），发现dogcom的可用性还是很高的，于是我就想到了在docker上部署dogcom，以此来保证宿舍网络的可用性，在出现问题时也能查看日志排查原因。
## 部署
我的设备是nanopi r2s，刷的骷髅头的docker固件，如果固件没有docker，需要手动安装，安装方法我不提供了，网上有很多教程，这里我只提供dogcom的docker部署方法。
### 准备工作
在openwrt上配置git或者直接下载dogcom的源码，然后将源码上传到服务器上，我这里使用的旁路由模式，而且电脑通过验证使得软路由正常上网，所以我直接使用git clone命令将源码下载到服务器上。
```bash
opkg update
opkg install libmbedtls git-http
```
然后克隆dogcom的源码到服务器上
```bash
git clone https://github.com/mchome/dogcom.git
```
如果网络条件不好可以使用代理加速
```bash
git clone https://ghproxy.com/github.com/mchome/dogcom.git
```
进入文件夹，新建Dockerfile文件，修改配置文件
```bash
vi Dockerfile
```
输入以下内容
```dockerfile
FROM alpine:latest
RUN apk add --no-cache gcc make musl-dev linux-headers
WORKDIR /app
COPY . .
RUN make
CMD ["./dogcom", "-m", "dhcp", "-c", "d.conf"]
```
修改配置文件
```bash
vi d.conf
```
输入以下内容
```conf
server = '10.100.61.3'
username = 'USERNAME'
password = 'PASSWORD'
CONTROLCHECKSTATUS = '\x20'
ADAPTERNUM = '\x1'
host_ip = '10.30.22.17'
IPDOG = '\x00'
host_name = 'dogcom'
PRIMARY_DNS = '10.10.10.10'
dhcp_server = '0.0.0.0'
AUTH_VERSION = '\x68\x00'
mac = 'MAC' #0x123456789abc
host_os = 'emu8086'
KEEP_ALIVE_VERSION = '\xdc\x02'
ror_version = False
```
其中server为dogcom服务器的地址，username和password为登录客户端的用户名和密码，host_ip为本机的ip地址（用处不大），PRIMARY_DNS为主DNS，dhcp_server为dhcp服务器的地址，mac为网络中心注册的mac地址，host_os为本机的操作系统，其他的配置项可以不用修改，如果需要修改，可以参考dogcom的配置文件说明。
### 构建镜像
在Dockerfile所在的目录下执行以下命令构建镜像
```bash
docker build -t dogcom .
```
### 运行容器
```bash
docker run -d --name dogcom --restart=always dogcom
```
### 查看日志
```bash
docker logs -f dogcom
```
## 总结
dogcom的docker部署就完成了，如果需要修改配置文件，可以直接修改d.conf文件，然后重启容器即可，如果需要查看日志，可以使用docker logs命令查看，如果需要查看dogcom的状态，可以使用docker stats命令查看，如果需要停止dogcom，可以使用docker stop命令停止容器，如果需要启动dogcom，可以使用docker start命令启动容器，如果需要删除dogcom，可以使用docker rm命令删除容器，如果需要删除dogcom镜像，可以使用docker rmi命令删除镜像。