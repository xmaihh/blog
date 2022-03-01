---
title: 如何在macOS使用DNS over Https(DoH)
date: 2022-03-01 16:19:56
categories: macOS
tags: [macOS]
toc: true
description: 新しいことを始める時に一番大切なことは、それを成し遂げたいという情熱です。
---

# 安装HomeBrew
`HomeBrew`官方推荐快速安装[Homebrew](https://brew.sh/)
```shell
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"
```

# 安装cloudflared
用Homebrew快速安装[cloudflared](https://github.com/cloudflare/cloudflared)
```shell
brew install cloudflare/cloudflare/cloudflared
```

# 设置cloudflared
DNS服务节点地址：[iQDNS](https://iqdns.xyz/all.html)

# 创建cloudflared配置文件：
```shell
mkdir -p /usr/local/etc/cloudflared
vim /usr/local/etc/cloudflared/config.yaml
```

粘贴以下内容：
```shell
proxy-dns: true
proxy-dns-upstream:
  - https://a.passcloud.xyz/dns-query
  - https://adh.avpclub.gq/dns-query
  - https://dns.futa.gg/dns-query
  - https://1.1.1.1/dns-query
  - https://1.0.0.1/dns-query
```
保存退出。

# 启动cloudflared
```shell
sudo cloudflared service install
```

cloudflared Service 执行命令启动后，以后开机都会自动启动cloudflared DNS 服务器，会在本机监听port 53。

# 删除cloudflared
```shell
sudo cloudflared service uninstall
```

# 修改macOS系统网络设置
打开macOS的 「系统偏好设置」，依次找到「网络」->「高级」->「DNS」，在DNS服务器填入 本机IP`127.0.0.1`即可。

# Reference
[如何在 macOS 使用 DNS over Https（DoH）](https://www.jkg.tw/p3361/)