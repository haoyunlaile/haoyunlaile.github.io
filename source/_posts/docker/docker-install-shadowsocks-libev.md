---
title: 快速部署 Shadowsocks Docker版
date: 2020-03-06
tags: ["docker","shadowsocks"]
categories:	docker
---

## 需求
搭梯子翻墙访问google

## 准备条件

1. 一台墙外VPS（本文使用腾讯云香港 - centos7.6）
2. 一台能SSH连接到VPS的本地电脑 （推荐连接工具xshell）

## 遇到的问题
因为在腾讯上直接安装使用shadowsocks遇到了"connect reset by peer"的问题，在公司访问(可直连境外)是正常的，用4g/家里wifi访问就会出现上述错误，怀疑是腾讯云做了相关网站的流量拦截，故想到这用docker再代理一层。

## 服务端安装步骤

安装docker

{% codeblock lang:bash %}
wget -qO- get.docker.com | bash
{% endcodeblock %}

查看docker的版本信息、加入开启启动、启动docker应用

{% codeblock lang:bash %}
docker version
systemctl enable docker
systemctl start docker
{% endcodeblock %}

拉取docker版shadowsocks-libev

{% codeblock lang:bash %}
docker pull appso/shadowsocks-libev
{% endcodeblock %}

创建shadowssocks配置文件，主要不要变动配置文件目录，默认配置路径为 **/etc/shadowsocks-libev/config.json**

  <!--more--> 

{% codeblock lang:bash %}
mkdir -p /etc/shadowsocks-libev/
touch /etc/shadowsocks-libev/config.json
vi /etc/shadowsocks-libev/config.json
{% endcodeblock %}

config.json 配置内容

{% codeblock lang:json %}
{
    "server":"0.0.0.0",
    "server_port":443,
    "password":"your client connection password",
    "timeout":300,
    "method":"aes-256-gcm",
    "fast_open":false,
    "mode":"tcp_and_udp"
}
{% endcodeblock %}

|    名称     |                             解释                             |
| :---------: | :----------------------------------------------------------: |
|   server    |                        服务端监听地址                        |
| server_port |                     客户端用于连接的端口                     |
|  password   |                     客户端用于连接的密码                     |
|   timeout   |                           超时时间                           |
|   method    | 默认为 `aes-256-cfb`，参阅 [Encryption](https://github.com/shadowsocks/shadowsocks/wiki/Encryption) |
|    mode     | 是否启用 TCP / UDP 转发，参阅 [shadowsocks-libev(8)](https://jlk.fjfi.cvut.cz/arch/manpages/man/shadowsocks-libev.8) |
|  fast_open  | 是否启用 [TCP Fast Open](https://github.com/shadowsocks/shadowsocks/wiki/TCP-Fast-Open) |

使用docker启动shadowsocks

{% codeblock lang:bash %}
docker run -d -p 443:443 -p 443:443/udp --name ss-libev -v /etc/shadowsocks-libev:/etc/shadowsocks-libev appso/shadowsocks-libev
{% endcodeblock %}

查看容器启动状态

{% codeblock lang:bash %}
[root@007_centos ~]# docker ps -as
CONTAINER ID        IMAGE                     COMMAND                  CREATED             STATUS                  PORTS                                        NAMES               SIZE
84c3fd45cbea        appso/shadowsocks-libev   "ss-server -c /etc/s…"   2 days ago          Up 2 days               0.0.0.0:443->443/tcp, 0.0.0.0:443->443/udp    ss-libev           0B (virtual 120MB)
{% endcodeblock %}

查看端口(443)监听状态

{% codeblock lang:bash %}
[root@007_centos ~]# netstat -anp | grep 443
tcp6       0      0 :::443                  :::*                    LISTEN      13435/docker-proxy  
udp6       0      0 :::443                  :::*                                13446/docker-proxy
{% endcodeblock %}

至此，服务端安装完毕。



## windows客户端安装

打开 https://github.com/shadowsocks/shadowsocks-windows/releases 下载最新版本客户端，截止本文编写时间，最新版本为 [4.1.9.2](https://github.com/shadowsocks/shadowsocks-windows/releases/tag/4.1.9.2) ，下载后直接打开对应客户端进行配置,应用确定即可。

![shoadowsocks-windows](/images/shoadowsocks-windows.png)

如果使用chrome代理浏览器流量可以下载SwitchyOmega插件，直接安装到chrome的拓展程序里面即可

下载地址： https://github.com/FelisCatus/SwitchyOmega/releases 

插件配置如下

![shoadowsocks-switchomega](/images/shoadowsocks-switchomega.png)

一般情况下，至此即可成功代理浏览器流量



## android客户端安装

打开  https://github.com/shadowsocks/shadowsocks-android/releases  下载最新版本客户端，截止本文编写时间，最新版本为  [v5.0.5](https://github.com/shadowsocks/shadowsocks-android/releases/tag/v5.0.5)  ，下载后直接打开对应客户端进行配置,点击那个小飞机即可。

配置跟windows端配置类似，挺简单的，自行摸索一会儿就可以搞定。

![shoadowsocks-android](/images/shoadowsocks-android.png)



##	参考资料

1.  https://github.com/shadowsocks/shadowsocks-libev 
2.  https://github.com/shadowsocks/shadowsocks-android
3.  https://github.com/shadowsocks/shadowsocks-windows
4.  https://hub.docker.com/r/appso/shadowsocks-libev/