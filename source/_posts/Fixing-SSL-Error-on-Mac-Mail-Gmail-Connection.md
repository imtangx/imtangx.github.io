---
title: Mac 邮件 Gmail 账户出现 SSL 错误连接失败解决方法
date: 2025-12-02 23:29:18
tags: ['Mail']
---

## Problem

在使用 Mac 自带邮件 App 的时候，在一段时间发现总是连不上我的 Gmail 账户，会显示如下：

![alt text](/images/gmail-ssl-error-1.png)

![alt text](/images/gmail-ssl-error-2.png)

## Reason

首先 Mac 拉取/发送 Gmail 邮件走的是 SMTP/IMAP 协议，它们的端口通常不是 433（HTTP）。

而默认的 GFW 会屏蔽 Gmail 相关网站，但它没有屏蔽 SMTP/IMAP 的端口，因此即使没有使用科学上网，Mac 邮件 App 也可以正常拉取/发送 Gmail 邮件（当然需要先科学上网添加账户）。所以我在刚开始使用 Mac 邮件 App 的时候没有发生这个问题。

当后续我的 Clash 常驻后，发现开始出现上述问题，迫不得已使用网站 Gmail。后续发现原因在于 Clash 普遍会将常见邮件端口屏蔽，导致 SMTP/IMAP 服务在上游代理节点失效。

## Solution

在 Clash 中设置特殊规则，将 SMTP/IMAP 协议接口走直连服务。这里用 Clash Verge 做示范：

![alt text](/images/clash-solve-ssl-error.png)

详细的操作可见官方文档：https://www.clashverge.dev/guide/rules.html?h=%E9%85%8D%E7%BD%AE
