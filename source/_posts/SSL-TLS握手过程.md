---
title: SSL/TLS握手过程
date: 2025-04-05 16:42:19
tags:
---
## 1. Client Hello

客户端发送第一个握手消息，包含：

- 支持的SSL/TLS协议版本
- 客户端随机数（Clinet Random）
- 支持的加密套件列表（Cipher Suites）
- 支持的压缩方法

## 2. Serer Hello

服务器回应，包含：

- 选定的SSL/TLS协议版本
- 服务器随机数（Server Random）
- 选定的加密套件
- 选定的压缩方法
- 服务器的证书

## 3. 证书验证

客户端需要验证服务器证书：

- 检查证书是否过期
- 确认证书的颁发机构是否可信
- 验证证书签名
- 检查证书域名是否匹配

## 4. 密钥交换

- 客户端生成Pre-master secret
- 使用服务器公钥加密Pre-master secret
- 发送给服务器
- 双方都通过Client Random、Server Random和Pre-master secret生成主密钥(Master Secret)

## 5. 完成握手

- 客户端发送"Change Cipher Spec"消息，表示后续使用协商的密钥和加密套件
- 客户端发送"Finished"消息
- 服务器发送"Change Cipher Spec"消息
- 服务器发送"Finished"消息

## 6. 开始使用对称密钥加密通信