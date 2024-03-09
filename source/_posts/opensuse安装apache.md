toc: true
title: opensuse安装apache
description: opensuse安装apache
date: 2015-12-04 16:31:06
tags: [opensuse, apache]

---
# 安装
 
zypper install apache2
zypper install apache2-doc
zypper install apache2-example-pages
zypper install yast2-http-server
 
# 配置
 
## 用户自定义配置目录
 
```bash
cd
mkdir .apache2
```
 
## 通过yast2配置apache
 
/usr/sbin/yast2 http-server
+ 配置监听地址
+ 增加include目录`IncludeOptional ~/.apache2/*.conf`
 
## 配置IP地址访问限制
 
只允许列出的IP地址访问
 
cat > ~/.apache2/allowip.conf << EOF
 
<Location />
Require ip 10.6.86.226
Require ip 10.6.86.225
Require ip 10.6.86.1
</Location>
 
EOF
 
## 配置proxy
 
cat > ~/.apache2/proxy.conf << EOF
 
ProxyRequests On
ProxyVia On
 
EOF
 
# 系统定义的配置文件
 
主要配置文件
/etc/apache2/http.conf
 
系统信息
/etc/apache2/mod_info.conf
 
系统状态
/etc/apache2/mod_status.conf
 
虚拟主机
/etc/apache2/vhost/*.conf
 
自定义配置
/etc/apache2/conf.d
 
其它系统定义的配置文件
/etc/apache2/listen.conf
/etc/apache2/uid.conf
/etc/apache2/errors.conf
/etc/apache2/sysconfig.d/*.conf
/etc/apache2/server-tuning.conf
 
**又增加了用户主目录自定义的配置文件**
~/.apache2/*.conf
 
# HTTPS
 
证书都要费用。这里生成一个自定义的证书，不能通过浏览器的检测，但是能将http加密传输，也有用处。
 
参考http://www.mike.org.cn/articles/ubuntu-config-apache-https/
 
opensuse版本的apache将证书放在`/etc/apache2/ssl.crt`，文件名取`apache.pem`
 
```
openssl req -x509 -newkey rsa:1024 -keyout /etc/apache2/ssl.crt/apache.pem -out /etc/apache2/ssl.crt/apache.pem -nodes -days 999
 
writing new private key to
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:CN　←输入国家代码
State or Province Name (full name) [Some-State]:Shanghai　← 输入省名
Locality Name (eg, city) []:Shanghai　←输入城市名
Organization Name (eg, company) [Internet Widgits Pty Ltd]:MIKE　← 输入公司名
Organizational Unit Name (eg, section) []:MIKE　← 输入组织单位名
Common Name (eg, YOUR name) []:opensuse　← 输入主机名
Email Address []:admin@opensuse.com　←输入电子邮箱地址
```
**注：在要求输入Common Name(eg, YOUR name)时，输入你的主机名。Common Name必须和httpd.conf中server name必须一致。**
 
 
先通过模版生成一个文件
 
```bash
cp /etc/apache2/vhost.d/vhost-ssl.template ~/.apache2/vhost-ssl.conf
```
编辑其中第43行，注释掉第44行
 
```
43行：          SSLCertificateFile /etc/apache2/ssl.crt/apache.pem
```
 
重启apache服务`sudo systemctl restart apache2`
 
