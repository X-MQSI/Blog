---
title: 个人VPS开设邮件服务器
description: iRedMail的安装配置与优化
slug: vps-iRedMail
date: 2024-10-09 20:30:00+0800
provider: gitalk
card: summary
image: iRedMail.png
keywords:
    - iRedMail
    - MQSI
    - VPS
    - 邮件服务器
categories:
    - 科学技术
tags:
    - VPS
    - 邮件服务器
---

最近在 [AlphaVps](https://alphavps.com) 买了一台 VPS 服务器，2.99欧元/月，折合为人民币大约是25人民币/月，个人感觉还算不错。  

最开始只打算做个 frp 映射本地端口，但是 frp 没做成，感觉闲置也是浪费资源，就决定搭建一个个人邮局。  

当然，本文章提及的资料及流程也有一定的时代局限性，毕竟未来谁也说不准呢。  

本文也不能覆盖所有的可能情况，你也需要多查询其他文章并合理使用 AI 来解决问题。  

言尽至此。  

**********

## 前情提要

- **System**: Ubuntu 22.04 （尽量全新安装）  
- **iRedMail**: iRedMail 1.7.1  

**********

## 0x01 前置准备

本章默认以 `root` 权限来执行命令，若在非 `root` 环境，请自行 `sudo`。  

### 1. 文档编辑器

欲利其功必先利其器。首先，你需要一个得心应手的文档编辑器，如 `nano` 或者其他。

### 2. 网络环境

由于 OpenVZ 虚拟平台的特性，IPv6 可能无法使用，请你确认自己的网络是否正常，保证安装顺利。

### 3. 端口

Ubuntu 或主机商提供的的防火墙可能会拦截 80 与 443 端口。你的 Ubuntu 也可能自带了 Apache2 等软件占用这两个端口。请确保端口畅通无侵占。

**********

## 0x02 正式安装

一些其他的参考文章：  
- [iRedMail 官方文档](https://docs.iredmail.org/install.iredmail.on.debian.ubuntu-zh_CN.html)  
- [知乎文章](https://zhuanlan.zhihu.com/p/509130006)  

### 1. 修改主机名  

   在本机公网 IP 后添加（或修改）主机名与短名称：  

   ```bash
   nano /etc/hosts
   ```

   ```bash
   # IP with Mail Hostname
   *.*.*.* mail.example.com mail localhost
   ```

   **检查**：  

   ```bash
   hostname -f
   ```

   此时应该输出：`mail.example.com`  

   > 注意，你应该把 IP 与域名换成自己的内容。

### 2. 安装解压软件

   安装 `tar` 和 `gzip` 程序用于解压缩文件：  

   ```bash
   apt-get install tar gzip
   ```

### 3. 下载解压与运行

   获取安装入口文件、解压并运行：

   ```bash
   wget https://github.com/iredmail/iRedMail/archive/refs/tags/1.7.1.tar.gz
   tar zxf 1.7.1.tar.gz
   cd iRedMail-1.7.1
   bash iRedMail.sh
   ```

### 4. 配置iRedMail

   根据安装提示自行配置。(参考[知乎文章](https://zhuanlan.zhihu.com/p/509130006) )

   > 安装过程中注意不要启用防火墙（Firewall），否则如果你安装的 V2Ray 等服务可能会出现故障。

### 5. 查看安装信息

   安装结束后，你可以在此处查看安装的配置信息与小贴士：

   ```bash
   nano /root/iRedMail-1.7.1/iRedMail.tips
   ```

### 6. 重启服务器

   ```bash
   reboot
   ```

   此时 iRedMail 的安装就结束了，你可以通过 `https://<ip地址>/mail` 体验访问网页版邮箱。

**********

## 0x03 完善配置

### 1. 绑定 IP

你需要设置如下 DNS 记录：  

- **A 记录**：主机名填 `mail`，IPv4 地址填你的服务器 IP  
- **MX 记录**：主机名填 `mail`，邮件服务器地址为 `mail.example.com`

绑定结束后，你可以通过 `https://mail.example.com/mail` 来访问网页版邮箱并尝试发信。  
> *如果收信失败，请阅读第 3 点的内容。*

如果你访问了网页版邮箱，你就会发现一个很要命的事情：没有证书。

### 2. 申请 SSL 证书

推荐使用 **Let's Encrypt** 来申请免费的 SSL 证书。你可以通过以下步骤申请和安装证书：

**Certbot** 是一个工具，可以帮助你自动申请和管理 Let's Encrypt 证书。  

```bash
apt update
apt install certbot python3-certbot-nginx
```

安装完成后，你可以使用以下命令申请证书：  

```bash
certbot --nginx -d mail.example.com
```

如果遇到 `command not found` 报错，可能是 Certbot 没有正确安装，可以尝试以下步骤：

```bash
add-apt-repository ppa:certbot/certbot
apt update
apt install certbot python3-certbot-nginx
```

安装完成后，检查 Certbot 是否成功安装：  

```bash
certbot --version
```

再继续申请证书：  

```bash
certbot --nginx -d mail.example.com
```

Certbot 会自动配置 Nginx 并申请证书。根据提示操作，Certbot 会处理大部分工作。  
> **记住申请后给出的 PEM 密钥地址。**

由于 iRedMail 的配置风格，你还需要手动调整。

申请完成后，Certbot 会自动更新你的 Nginx 配置并启动服务。可以使用以下命令检查 Nginx 是否正常运行：  

```bash
nginx -t  # 测试配置
systemctl restart nginx  # 重启 Nginx
```

你需要找到并编辑 Nginx 的配置文件：

```bash
cd /etc/nginx/sites-enabled/
ls
```

你会看到两个配置文件，如：  

```bash
00-default-ssl.conf  00-default-ssl.conf
```

编辑带有 "ssl" 字样的文件：  

```bash
nano 00-default-ssl.conf
```

参考如下配置：  

```conf
server {
    listen 443 ssl http2;
    server_name mail.example.com;

    ssl_certificate /etc/letsencrypt/live/mail.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/mail.example.com/privkey.pem;

    root /var/www/html;
    index index.php index.html;

    if ($host != 'mail.example.com') {
        return 418;
    }

    include /etc/nginx/templates/misc.tmpl;
    include /etc/nginx/templates/iredadmin.tmpl;
    include /etc/nginx/templates/roundcube.tmpl;
    include /etc/nginx/templates/sogo.tmpl;
    include /etc/nginx/templates/netdata.tmpl;
    include /etc/nginx/templates/php-catchall.tmpl;
    include /etc/nginx/templates/stub_status.tmpl;
}
```

> 注意，你应该把 IP 与域名换成自己的内容。

完成配置后，重启服务器并再次访问，此时应该不会再抛出 SSL 错误。  

**Let's Encrypt** 证书有效期为 90 天，建议设置自动续期。可以使用以下命令设置：  

```bash
sudo certbot renew --dry-run
```

如果没有问题，系统会自动在到期前续期。

### 3. 优化性能与可达性

iRedMail 自带了一些安全扫描与灰黑名单功能，这可能导致邮件的可达性存在问题。你可以参考这篇文章进行处理：  

- [处理灰黑名单问题](https://www.cnblogs.com/maybreath/p/14966598.html)

### 4. 反垃圾邮件处理

大部分邮箱服务提供商会将邮件标记为垃圾邮件。你可以先在 [mail-tester.com](https://www.mail-tester.com/) 进行一个测试。

为降低垃圾邮件特征，首先需要为邮箱添加 SPF 记录：  

```text
TXT 记录：主机名设置为“@”，内容为“v=spf1 mx a include:mail.example.com ~all”
TXT 记录：主机名设置为“mail”，内容为“v=spf1 a mx ip4:*.*.*.* ~all”
```

> **记得改为自己的域名和 IP。**

然后，添加 DKIM 记录。首先获取记录值（公钥）：  

```bash
amavisd-new showkeys
```

记录值类似于：  

```text
v=DKIM1; p=xxxxxxxxxxxxxxxxxxxxxx
```

接着添加解析：  

```text
TXT 记录：主机名设置为“dkim._domainkey”，内容为上面的记录值。
```

还需要配置 DMARC 记录来增强邮件验证：  

```text
TXT 记录：主机名设置为“_dmarc.mail”，内容为“v=DMARC1; p=none; rua=mailto:管理员邮箱”
```

验证这些记录可以使用 [dmarcly.com](https://dmarcly.com/tools)。  

最后，你需要设置一个反向 DNS 记录（PTR 记录）。这需要联系你的主机提供商协助解决，可以提交工单或发送邮件请求支持。

至此，你应该一切配置就绪，可以向 QQ、163、Outlook、Gmail 等邮箱发送测试邮件了。

**********

## 0x04 后台入口

Iredmail配置完成后，后台管理的入口路径通常有以下几个：

1. **iRedAdmin (iRedMail的Web管理控制台)**  
   默认路径：`https://<你的域名或IP>/iredadmin`  
   这是iRedMail自带的邮件服务器管理后台，可以管理邮箱用户、域名、别名等。

2. **Roundcube (Web邮件客户端)**  
   默认路径：`https://<你的域名或IP>/mail`  
   这是iRedMail提供的Web邮件客户端，用户可以通过这个路径登录和收发邮件。

3. **Postfix Admin (如果启用，管理邮箱的Web界面)**  
   默认路径：`https://<你的域名或IP>/postfixadmin`  
   有时iRedMail安装时会附带这个工具，用于管理邮箱和域名。

4. **phpMyAdmin (用于管理MySQL或MariaDB数据库)**  
   默认路径：`https://<你的域名或IP>/phpmyadmin`  
   如果安装了phpMyAdmin，可以通过这个路径管理邮件服务器所用的数据库。

5. **SOGo (如果启用，带有日历、联系人等功能的Web邮件客户端)**  
   默认路径：`https://<你的域名或IP>/SOGo`  
   如果你在安装时选择启用SOGo，可以通过这个路径访问。

**********

这篇分享到这里就结束了~  
由于步骤都是事后整理的，若有任何问题或错误，敬请读者在评论区指出。  
祝大家成功实现自己想要的效果。

2024年10月09日  
于 太原