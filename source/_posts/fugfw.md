title: 科学上网：Digital Ocean + Shadowsocks
date: 2015-12-06 18:37:12
tags: vps_proxy
categories: Misc
banner: http://7xoy8o.com1.z0.glb.clouddn.com/8_1.png
---
周末在家研究了下“科学上网”，网上攻略众多，找了个同事强推的工具Shadowsocks试了下，果然很简单，很快，很稳定。这里简单记录了下搭建的整个过程。
主要核心工具就是Digital Ocean的VPS + Shadowsocks工具。
<!--more--> 

## Digital Ocean的VPS
部署服务端首先需要一台VPS服务器（"Virtual Private Server"，或简称 "VPS"），这里主要使用的是Digital Ocean,该公司提供VPS比较稳定，而且提供SSD。最重要的是时常提供优惠，打折。可以降低使用的成本。优惠码和折扣信息可以自行在网上搜索，很多。
点击[https://www.digitalocean.com/](https://www.digitalocean.com/) 进入Digital进入Digital Ocean后注册一个用户后，需要绑定信用卡或Paypal支付验证，我使用了Paypal的支付，充值了5美金。
然后选择Droplet（就是VPS），选择了最便宜的配置：512M内存，20GB的硬盘。Image选择了14.04的ubuntu，SF的机房。5美金一个月，当订单完成后，1分钟内会部署好你的VPS，同时有一封邮件发送到你的注册邮箱，其中提供了登陆的方法和账号密码等信息。
![](http://7xoy8o.com1.z0.glb.clouddn.com/1.PNG)

这样我们就有一台VPS了，可以登上去测试下。


## Shadowsocks
shadowsocks是一款私有协议（GFW就无法分析了）的代理软件，，目前还可以稳定提供上网代理。
shadowsocks客户端会在本地开启一个socks5代理，通过此代理的网络访问请求由客户端发送至服务端，服务端发出请求，收到响应数据后再发回客户端。
所以要选择一台台墙外的服务器来部署 shadowsocks 服务端。

### 安装
在Ubuntu下安装shadowsocks：
```bash
apt-get install python-pip
pip install shadowsocks
```

### 配置
```bash
root@royluo:#vim /etc/shadowsocks.json
{
	"server":"0.0.0.0", 
	"server_port":8388, 
	"local_port":1080, 
	"password":"password", //<======config your own passwd
	"timeout":600, 
	"method":"aes-256-cfb", 
}
```

### 启动和关闭
```bash
ssserver -c /etc/shadowsocks.json -d start
ssserver -c /etc/shadowsocks.json -d stop
```

Server端就是这么多配置。
	
## 客户端配置
### Win10
#### 代理服务器设置
[http://sourceforge.net/projects/shadowsocksgui/files/dist/](http://sourceforge.net/projects/shadowsocksgui/files/dist/)点击下载dotnet4.0的那个。
下载后解压运行，在界面中选择编辑服务器，配置如下,完成后启动代理。
![](http://7xoy8o.com1.z0.glb.clouddn.com/2.PNG)

#### Chrome设置
下载Proxy SwitchySharp 插件:
[https://chrome.google.com/webstore/detail/proxy-switchysharp/dpplabbmogkhghncfbfdeeokoefdjegm?hl=zh-CN](https://chrome.google.com/webstore/detail/proxy-switchysharp/dpplabbmogkhghncfbfdeeokoefdjegm?hl=zh-CN)
配置如下：
![](http://7xoy8o.com1.z0.glb.clouddn.com/3.PNG)
![](http://7xoy8o.com1.z0.glb.clouddn.com/4.PNG)
最后在chrome的右上角会出现一个地球的button，可以切换代理的开关。
![](http://7xoy8o.com1.z0.glb.clouddn.com/5.PNG)
试试google吧，是不是感觉呼吸到自由的空气了。
![](http://7xoy8o.com1.z0.glb.clouddn.com/6.PNG)

本人不才，Firefox的设置一直没成功，望高手指点。。。。。

### MAC
MAC的配置就更简单了，下载
[http://pan.baidu.com/s/1dDLXimt](http://pan.baidu.com/s/1dDLXimt)，安装。
如下配置：
![](http://7xoy8o.com1.z0.glb.clouddn.com/7.png)
一步到位。
