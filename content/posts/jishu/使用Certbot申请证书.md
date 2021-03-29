---
title: "使用 Certbot 自动申请并续订阿里云 DNS 免费泛域名证书"
description: "使用 Certbot 自动申请并续订阿里云 DNS 免费泛域名证书"
date: 2021-03-29T14:43:00+08:00
weight: 40
keywords: [ "Certbot ", "DNS " ]
tags: [ "Certbot ", "技术", "DNS " ]
categories: ["技术"]
author: "ggguo"
autoCollapseToc: true
draft: false

---

# 使用 Certbot 自动申请并续订阿里云 DNS 免费泛域名证书


## 背景
Certbot 支持自动申请 LetsEncrypt 的泛域名证书，但是官方插件不支持阿里云，在 GitHub 搜索发现已经有人写好了阿里云 DNS 插件，下面只需要进行简单的配置即可免费申请一个泛域名证书并自动续订。

## 安装 Certbot 和 certbot-dns-aliyun
**本文基于 CentOS 7**

1. 首先安装 Python 3

``` shell
yum install -y python36
```

2. 创建并激活虚拟环境

``` shell
mkdir -p /mnt/certbot
cd /mnt/certbot
python3 -m venv venv
source venv/bin/activate
```

3. 安装 Certbot 和 certbot-dns-aliyun

``` shell
pip install certbot certbot-nginx certbot-dns-aliyun
```

## 申请并配置阿里云 DNS 访问密钥
前往 https://ram.console.aliyun.com 申请阿里云子账号并授予 AliyunDNSFullAccess 权限。然后为子账号创建 AccessKey 并记录。

创建 certbot-dns-aliyun 配置文件：

``` shell

cat > /mnt/certbot/credentials.ini <<EOF
certbot_dns_aliyun:dns_aliyun_access_key = 12345678
certbot_dns_aliyun:dns_aliyun_access_key_secret = 1234567890abcdef1234567890abcdef
EOF
```

修改文件权限

``` shell
chmod 600 /mnt/certbot/credentials.ini
```

## 申请证书

``` shell
/mnt/certbot/venv/bin/certbot certonly \
-a certbot-dns-aliyun:dns-aliyun \
--certbot-dns-aliyun:dns-aliyun-credentials /mnt/certbot/credentials.ini \
-d yourdomain.com \
-d "*.yourdomain.com"
```

**配置自动续订：**

`echo "0 0,12 * * * root python -c 'import random; import time; time.sleep(random.random() * 3600)' && /mnt/certbot/venv/bin/certbot renew -q" | sudo tee -a /etc/crontab > /dev/null`

## 配置 nginx
```nginx

cat > /etc/nginx/conf.d/nginx.header <<EOF`
`listen 80;`
`listen 443 ssl;`
`if ($scheme != https) {`
    `rewrite ^/(.*) https://$server_name/$1 permanent;`
`}` 
`ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;`
`ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;`
`include /etc/letsencrypt/options-ssl-nginx.conf;`
`ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;`
`EOF`
```

```nginx
cat > /etc/nginx/conf.d/yourdomain.com.conf <<EOF
server {
    server_name  yourdomain.com;
    include      /etc/nginx/conf.d/nginx.header;
 
    location / {
        proxy_set_header  Host $host;  
        proxy_set_header  X-Real-IP $remote_addr;  
        proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;  
        proxy_set_header  X-Forwarded-Proto $scheme;  
 
        proxy_pass http://127.0.0.1:8080;
    }
}
EOF
```
————————————————
本文转载自CSDN博主「tabsp」的原创文章，原文链接：https://blog.csdn.net/qq_16417621/article/details/105859886