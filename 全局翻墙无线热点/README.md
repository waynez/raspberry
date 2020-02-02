# 使用树莓派搭建全局翻墙的Wifi热点

本文利用树莓派创建一个即插即用的Wifi热点，通过网线连接公司或家庭的有线网络，为无线接入设备提供全局翻墙服务。

适用设备：Raspberry Pi 3B

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

# 安装v2ray

```
wget https://install.direct/go.sh
sudo bash go.sh
```
## 配置v2ray客户端
修改`/etc/v2ray/config.json`
1. 填入远端v2ray server(VPS)信息
2. 在本机1080端口启动socks5代理
```
{
  "log": {
    "access": "/var/log/v2ray/access.log",
    "error": "/var/log/v2ray/error.log",
    "loglevel": "warning"
  },
  "inbounds": [{
    "port": 1080,
    "listen": "0.0.0.0",
    "protocol": "socks",
    "settings": {
      "auth": "noauth",
      "udp": false
    }
  }],
  "outbounds": [{
    "protocol": "vmess",
    "settings": {
        "vnext": [{
          "address": "your_remote_vps",
          "port": 19368,
          "users": [{
              "id": "cc4e4ab1-e417-4618-8b82-f92065415407",
              "alterId": 64,
              "security": "auto"
          }]
        }]
    }
  },{
    "protocol": "blackhole",
    "settings": {},
    "tag": "blocked"
  }],
  "routing": {
    "rules": [
      {
        "type": "field",
        "ip": ["geoip:private"],
        "outboundTag": "blocked"
      }
    ]
  }
}
```

## 启动服务并持久化
```
# 启动服务并检查状态
sudo systemctl start v2ray
sudo systemctl status v2ray
curl cip.cc; curl --socks5 localhost:1080 cip.cc

# 设置开机启动
sudo systemctl enable v2ray
```

## 检查
```
sudo reboot
sudo systemctl status v2ray
curl cip.cc; curl --socks localhost:1080 cip.cc
```

# 安装DNSCrypt

```
# 下载编译好的binary
wget https://github.com/DNSCrypt/dnscrypt-proxy/releases/download/2.0.36/dnscrypt-proxy-linux_arm-2.0.36.tar.gz
tar xvf dnscrypt-proxy-linux_arm-2.0.36.tar.gz

# 安装
sudo cp dnscrypt-proxy /usr/local/bin

# 创建工作目录
sudo mkdir /etc/dnscrypt

# 拷贝配置文件
sudo cp example-dnscrypt-proxy.toml /etc/dnscrypt/dnscrypt-proxy.toml
```

## 配置DNSCrypt
```
sudo mkdir /etc/dnscrypt
sudo cp example-dnscrypt-proxy.toml /etc/dnscrypt/dnscrypt-proxy.toml

# 配置 server_names, listen_address('127.0.0.1:15353'), log_file('/var/log/dnscrypt-proxy.log'), query_log(file = '/var/log/query.log')
sudo vi /etc/dnscrypt/dnscrypt-proxy.toml
```

## 检查后kill进程
```
# 安装dig等工具
sudo apt install dnsutils

# 手动运行服务
/usr/local/bin/dnscrypt-proxy -config /etc/dnscrypt/dnscrypt-proxy.toml

# 检查dnscrypt正常工作于15353端口，并正确输出结果
dig +short @127.0.0.1 -p 15353 cloudflare.com AAAA

# 结束进程
# kill -9 ${pid}
```

## 持久化
```
# 安装service
/usr/local/bin/dnscrypt-proxy -service install

# 修改工作目录，WorkingDirectory=/etc/dnscrypt
# 指定配置文件，-config /etc/dnscrypt/dnscrypt-proxy.toml
sudo vi /etc/systemd/system/dnscrypt-proxy.service

# reload systemd
sudo systemctl daemon-reload

# 重启服务
sudo systemctl restart dnscrypt-proxy

# 检查
dig +short @127.0.0.1 -p 15353 cloudflare.com AAAA

# 重启，并检查服务状态
sudo reboot
sudo systemctl status dnscrypt-proxy
dig +short @127.0.0.1 -p 15353 cloudflare.com AAAA
```


# 配置wlan0为Wireless AP

## 安装dnsmasq，hostapd
```
sudo apt install dnsmasq hostapd
```

### 停止服务，并进行配置
```
sudo systemctl stop dnsmasq
sudo systemctl stop hostapd
```

### 给wlan0配置固定IP，并指定dnscrypt为系统提供dns服务(eth0 & wlan0)
```
# 配置以下信息
# interface wlan0
#     static ip_address=192.168.4.1/24
#     nohook wpa_supplicant
#
# no-resolv
# server=127.0.0.1#15353
sudo vi /etc/dhcpcd.conf
```

### 重启dhcpcd
```
sudo service dhcpcd restart
```

### 配置DHCP服务（dnsmasq）
```
# 备份原配置文件
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig

# 创建配置文件:
#
# interface=wlan0      # Use the require wireless interface - usually wlan0
# dhcp-range=192.168.4.2,192.168.4.20,255.255.255.0,24h
sudo vi /etc/dnsmasq.conf
```

### 重启dnsmasq服务
```
sudo systemctl restart dnsmasq
```

### 配置hostapd
```
# 配置以下内容
#
# interface=wlan0
# driver=nl80211
# ssid=NameOfNetwork
# hw_mode=g
# channel=7
# wmm_enabled=0
# macaddr_acl=0
# auth_algs=1
# ignore_broadcast_ssid=0
# wpa=2
# wpa_passphrase=AardvarkBadgerHedgehog
# wpa_key_mgmt=WPA-PSK
# wpa_pairwise=TKIP
# rsn_pairwise=CCMP
sudo vi /etc/hostapd/hostapd.conf

# 指定配置文件 DAEMON_CONF="/etc/hostapd/hostapd.conf"
sudo vi /etc/default/hostapd
```

### 重启hostapd服务
```
sudo systemctl unmask hostapd
sudo systemctl enable hostapd
sudo systemctl start hostapd
```

## 检查
```
# wlan0成功获取IP（static: 192.168.4.1）
ifconfig

# 可以用iPad等设备连接Wifi，并成功获取IP
# 此时iPad无法联网，因IPTables无法处理wlan0的流量，只能通过AP访问到raspberrypi.local

# 重启，并再次验证AP的访问功能
sudo reboot
```

# 允许IPV4包转发

```
# net.ipv4.ip_forward=1
sudo vi /etc/sysctl.conf
# 开启ip转发
sudo sh -c "echo 1 > /proc/sys/net/ipv4/ip_forward"
```

# 安装redsocks

```
sudo apt install redsocks
```

## 配置Redsocks
```
# 停止redsocks服务
sudo systemctl stop redsocks

# 配置以下内容
# redsocks {
#    local_ip = 0.0.0.0;
#    local_port = 12345;
#    ip = 127.0.0.1;
#    port = 1080;
#    type = socks5
# }
sudo vi /etc/redsocks.conf
```

## 持久化
N/A，已自动安装

## 检查服务
```
sudo reboot
sudo systemctl status redsocks
```

# 配置IPTables

```
# 新建规则链
sudo iptables -t nat -N SOCKS

# 忽略节点域名, 这里需要将自己的VPS IP填入
sudo iptables -t nat -A SOCKS -d ${VPS_IP} -j RETURN

# 忽略局域网地址
sudo iptables -t nat -A SOCKS -d 0.0.0.0/8 -j RETURN
sudo iptables -t nat -A SOCKS -d 10.0.0.0/8 -j RETURN
sudo iptables -t nat -A SOCKS -d 127.0.0.0/8 -j RETURN
sudo iptables -t nat -A SOCKS -d 169.254.0.0/16 -j RETURN
sudo iptables -t nat -A SOCKS -d 172.16.0.0/12 -j RETURN
sudo iptables -t nat -A SOCKS -d 192.168.0.0/16 -j RETURN
sudo iptables -t nat -A SOCKS -d 224.0.0.0/4 -j RETURN
sudo iptables -t nat -A SOCKS -d 240.0.0.0/4 -j RETURN

# 流量转发到redsocks
sudo iptables -t nat -A SOCKS -p tcp -j REDIRECT --to-ports 12345
sudo iptables -t nat -A OUTPUT -p tcp -j SOCKS
sudo iptables -t nat -A PREROUTING -p tcp -j SOCKS
```

## 持久化
```
sudo apt install iptables-persistent

# 当前IPTables rules被保存至
ls -l /etc/iptables/rules.v4
ls -l /etc/iptables/rules.v6

# 由iptables-persistent安装
sudo systemctl status netfilter-persistent
```

## 检查
1. raspberrypi.local上的tcp流量都自动转发至redsocks的12345端口，再由redsocks服务自动转发至v2ray，并最终离开本机
```
# 以下命令应该显示远端VPS的IP
curl cip.cc

# 应看到cip.cc的解析结果
tail /var/log/query.log

# 应得到服务器的页面
curl www.google.com
```
2. 从接入wifi的移动设备访问www.cip.cc，应该显示VPS的IP
```
# 应看到cip.cc的解析结果
tail /var/log/query.log
```
3. 从接入wifi的移动设备访问www.youtube.com，应顺利打开
```
# 应看到www.youtube.com相关的解析结果
tail /var/log/query.log
```
4. 重启，并重复以上测试

**The End**
