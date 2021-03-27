---
title: "Curator - 管理你的 Elasticsearch 索引"
description: "Curator - 管理你的 Elasticsearch 索引"
date: 2021-03-27T00:43:33+08:00
lastmod: 2021-03-27T00:43:33+08:00
weight: 40
keywords: [ "Curator ", "Elasticsearch " ]
tags: [ "Curator", "技术", "Elasticsearch " ]
categories: ["技术"]
author: "ggguo"
autoCollapseToc: true
draft: false
---

## 前言

1. 欢迎在文末留言，共同进步。
2. 本文采用署名 - 非商业性使用 - 相同方式共享 4.0 国际 (CC BY-NC-SA 4.0) 许可协议，转载请注明出处！

## 准备工作

我这里以 CentOS 7 为例来进行说明。

### 安装

官方推荐最简单的安装方式就是使用 `pip`，没错，这货是 `python` 写的。

1. 最简化安装的 CentOS，虽然自带了 `python`，但是没有相关头文件，无法安装 `pip`，干脆编译安装 `python3`。既然要编译，先装个开发工具包再说

   复制

   ```
   yum -y groupinstall 'Development Tools'
   ```

2. 编译 `python3`，一定要注意 `OpenSSL` 支持是不是启用了，否则后面没办法下载各种包了，还需要这些依赖，不安装会报错的（都是试错过后的总结，都是眼泪😢）

   复制

   ```
   yum -y install openssl openssl-devel zlib zlib-devel libffi-devel
   ```

3. 下载 `python3` 的源码包，然后解压编译并执行 `configure`

   复制

   ```
   # 下载解压
   wget https://www.python.org/ftp/python/3.7.3/Python-3.7.3.tar.xz
   tar -Jxf Python-3.7.3.tar.xz
   cd Python-3.7.3/
   # 执行配置
   ./configure
   ```

4. 执行完 `configure` 后，注意看 `OpenSSL` 支持是不是启用了。如图是成功启用的。
   [![包含 OpenSSL 支持](https://gitee.com/guoyongsoft/image/raw/master/img/20210326164046.png)](https://www.wdmcheng.cn/Curator-管理你的Elasticsearch索引/OpenSSL.png)

   [包含 OpenSSL 支持](https://www.wdmcheng.cn/Curator-管理你的Elasticsearch索引/OpenSSL.png)

   

5. 后面编译安装

   复制

   ```
   make
   make install
   ```

6. 这个时候没有 `pip` 命令，只有 `pip3`。不用别名处理了，顺手更新一下 `pip` 版本，也会自动安装 `pip` 这个命令

   复制

   ```
   pip3 install --upgrade pip
   ```

7. 进入正题，安装 `curator`

   复制

   ```
   pip3 install elasticsearch-curator
   ```

### 配置

1. 执行 `curator` 命令时，默认读取的配置文件在 `~/.curator/curator.yml`。创建配置文件，内容如下

   复制

   ```
   ---
   # Remember, leave a key empty if there is no value.  None will be a string,
   # not a Python "NoneType"
   client:
     # ES 节点列表
     hosts:
       - 10.0.79.14
       - 10.0.79.22
       - 10.0.79.23
       - 10.0.79.24
     port: 9200
     url_prefix:
     use_ssl: False
     certificate:
     client_cert:
     client_key:
     ssl_no_validate: False
     http_auth: username:password
     timeout: 30
     master_only: False
   
   logging:
     loglevel: INFO
     # logfile 文件路径，一定要保证文件的父级文件夹存在
     logfile: /data/es-curator-log/es-curator.log
     logformat: default
     blacklist: ['elasticsearch', 'urllib3']
   ```

   上面的 `http_auth` 部分，是用户名和密码，用 `:` 分隔。更多配置文件写法，参考 官方文档

2. 创建任务文件，这里提供一个参考，内容如下。详细信息参考官方文档

   复制

   ```
   ---
   # Remember, leave a key empty if there is no value.  None will be a string,
   # not a Python "NoneType"
   #
   # Also remember that all examples have 'disable_action' set to True.  If you
   # want to use this action as a template, be sure to set this to False after
   # copying it.
   actions:
     1:
       action: delete_indices
       description: >-
         Delete indices older than 31 days (based on index name), for -pro- indices.
   
         删除31天之前的 -pro- 索引
       options:
         ignore_empty_list: True
         timeout_override: 300
         continue_if_exception: True
         disable_action: False
       filters:
       - filtertype: pattern
         kind: regex
         value: '.*-pro-.*'
         exclude:
       - filtertype: age
         source: name
         direction: older
         timestring: '%Y.%m.%d'
         unit: days
         unit_count: 31
         exclude:
     2:
       action: delete_indices
       description: >-
         Delete indices older than 7 days (based on index name), except -pro- indices.
   
         删除7天之前的非 -pro- 索引
       options:
         ignore_empty_list: True
         timeout_override: 300
         continue_if_exception: True
         disable_action: False
       filters:
       - filtertype: pattern
         kind: regex
         value: '.*-pro-.*'
         exclude: True
       - filtertype: age
         source: name
         direction: older
         timestring: '%Y.%m.%d'
         unit: days
         unit_count: 7
         exclude:
     3:
       action: delete_indices
       description: >-
         Delete indices older than 3 days (based on index name), for system indices.
   
         删除3天之前的系统索引
       options:
         ignore_empty_list: True
         timeout_override: 300
         continue_if_exception: True
         disable_action: False
       filters:
       - filtertype: pattern
         kind: regex
         value: '^\..*-.*'
         exclude:
       - filtertype: age
         source: name
         direction: older
         timestring: '%Y.%m.%d'
         unit: days
         unit_count: 3
         exclude:
   ```

3. 现在可以使用 `--dry-run` 参数试运行测试一下（这个参数的作用就是不会真正的去执行，只是模拟）。`--config ~/.curator/curator.yml` 可省略

   复制

   ```
   curator --config ~/.curator/curator.yml --dry-run ~/delete_indecies.yml
   ```

### 配置定时任务

先说一下用法，后面再列几篇文章供参考

1. 执行 `crontab -e`，会自动打开 `vim` 编辑器
2. 我这里是每隔 6 个小时执行一次，因此添加一行 `0 0,6,12,18 * * * /usr/local/bin/curator ~/delete_indecies.yml`
3. 保存后大功告成

### 参考资料

- Curator 文档
- crontab 参数解释
- Linux 定时任务

------------- 本文结束感谢您的阅读 -------------

- **本文作者：** 雾都迷城
- **本文链接：** [https://www.wdmcheng.cn/Curator - 管理你的 Elasticsearch 索引 /](https://www.wdmcheng.cn/Curator-管理你的Elasticsearch索引/)
- **版权声明：** 本博客所有文章除特别声明外，均采用 BY-NC-SA 许可协议。转载请注明出处！