# 使用树莓派搭建全局翻墙的Wifi热点

本文利用树莓派创建一个即插即用的Wifi热点，通过网线连接公司或家庭的有线网络，为无线接入设备提供全局翻墙服务。

适用设备：Raspberry Pi 3B

主要步骤及原理：

1. 安装v2ray，为本机提供socks5代理（1080端口）
2. 安装dnsmasq，为本机提供dns解析服务
3. 配置v2ray，为特殊dns域名解析请求提供端口转发服务（15353端口）
4. 安装redsocks，为TCP流量提供端口转发服务（12345端口）
5. 为gfw准备IPSet和DNSMasq规则（部分域名解析->15353，符合的IP->IPSet）
6. 配置IPTables，将符合的IPSet的流量转发至12345端口
7. 配置无线热点

## 刷Raspbian
```
sudo dd bs=1m if=./2019-09-26-raspbian-buster-lite.img of=/dev/rdiskN conv=sync
```

连接网线，上电

### Enable SSH
Enable SSH through 'raspi-config' -> 'Interfacing Options' -> 'SSH'
```
sudo raspi-config
```

### 修正系统时间（否则可能导致无法安装/升级系统）
Select timezone through 'raspi-config' -> 'Localisation Options' -> 'Change Timezone'
```
sudo date -s '8 Jan 2020 9:09'
```

### 检查
1. eth0 成功获得IP (from 光猫)
```
ifconfig
```
2. ssh成功login（pi/raspberry)
```
ssh pi@raspberrypi.local
```
3. 系统时间正确
```
date
```

*Note* 以下操作均通过SSH操作

## 更新系统源索引
```
sudo apt update
```

### 安装依赖包

```bash
sudo apt-get install curl dnsutils lsof -y
```

# 安装v2ray

```
wget https://install.direct/go.sh
sudo bash go.sh
```
## 1. 配置outbounds规则，连接远端vps

将以下配置填入`/etc/v2ray/config.json`

```json
    {
      "protocol": "vmess",
      "settings": {
        "vnext": [{
          "address": "your_vps_ip",
          "port": 19368,
          "users": [{
              "id": "cc4e4ab1-e417-4618-8b82-f92065415407",
              "alterId": 64,
              "security": "auto"
          }]
        }]
      }
    }
```

## 2. 配置inbounds，向本机1080端口提供socks5代理

将以下配置填入`/etc/v2ray/config.json`

```json
    {
      "port": 1080,
      "listen": "0.0.0.0",
      "protocol": "socks",
      "settings": {
        "auth": "noauth",
        "udp": false
      }
    }
```

## 3. 测试连接

查询本机ip，确保显示本地ISP提供的ip地址

```bash
curl cip.cc
```

使用socks5代理查询本机ip，确保显示远端vps的ip地址

```bash
curl --socks5 localhost:1080 cip.cc
```

## 4. 启动服务并持久化

```
# 启动服务并检查状态
sudo systemctl start v2ray
sudo systemctl status v2ray
```

## 5. 检查
```
sudo reboot
sudo systemctl status v2ray
curl cip.cc; curl --socks localhost:1080 cip.cc
```

# 安装dnsmasq

检查当前DNS解析服务，一般由上级路由提供（192.168.1.1#53）

```bash
pi@raspberrypi:~ $ nslookup www.baidu.com
Server:     192.168.1.1
Address:    192.168.1.1#53

Non-authoritative answer:
www.baidu.com   canonical name = www.a.shifen.com.
Name:   www.a.shifen.com
Address: 180.101.49.11
Name:   www.a.shifen.com
Address: 180.101.49.12
```

## 使用dnsmasq为本机提供dns解析服务
```bash
sudo apt-get install dnsmasq
```

检查当前DNS解析服务，由本机dnsmasq在提供（127.0.0.1#53）

```bash
pi@raspberrypi:~ $ nslookup www.baidu.com
Server:     127.0.0.1
Address:    127.0.0.1#53

Non-authoritative answer:
www.baidu.com   canonical name = www.a.shifen.com.
Name:   www.a.shifen.com
Address: 180.101.49.12
Name:   www.a.shifen.com
Address: 180.101.49.11

pi@raspberrypi:~ $ sudo lsof -i :53
COMMAND PID    USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
dnsmasq 523 dnsmasq    7u  IPv4  14077      0t0  UDP *:domain
dnsmasq 523 dnsmasq    8u  IPv4  14078      0t0  TCP *:domain (LISTEN)
dnsmasq 523 dnsmasq    9u  IPv6  14079      0t0  UDP *:domain
dnsmasq 523 dnsmasq   10u  IPv6  14080      0t0  TCP *:domain (LISTEN)
```

## 配置dnsmasq

暂停服务，备份原配置文件

```bash
sudo systemctl stop dnsmasq
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.bak
```

写入以下配置内容 `/etc/dnsmasq.conf`

```
no-resolv # 不使用 /etc/resolv.conf 做 dns 解析
server=127.0.0.1
server=114.114.114.114 # 国内默认使用 114.114.114.114 来进行 dns 解析
conf-dir=/etc/dnsmasq.d
```

## 重启dnsmasq

```bash
sudo systemctl restart dnsmasq
```

# 配置v2ray在15353端口提供udp转发服务

### 1. 配置inbounds规则

将以下配置填入`/etc/v2ray/config.json`

```json
    {
      "protocol": "dokodemo-door",
      "port":15353,
      "settings":{
        "address":"8.8.8.8",
        "port":53,
        "network": "udp",
        "timeout": 30,
        "followRedirect": false
      }
    }
```

以上配置在本机15353端口接受udp入站请求，并将其转发至8.8.8.8#53

### 2. 重启v2ray

```bash
sudo systemctl restart v2ray
```

### 3. 测试

检查15353端口

```bash
sudo lsof -i :15353
```

测试53端口域名解析结果（墙内，受污染）

```bash
dig @127.0.0.1 -p 53 www.google.com
```

测试15353端口域名解析结果（墙外，8.8.8.8#53）

```bash
dig @127.0.0.1 -p 15353 www.google.com
```

# 安装redsocks，在12345端口提供流量转发服务

### 1. 配置包转发

 编辑`/etc/sysctl.conf`

```bash
net.ipv4.ip_forward=1
```

开启包转发

```bash
sudo sh -c "echo 1 > /proc/sys/net/ipv4/ip_forward"
```

### 2. 安装redsocks

```bash
sudo apt-get install redsocks
sudo systemctl stop redsocks
```

### 3. 配置redsocks

编辑`/etc/redsocks.conf`

```json
redsocks {
  local_ip = 0.0.0.0;
  local_port = 12345;
  ip = 127.0.0.1;
  port = 1080;
  type = socks5
}
```

### 4. 启动redsocks

```bash
sudo systemctl start redsocks
```

# IPSet，为gfw准备ipset和dnsmasq规则

```bash
sudo apt-get install git ipset ssh
git clone https://github.com/cokebar/gfwlist2dnsmasq.git
cd gfwlist2dnsmasq/
bash gfwlist2dnsmasq.sh -o gfwlist.conf -s gfwlist -p 15353 # -o 为输出文件， -s 为设置 ipset 集合名，-p 为 dns 端口
sudo cp gfwlist.conf /etc/dnsmasq.d/
```

以上生成的dnsmasq规则将执行以下动作：

- 符合规则的域名请求发送至本机`15353`端口来进行DNS域名解析
- 该域名解析出来的ip添加到名为`gfwlist`的ipset集合中

### 重启dnsmasq

```bash
sudo systemctl restart dnsmasq
```

# 设置iptables和ipset

```bash
sudo ipset -N gfwlist iphash # 新建一个名字为 gfwlist 的 ip 散列集合
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE # 将所有网卡的请求转发至eth0

# 如果发现由其它网卡转发过来请求的 ip 与 gfwlist 中的 ip 匹配，转发到 12345 端口
sudo iptables -t nat -A PREROUTING -p tcp -m set --match-set gfwlist dst -j REDIRECT --to-port 12345
# 本网卡请求的 ip 与 gfwlist 中的 ip 匹配，转发到 12345 端口
sudo iptables -t nat -A OUTPUT -p tcp -m set --match-set gfwlist dst -j REDIRECT --to-port 12345
```

### 持久化ipset和iptables

```bash
sudo apt-get install ipset-persistent
sudo apt-get install iptables-persistent
```

### 检查

测试当前ip，应显示当前ISP提供的IP地址

```bash
curl cip.cc
```

测试墙外网站连通性

```bash
curl www.google.com
```

# 配置热点

### 安装hostapd

```
sudo apt install hostapd
```

### 停止服务，并进行配置
```
sudo systemctl stop dnsmasq
sudo systemctl stop hostapd
```

### 给wlan0配置固定IP

编辑`/etc/dhcpcd.conf`

```
interface wlan0
    static ip_address=192.168.4.1/24
    nohook wpa_supplicant
```

### 重启dhcpcd
```
sudo service dhcpcd restart
```

### 配置DHCP服务（dnsmasq）

写入以下配置`/etc/dnsmasq.conf`

```
interface=wlan0
dhcp-range=192.168.4.2,192.168.4.20,255.255.255.0,24h
```

### 重启dnsmasq服务
```
sudo systemctl restart dnsmasq
```

### 配置hostapd

创建配置文件`/etc/hostapd/hostapd.conf`

```
interface=wlan0
driver=nl80211
ssid=NameOfNetwork
hw_mode=g
channel=7
wmm_enabled=0
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=AardvarkBadgerHedgehog
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
```

配置hostapd使用该配置，`/etc/default/hostapd`

```
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```

### 重启hostapd服务

```
sudo systemctl unmask hostapd
sudo systemctl enable hostapd
sudo systemctl start hostapd
```

## 检查

1. wlan0成功获取IP（static: 192.168.4.1）
2. 无线设备可成功连接wifi，并获取IP
3. 无线设备可访问墙外网站
4. 无线设备访问cip.cc，显示本地ISP的IP地址

## Reference

[raspberry-pi-dns, 使用树莓派做 DNS 服务器/透明代理](https://briteming.blogspot.com/2019/09/raspberry-pi-dns-dns.html)

[gfwlist2dnsmasq](https://github.com/cokebar/gfwlist2dnsmasq)

[v2ray](https://www.v2ray.com/chapter_02/protocols/dokodemo.html)

[使用树莓派实现路由器功能](https://beekc.top/2019/08/20/raspberry-wifi-ap/)

**The End**