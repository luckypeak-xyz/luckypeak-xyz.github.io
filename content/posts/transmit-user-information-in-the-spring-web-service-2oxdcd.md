---
title: 在 Spring Web 服务中传递用户信息
slug: transmit-user-information-in-the-spring-web-service-2oxdcd
url: /post/transmit-user-information-in-the-spring-web-service-2oxdcd.html
date: '2025-01-11 08:26:30+08:00'
lastmod: '2025-01-11 08:51:55+08:00'
tags:
  - SpringBoot
categories:
  - 快乐开发一点通
keywords: SpringBoot
toc: true
isCJKLanguage: true
---



面向 C 端的服务结构大致如下：

<iframe src="/widgets/widget-excalidraw/" data-src="/widgets/widget-excalidraw/" data-subtype="widget" border="0" frameborder="no" framespacing="0" allowfullscreen="true"></iframe>

用户信息以 token 的形式从 客户端 经过 网关（或者 BFF）完成鉴权后，以 account_id 的形式在后端服务内部及后端服务之间传递。

## 基于 ThreadLocal 在代码内传递用户信息

## 基于 `feign.RequestInterceptor`​ 在服务间传递用户信息
