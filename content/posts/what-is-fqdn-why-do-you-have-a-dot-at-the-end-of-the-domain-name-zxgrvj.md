---
title: FQDN 是什么？（为啥域名末尾带个点 .）
slug: what-is-fqdn-why-do-you-have-a-dot-at-the-end-of-the-domain-name-zxgrvj
url: >-
  /post/what-is-fqdn-why-do-you-have-a-dot-at-the-end-of-the-domain-name-zxgrvj.html
date: '2025-02-16 22:08:03+08:00'
lastmod: '2025-02-16 22:19:11+08:00'
toc: true
isCJKLanguage: true
---



最近公司项目上有个需求是开放一个公网接口给到外部协作方以供回调。之前服务都是通过 BFF 给 C 端提供服务的，这次是纯后端之间的通信，所以需要我们的后端服务也整个公网域名以满足服务间的公网通信。

申请公网域名就是需要申请一个子域名到 GSLB 的 CNAME 记录。在创建 CNAME 记录的是否发现 GSLB 域名的后面有个 .，起初还以为是个英文句号，但仔细看发现并不是，就是在域名后有一个 .。今天我们就来看看为啥域名末尾带个 .。

‍
