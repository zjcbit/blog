---
title: "科学上网之加密梯子+原生IP"
date: 2023-05-07T13:24:55+08:00
draft: false
tag: ["技术", "梯子", "原生IP"]
categories: ["技术", "梯子", "原生IP"]
---
![img](http://rucltvshi.hn-bkt.clouddn.com/n.jpeg)
## 加密梯子+原生IP解决chatgpt封号问题

最近chatgpt账号很多都被封了，我之前申请的也被封了。上网看了一下原因可能有以下几个：

- 淘宝买的账号：相同IP申请了太多的账号；
- 流量异常：IP属地经常变更，判断为机器人；

因为我之前是用的TZ，经常网络延迟300ms左右，再加上自己的账号被封，导致不能玩了。再有共享的TZ安全风险也比较大，如果别人访问了不正经的网站，我也会收到影响。所以决心自己搭一个加密TZ，恰好此时opsnull 分享了一个他的搭建流程+[左耳朵耗子的文章](https://haoel.github.io/)。自己用周末时间搭建完成，现在总结一下分享出来。做为一个程序员，不但要知其然，还要知其所以然。首先会讲一下原理。如果只想知道搭建过程，请翻到最后。

现在的TZ实现方式非常多。只讲一下自己实践的方案原理。

## 技术选型

### 原生IP

所谓“原生 IP”就是指该网站的 IP 地址和其机房的 IP 地址是一致的，但是，很多 IDC 提供商的 IP 都是从其它国家调配来的，这导致我们就算是用了TZ了，也是使用了美国的 VPS，但是还是访问不了相关的服务。或者能够访问也出现很多问题，比如：

- google流量异常，频繁跳出人机身份验证的问题。
- IP定位漂移(可能是中国的IP)的问题。
- chatgpt、New Bing 无法访问的问题。

解释一下IP定位漂移：IP 地址归属地，和实际流量出口地 不一致。而在云服务器搭建的vps，会很容易出现ip漂移的问题，根本原因就是云厂商的 IP 段是可以自己调配的，都是全球分配的，而整个网络路由拓扑也是和运营商网络有差异。从而导致一个本属于美国的IP地址，其流量出口可能是欧洲。其中的流量异常也往往是因为ip定位漂移导致的。那么要如何解决呢？使用[Cloudflare WARP](https://p3terx.com/go/aHR0cHM6Ly8xLjEuMS4xLw) 

[Cloudflare WARP](https://p3terx.com/go/aHR0cHM6Ly8xLjEuMS4xLw) 是 Cloud­flare 提供的一项基于 Wire­Guard 的网络流量安全及加速服务，能够让你通过连接到 Cloud­flare 的边缘节点实现隐私保护及链路优化。换句话说就是通过CDN加速的方式使流量归属地与实际流量出口一致。

### 安全

说到安全就要去区分一下各种TZ的技术差异了。

#### VPN

Vpn，全称“虚拟私人网络（Virtual Private Network）”，是一种加密通讯技术。vpn是一个统称，它有很多的具体实现，比如PPTP、L2TP、IPSec和openvpn。vpn出现远早于GFW，所以它不是为了TZ而生的。我上面说了，vpn是一种加密通讯技术。虽然是加密的，但是也存在很多问题，最严重的一个就是流量特征过于明显。墙目前已经能够精确识别绝大部分vpn协议的流量特征并给予封锁，所以，vpn这种方式基本已经废了。现在随便一搜就有各种vpn流量特征识别的论文。论安全的话，最靠谱的应该还是使用HTTPS代理，然后把你的服务伪装成一个Web服务器，**我感觉这比其它的流量伪装更好，也更隐蔽。**

#### ****Proxy****

Proxy（代理）又分为正向代理和反向代理。TZ所用的代理都是正向代理。反向代理的作用主要是为服务器做缓存和负载均衡。正向代理主要有HTTP、HTTP over TLS(HTTPS)、Socks、Socks over TLS几种。其中，HHTTP和Socks都是明文协议，不能加密，只能匿名，无法保证安全，Socks over TLS 用的人又太少。HTTPS既可以匿名，也可以用于加密通信。HTTPS代理的流量特征和我们平时访问网站时所产生的HTTPS流量几乎一模一样，应该是最为安全的。HTTPS最大的缺点就是配置复杂。

#### Shadowsocks

Shadowsocks同样是一种代理协议，相对于vpn，shadowsocks有着极强的隐匿性；相对于HTTP代理，shadowsocks提供了较为完善的加密方案，相对于HTTPS代理，shadowsocks的安装配置更为简单。

最终选择https-proxy的方案，虽然比较复杂，但是也比较适合程序员玩一玩。

最终整体的技术选型为: 通过https代理确保安全，通过Cloudflare warp做加速且保证原生ip。

## 开搞

技术选型确定后，就是部署方案了。Https代理都需要哪些东西呢？实际上和部署一个https的服务是一样的。就需要四个东西：服务器、域名、证书、服务本身。

### 服务器

- [AWS LightSail](https://lightsail.aws.amazon.com/): 是一个非常便宜好用的服务，最低配置一个月 $3.5 美金，流量不限，目前的Zone不多，推荐使用日本，新加坡或美国俄勒冈。现对 2021/8/7 之后使用 Lightsail 的用户提供3个月的免费试用。
- [AWS EC2](https://aws.amazon.com/cn/): 香港、日本或韩国申请个免费试用一年的EC2 VPS。
- 此外还有[Google Cloud Platform](https://cloud.google.com/)、[Linode、](https://www.linode.com/)[Conoha、](https://www.conoha.jp/zh/)[Vultr、](https://www.vultr.com/)[Oracle Cloud](https://www.oracle.com/cloud/free/)，但是我都没试过，只试过EC2 和LightSail

#### AWS LightSail

以aws lightsail为例

- 选择 2GB/1Core 的规格 (10 刀一个月), 否则性能不足;
- 选择 ubuntu 20.04 系统, 18.04 内核版本太低，不支持 bbr, 后续安装 Cloudflare WARP 依赖的 WireGuard 启动时卡住;
- lighsail networking 部分为 VM 绑定一个公网 IP, 开放 TCP 443 端口;
- 开启 IPV6 协议(默认)
- 如果需要使用中文需要安装中文包，以支持 zh_CN.UTF‐8 locale。

```bash
#sudo apt-get install language-pack-zh-hans
#local-gen
#localectl list-locales
# 如果最终显示中包含了 zh_CN.UTF‐8 locale，即为成功了
```
#### 注意

1. region的选择，最好是日本，中国互联网流量国际出口有三条，分别位于青岛、汕头和上海的海底光缆连接点。通过日本是最快的。
2. 如果需要验证DNS配置的是否正确，可以将防火墙开放ICMP协议，然后ping自己的域名，看返回的公网IP是否正确。
3. 使用LightSail会存在一个问题就是LightSail本身需要外网访问，而在没有tz的情况下又无法访问，会有鸡生蛋的问题。如果想使用LightSail，最好先申请一个EC2。然后给予EC2使用简单的ssh tunnel来实现tz后，再进行后续的操作。

```bash
ssh -D 1080 -qCN -i key username@server:port
```

解释：

- `D`：本机SOCKS 服务端口
- `q` : quiet 模式，没有输出
- `C` : 数据压缩，可以节约一些带宽
- `N` : 不运行远程命令，只做端口转发
- key为从lightsail下载的私钥；username为服务器的用户名默认为ubuntu。server为公网ip。port为端口，默认为22

登录成功以后,本地 `1080`端口会开启一个 `SOCKS5` 协议的代理，只要配置好代理就能使用这个端口上网。如果是谷歌浏览器，配置好`SwitchyOmega`插件就能实现上外网。

### 域名

可以在GoDaddy上申请，GoDaddy上比较便宜，第一年3刀；也可以直接在aws上申请，比较贵，10+刀，在aws上申请相对配置会简单些。我是用GoDaddy(需要使用美版的)上申请的。

- DNS配置，(DNS)域名系统，的主要任务就是将域名映射到为IP地址。
    - 首先需要在GoDaddy上配置nameservers，将默认的nameservers修改为lightsail的nameservers。左侧为lightsail的nameservers，右侧为

DNS配置A记录

其中name为申请的域名，例如example.com, value为公网IP

### 证书、代理服务

- 证书：然后使用 [Let's Encrypt](https://letsencrypt.org/)来签 一个证书。使用 Let's Encrypt 证书你需要在服务器上安装一个 [certbot](https://certbot.eff.org/instructions)。但是此证书有效期只有90天，所以需要使用一个 cron job 来定期更新。
- 代理服务，[gost](https://github.com/ginuerzh/gost) 是一个非常强的代理服务，它可以设置成 HTTPS 代理，然后把你的服务伪装成一个Web服务器。

网友已经将启动gost代理服务、安装证书写成了自动化脚本。当前只有ubuntu的自动化脚本。

```bash
wget https://raw.githubusercontent.com/haoel/haoel.github.io/master/scripts/install.ubuntu.18.04.sh
chmod a+x install.ubuntu.18.04.sh
./install.ubuntu.18.04.sh
```

分别执行1, 2, 3, 4, 8。

- 注意，第3步生成证书时需要使用域名，切域名已经配好DNS。
- 第4步时需要使用域名，且要输入用户名和密码，用户名和密码随便定义就行，并记录下来，后面需要配置到客户端上。

```jsx
菜单选项

1) 安装 TCP BBR 拥塞控制算法
2) 安装 Docker 服务程序
3) 创建 SSL 证书
4) 安装 Gost HTTP/2 代理服务
5) 安装 ShadowSocks 代理服务
6) 安装 VPN/L2TP 服务
7) 安装 Brook 代理服务
8) 创建证书更新 CronJob
9) 退出
```

- 证书验证

```jsx
curl ‐v "https://www.google.com" ‐‐proxy "https://你的域名" ‐‐proxy‐user '第四步的用户名:第四步的密码'
```

### 原生IP

#### 安装

做完安装https服务代理相关的操作后，还要配置原生IP。需要做以下几个步骤

- 安装软件包
- 注册配置Cloudflare WARP(可以黑屏注册，也可以到官网注册)，记得要备份
- 配置双栈全局网络

可以使用这个一键安装的脚本来快速完成安装 [https://github.com/P3TERX/warp.sh](https://github.com/P3TERX/warp.sh) 来一键安装。

```jsx
~# wget git.io/warp.sh
~# chmod +x warp.sh
~# /warp.sh menu
```

```jsx
1. 安装 Cloudflare WARP 官方客户端
2. 自动配置 WARP 客户端 SOCKS5 代理
3. 管理 Cloudflare WARP 官方客户端
4. 安装 WireGuard 相关组件
5. 自动配置 WARP WireGuard IPv4 网络
6. 自动配置 WARP WireGuard IPv6 网络
7. 自动配置 WARP WireGuard 双栈全局网络
8. 管理 WARP WireGuard 网络
请输入选项:
```

先后执行如下步骤:

- 1 ‐ 安装 Cloudflare WARP 官方客户端
- 4 ‐ 安装 WireGuard 相关组件
- 7 ‐ 自动配置 WARP WireGuard 双栈全局网络

这个脚本会启动两个 Systemd 服务，一个是 warp‐svc，另一个是 wg‐quick@wgcf。第一个是 CloudflareWarp 的服务，第二个是 WireGuard 的服务。

使用./warp.sh status 来查看服务的状态，如果正常的话，会显示如下的信息。如果你的服务没有正常启动，那么会自动 disable wg‐quick@wgcf。

#### 检查

```jsx
./warp.sh status
 \\ //\ │ _\│ _\ │__│___ ___││____│││______
 \\/\//_\││_)││_)│ │││'_\/__│__/_`│││/_\'__│
 \V V/___\│ _<│ __/ │││││\__\││(_││││ __/│
 \_/\_/_/ \_\_│ \_\_│ │___│_│ │_│___/\__\__,_│_│_│\___│_│

 Copyright (C) P3TERX.COM │ https://github.com/P3TERX/warp.sh

 [INFO] Status check in progress... 12
 ‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐
 WARP Client : Running
 SOCKS5 Port : Off
 ‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐
 WireGuard
 IPv4 Network
 IPv6 Network
 ‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐
```

使用 curl [ipinfo.io](http://ipinfo.io/) 命令来检查你的 IP 地址，如果显示的是 Cloudflare 的 IP 地址，那么恭喜你，你已经 成功了

```jsx
root@ip‐172‐26‐1‐38:~# curl ipinfo.io 2{
"ip": "xx.xx.xx.xx",
"city": "Tokyo",
"region": "Tokyo",
"country": "JP",
"loc": "35.6895,139.6917",
"org": "AS13335 Cloudflare, Inc.",
"postal": "101‐8656",
"timezone": "Asia/Tokyo",
"readme": "https://ipinfo.io/missingauth"
```

执行以下命令检查是否连通。同时也能看到正在使用的是 Cloudflare 的网络

```jsx
curl ‐6 ip.p3terx.com
curl ‐4 ip.p3terx.com
curl ip.p3terx.com
```

执行如下命令进行测速

```jsx
~# curl ‐fsSL git.io/speedtest‐cli.sh │ sudo bash
~# speedtest
```

```jsx
# 可以看到包含如下返回的数据
Cloudflare
    2.46 ms   (0.12 ms jitter)
  315.40 Mbps (data used: 383.4 MB)
  242.88 Mbps (data used: 354.9 MB)
```

### 客户端配置

#### 电脑端

使用gost作为电脑的代理客户端,最终的转接流程类似于

```jsx
┌─────────────┐  ┌─────────────┐            ┌─────────────┐
│ ShadowSocks │  │             │            │             │
│    Client   ├──► Gost Client ├────────────► Gost Server │
│ (PAC Auto)  │  │             │            │             │
└─────────────┘  └─────────────┘            └─────────────┘
```

```jsx
brew install gost
```

```jsx
gost -L socks5://:1080 -F  'http2://你的用户名:你的密码@你的域名:443?bypass=/xxx/ns-whitelist.txt'
```

- 其中/xxx/ns-whitelist.txt 为需要直连，不代理的服务，网友总结了很多可以不用代理的服务列表。可以从 [https://github.com/felixonmars/dnsmasq-china-list/blob/master/ns-whitelist.txt](https://github.com/felixonmars/dnsmasq-china-list/blob/master/ns-whitelist.txt) 直接复制一个过来
- 配置浏览器代理插件 SwitchyOmega，添加一个SOCKS5类型的代理，这个应该都很熟悉了。
- 配置终端应用代理环境变量
    - 常见的终端软件 curl , git, wget 都能通过设置 HTTP_PROXY,HTTPS_PROXY，NO_PROXY 来配置一个网络代理，NO_PROXY 用来配置不需要代理的主机 (多个用逗号隔开), 那么我们就可以编写一个 bash 函数来运行需要走代理的命令:
    
    ```jsx
    with_proxy(){
    		HTTPS_PROXY=http://127.0.0.1:1080 HTTP_PROXY=http://127.0.0.1:1080 "$@"
    }
    ```
    
    - 把上面的 127.0.0.1:1080 改成你自己的网络代理, 将上面脚本写入到 ~/.bashrc 中，source ~/.bashrc 后就能使用 with_proxy 这个函数了，比如我想要使用代理网络下载一个文件 with_proxy wget https://⋯., 想要使用代理网络从 github clone 一个项目 with_proxy git clone https://⋯, 当我们不用with_proxy 这个函数的时候命令是不会走代理的。
    - 另外，你也可以使用如下的两个 alias, 这样就可以在需要代理的时候输入 proxy，不需要的时候输入unproxy:
    
    ```jsx
    SOCKS="socks5://127.0.0.1:1080"
    alias proxy="export http_proxy=${SOCKS} https_proxy=${SOCKS} all_proxy=${SOCKS}"
    alias unproxy='unset all_proxy http_proxy https_proxy'
    ```
    

#### 手机端

- iPhone，可以考虑使用 `ShadowRocket` （需要付费,使用美区账号），其中使用 HTTPS 的代理，配置上就好了。
- Android，可以考虑使用这个Plugin - [ShadowsocksGostPlugin](https://github.com/xausky/ShadowsocksGostPlugin)

其中`ShadowRocket`的配置如下

- 类型: HTTP2
- 地址: VPS 域名, 如 us.opsnull.com
- 端口: 443
- 用户: {USER}, gost 代理用户名;
- 密码: {PASSWORD}, gost 代理密码;

【参考如下】

- 防火长城wiki百科 [https://zh.wikipedia.org/wiki/防火长城?spm=ata.21736010.0.0.7d8b65b1nf8FfJ](https://zh.wikipedia.org/wiki/%E9%98%B2%E7%81%AB%E9%95%BF%E5%9F%8E?spm=ata.21736010.0.0.7d8b65b1nf8FfJ)
- 各种代理的区别 [https://medium.com/@thomas_summon/浅谈vpn-vps-proxy以及shadowsocks之间的联系和区别-b0198f92db1b](https://medium.com/@thomas_summon/%E6%B5%85%E8%B0%88vpn-vps-proxy%E4%BB%A5%E5%8F%8Ashadowsocks%E4%B9%8B%E9%97%B4%E7%9A%84%E8%81%94%E7%B3%BB%E5%92%8C%E5%8C%BA%E5%88%AB-b0198f92db1b)
- 科学上网原理 [https://hengyun.tech/something-about-science-surf-the-internet/?spm=ata.21736010.0.0.1eac78759TPw4G](https://hengyun.tech/something-about-science-surf-the-internet/?spm=ata.21736010.0.0.1eac78759TPw4G)
- 大神 opsnull 的整理