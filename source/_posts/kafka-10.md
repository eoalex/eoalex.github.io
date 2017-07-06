---
title: 'Kafka入门之十:Kafka的SSL加密和认证'
tags:
  - Kafka
id: 1491
categories:
  - 云计算及虚拟化
  - 运维
date: 2016-12-16 20:53:44
---
## 1. 简介
SSL(Secure Sockets Layer 安全套接层),及其继任者传输层安全（Transport Layer Security，TLS）是为网络通信提供安全及数据完整性的一种安全协议。TLS与SSL在传输层对网络连接进行加密。在SSL中使用密钥交换算法交换密钥；使用密钥对数据进行加密；使用散列算法对数据的完整性进行验证，使用数字证书证明自己的身份。下面我们就Kafka中如何实现及步骤介绍。
> 生成服务器keystore (密钥和证书)> 
> 生成客户端keystore (密钥和证书) > 
> 创建CA证书> 
> 将CA证书导入服务器truststore> 
> 将CA证书导入客户端truststore> 
> 从服务器keystore导出证书> 
> 用CA证书给服务器证书签名> 
> 将CA证书导入服务器keystore> 
> 将CA证书导入客户端keystore> 
> 将已签名服务器证书导入服务器keystore

## 2. 设置证书
以下设置我们以kakfa服务器kafka0为例。

### 2.1 生成服务区Keystore

    keytool -keystore kafka0.keystore.jks -alias kafka0 -validity 365 -storepass test1234 -keypass test1234 -genkey
keystore: 密钥仓库存储证书文件。
validity: 证书的有效时间，单位天
### 2.2 生成客户端Keystore
    keytool -keystore client.keystore.jks -alias kafka0 -validity 365 -storepass test1234 -keypass test1234 -genkey
### 2.3 创建CA证书
    openssl req -new -x509 -keyout ca.key -out ca.crt -days 365 -passout  pass:test1234
### 2.4 将CA证书导入服务器truststore
    keytool -v -keystore kafka0.truststore.jks -alias CARoot -import -file ca.crt -storepass test1234
### 2.5 将CA证书导入客户端truststore
    keytool -v -keystore client.truststore.jks -alias CARoot -import -file ca.crt -storepass test1234
### 2.6 从服务器keystore导出证书
    keytool -keystore kafka0.keystore.jks -alias kafka0 -certreq -file kafka0.crt -storepass test1234
### 2.7 用CA证书给服务器证书签名
    openssl x509 -req -CA ca.crt -CAkey ca.key -in kafka0.crt -out kafka0-signed.crt -days 365 -CAcreateserial -passin pass:test1234
### 2.8 将CA证书导入服务器keystore
    keytool -keystore kafka0.keystore.jks -alias CARoot -import -file ca.crt -storepass test1234
### 2.9 将CA证书导入客户端keystore
    keytool -keystore client.keystore.jks -alias CARoot -import -file ca.crt -storepass test1234
### 2.10 将已签名服务器证书导入服务器keystore
    keytool -keystore kafka0.keystore.jks -alias kafka0 -import -file kafka0-signed.crt -storepass test1234
## 3. 配置kafka broker
配置server.perproties
    
    listeners=PLAINTEXT://kafka0:9092,SSL://kafka0:9093
    ssl.client.auth=required
    ssl.keystore.location=/opt/kafka/kafka0.keystore.jks
    ssl.keystore.password=test1234
    ssl.key.password=test1234
    ssl.truststore.location=/opt/kafka0.truststore.jks
    ssl.truststore.password=test1234
    
>“required”=>客户端身份验证是必需的，“requested”=>客户端身份验证请求，客户端没有证书仍然可以连接。
    
## 4. 配置kafka客户端
配置client.perproties
    
    security.protocol=SSL
    ssl.keystore.location=/opt/kafka/client.keystore.jks
    ssl.keystore.password=test1234
    ssl.key.password=test1234
    ssl.truststore.location=/opt/kafka/client.truststore.jks
    ssl.truststore.password=test1234
## 5. 验证ssl是否生效
    openssl s_client -debug -connect kafka0:9093 -tls1

如果证书没有出现或者有任何其他错误信息，那么你的keystore设置不正确。