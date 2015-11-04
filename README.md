使用 Asus Merlin 实现路由器翻墙
==============================

onlyice 2015-11-1

## 前置条件

1. 你有一台刷上 Asus Merlin 固件并能支持 USB 扩展的路由器
2. 你已经有一台海外 VPS，并已经运行 Shadowsocks 服务端
3. 基础的 ssh 或 telnet 知识，基础的命令行知识
4. 耐心

## 思路

要翻墙，首先得看看墙做了什么，再看看如何应对它。

### 墙的干扰手段

下面列举一些 GFW 最常见的干扰手段。

1. 域名 / IP 屏蔽

   比如你发起一个到 Google 服务的 HTTP 请求，GFW 会识别你访问的域名，并直接给你一个 Connection Reset

2. 关键词屏蔽

   URL 中如果有敏感词，也会被 GFW 屏蔽

3. DNS 污染

   对于一些 GFW 不想让你访问的网站，使用国内的 DNS 服务器，很难解析出一个正确可用的 IP

4. 丢包

   对于像 Shadowsocks 的加密流量，GFW 并不能知道里面的内容是什么，于是判断到有大量与海外通讯的流量时，它就随机丢掉其中一些包

### 应对手段

1. 海外流量走 Shadowsocks，国内流量直连

2. 海外域名的 DNS 请求也通过 Shadowsocks 转发到 VPS 上返回
   因为你对这些海外服务的请求也是通过 Shadowsocks 发到 VPS 上来完成的，所以 DNS 也需要使用 VPS 解析出来的 IP

3. VPS 上装 [net-speeder][net-speeder] 或锐速等软件，用来优化高延迟、高丢包线路

[net-speeder]: https://github.com/snooda/net-speeder

## 具体做法

首先你需要先让你的路由器运行上 Asus Merlin。Asus Merlin 的 [wiki][asus-merlin-wiki] 上有详细的讲解。

下面的大部分内容，如果你不能顺利完成，请多参照 wiki 上的内容。

### 安装 Entware

Entware 原本是设计给 OpenWRT 使用的，也被移植到 Asus Merlin 上来了。它是一个包管理器，类似 Ubuntu 上的 `apt-get`，`Federo` 上的 `yast` 等。我们将用它来安装 Shadowsocks。

安装方法：

1. 插上你的 USB Stick
2. SSH / Telnet 到路由上，运行 `entware-setup.sh`
3. 试试运行 `opkg` 命令，观察是否正常

P.S. SSH / Telnet 的用户名/密码，跟你在网页上登陆路由器是一样的

详细的安装教程看 [wiki][entware-wiki]。

### 安装并运行 Shadowsocks

`opkg` 的仓库中已经有 Shadowsocks 了，分成两个包：`shadowsocks-libev-polarssl` 以及 `shadowsocks-libev`。据说 `shadowsocks-libev-polarssl` 更节约资源，但是我的路由足够快，所以我装的是 `shadowsocks-libev`。

安装方法：

```bash
$ opkg install shadowsocks-libev-polarssl
```

检查下你的 Entware 的 `bin` 目录（一般是 `/opt/bin`）有无 `ss-redir`, `ss-tunnel`。

在你的 Entware 的 etc 目录中（一般是 `/opt/etc/`）建立 `shadowsocks.json`，写入 Shadowsocks 的配置。配置的格式与服务端的 Shadowsocks 一致。

在 `/jffs/post-mount` （如果不存在这个文件，就创建它）中加入下面内容，表示路由启动并装载上 USB 后，运行 Shadowsocks：

```bash
nohup /tmp/opt/bin/ss-redir -c /tmp/opt/etc/shadowsocks.json &
nohup /tmp/opt/bin/ss-tunnel -c /tmp/opt/etc/shadowsocks.json -l 7913 -L 8.8.8.8:53 -u&
```

其中 `ss-redir` 会绑定某个本地端口，发到这个端口的流量都会被转发到 VPS 去；`ss-tunnel` 建立了一个通道，发到这个 `7913` 端口的请求都会被转到 VPS，VPS 再去请求 Google DNS (8.8.8.8) 做 DNS 解析。

### 生成 iptables

iptables 是 Linux 系统下用来做流量控制的工具。我们要用它来做实际的流量转发。

目的是实现：

1. 到内网的流量（如 127.0.0.1, 192.168.1.\*) 直连
2. 到国内 ISP 的流量直连
3. 到 VPS 的流量直连
4. 其他流量都转到 VPS 上

TL;DR: 下载 [`config/iptables_command.sh`][iptables-command-config] 文件，跳到 “使 iptables 规则被应用” 一节。

---

现在先新建一个文件，我们用它来保存 iptables 的命令。并在文件头写入下面内容，表示建立一条名为 SHADOWSOCKS 的 chain。

`iptables -t nat -N SHADOWSOCKS`

#### 内网流量的 iptables

内网的 IP 分布在固定的几个网段中，所以直接加入以下内容即可：

```bash
# Ignore LANs IP address
iptables -t nat -A SHADOWSOCKS -d 0.0.0.0/8 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 10.0.0.0/8 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 127.0.0.0/8 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 169.254.0.0/16 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 172.16.0.0/12 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 192.168.0.0/16 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 224.0.0.0/4 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 240.0.0.0/4 -j RETURN
```

#### 使用 bestroutetb 生成国内 IP 段的 iptables

[bestroutetb][] 是个 Node.js 程序，用来获取国内 ISP 的网段并生成一些转发的规则。安装好后运行：

```bash
$ bestroutetb -p custom --rule-format="iptables -t nat -A SHADOWSOCKS -d %prefix/%mask -j %gw"$'\n'  --gateway.net="RETURN" -o ./iptables
$ grep RETURN ./iptables > ./iptables.china
```

然后把 `iptables.china` 文件的内容复制到前面我们创建的文件中。

#### 到 VPS 的流量直连

加入：`iptables -t nat -A SHADOWSOCKS -d 167.160.165.237 -j RETURN`

#### 其他流量都连到 VPS 上

加入：

```bash
iptables -t nat -A SHADOWSOCKS -p tcp -j REDIRECT --to-ports 1080
iptables -t nat -I PREROUTING -p tcp -j SHADOWSOCKS
```

最后，把生成好的这个文件保存为 `/jffs/iptables_command.sh`。

#### 使 iptables 规则被应用

现在我们需要让这个文件在合适的时机被执行。我观察到 Asus Merlin 会在使用 PPPoE 跟运营商认证通过后（也就是平时说到的拨号上网过程），将 iptables 刷新。因此这个文件的运行时机应该是路由拨号过程完毕。

观察到深圳电信每一天会使你的 PPP 会话中断，即路由每一天会重新做一次拨号。而拨完号后有似乎有一些路由内置的功能会刷新 iptables，导致你自己注入的 iptables 被清除。看了下 Asus Merlin 的 [Custom config files][custom-config-files]，似乎只有 `ddns-start` 事件适合这个场景。DDNS 即 Dynamic DNS，如果你使用了 Asus Merlin 提供的 DDNS 服务，路由器每次从运营商拿到新 IP 时，它会将这个新 IP 注册到 DDNS 服务器上，注册完后运行 `ddns-start` 脚本。因此，新建 `/jffs/scripts/ddns-start` 文件，写入：

```bash
#!/bin/sh

sh /jffs/iptables_command.sh
```

Update: 如果你开启了 QOS 功能，Asus Merlin 会把 iptable 中某些表清除掉。可以把 `ddns-start` 文件换成 `qos-start` 来解决。

这个时候，我们区分国内外流量的事情就做完了。你应该重启下路由，然后试着在你的电脑上访问一个被墙的 IP 的网站服务，比如 Google 的 [`http://74.125.239.144`][google-website]，观察能否正常连上。

[iptables-command-config]: https://github.com/onlyice/asus-merlin-cross-the-gfw/tree/master/config/iptables_command.sh
[asus-merlin-wiki]: https://github.com/RMerl/asuswrt-merlin
[entware-wiki]: https://github.com/RMerl/asuswrt-merlin/wiki/Entware
[bestroutetb]: https://github.com/ashi009/bestroutetb
[google-website]: http://74.125.239.144

### 国外网站走 VPS 解析 DNS  

国内的 DNS 服务器在解析国外网站时，返回的 IP 基本上是不能正常访问的。同时我们想要国外的流量走 VPS，因此对于国外网站的 DNS 解析，也需要转发到 VPS 上去做。

`Asus Merlin` 中内置了 dnsmasq，可以用来指定特定的域名走特定的 DNS 服务器进行解析。所以剩下的问题是如何区分国内和国外的域名。

TL;DR: 下载 [`config/dnsmasq-conf`][dnsmasq-conf-files] 中的两个文件，然后跳到这一节最后部分。

---

我使用的 [dnsmasq-china-list][dnsmasq-china-list] 提供的中国网站列表。

dnsmasq-china-list 项目提供了三个 `dnsmasq` 的配置文件：

1. `accelerated-domains.china.conf`

   这个文件里面存放了绝大部分中国网站的域名，并使 dnsmasq 对这些域名使用 114.114.114.114 公共 DNS 进行解析。

2. `bogus-nxdomain.china.conf`

   这个文件存放了一些 IP，表示当远程 DNS 服务器返回这些 IP 时，dnsmasq 将其当作解析失败来处理。原因是一些运营商对一些不存在的域名做解析时，没有正常地返回域名不存在的错误，而是返回了自己的一些广告网站的 IP。

3. `google.china.conf`

   加速 Google 服务，我不太关心这个，没细看。

我这边的运营商并没有做上面第 2 点提到的 DNS 污染，所以我没有使用 `bogus-nxdomain.china.conf` 文件，只使用了 `accelerated-domains.china.conf` 文件。

对于 `accelerated-domains.china.conf` 文件，它使用了 114 公共 DNS 做解析。我觉得对于国内的域名，走运营商自己提供的 DNS 服务器会更好，于是执行了下面的命令，把这个文件中的 DNS 服务器换成默认的：

```bash
sed -i "s|^\(server.*\)/[^/]*$|\1/#|" ./accelerated-domains.china.conf
```

然后，生成一个文件告诉 dnsmasq 除了国内的域名，其他的走之前 ss-tunnel 绑定的端口，转发到 VPS 上走 Google 的 8.8.8.8 DNS 服务器。

```bash
echo server=/#/127.0.0.1#7913 > foreign-domains.conf
```

然后我们要把这个配置文件让 dnsmasq 感知到。这里使用 Asus Merlin 提供的 [Custom config files][custom-config-files] 能力，将一条配置加入 `dnsmasq.conf` 中：

```bash
mkdir /jffs/dnsmasq-conf
cp ./accelerated-domains.china.conf /jffs/dnsmasq-conf
cp ./foreign-domains.conf /jffs/dnsmasq-conf

echo conf-dir=/jffs/dnsmasq-conf > /jffs/configs/dnsmasq.conf.add
```

[dnsmasq-conf-files]: https://github.com/onlyice/asus-merlin-cross-the-gfw/tree/master/config/dnsmasq-conf
[dnsmasq-china-list]: https://github.com/felixonmars/dnsmasq-china-list
[custom-config-files]: https://github.com/RMerl/asuswrt-merlin/wiki/Custom-config-files

### Done!

重启路由，把你的电脑/手机的 DNS 缓存消除，试试能不能访问境外网站。

如果失败了:

1. 看看路由器上 `/tmp/syslog.log` 中的 log
2. 看看 `ss-redir` 及 `ss-tunnel` 有没有正常运行
3. 查阅 Asus Merlin 的 wiki 等，确认你的各个步骤正确
4. 多使用 `dig` 或 `nslookup` 命令做验证

Enjoy the free Internet.
