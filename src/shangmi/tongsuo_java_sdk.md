# Tongsuo-Java-SDK

## 简介

Tongsuo-Java-SDK提供了一个面向Java生态的现代密码学算法和安全通信协议的开源基础密码库. 该库在实现上完全遵循OpenJDK Security Provider所定义的JCA/JCE/JSSE框架与接口, 因此具备良好的兼容性与可插件化. 此外, Tongsuo-Java-SDK基于Tongsuo作为底层实现进行开发, 采用Native方式实现加解密与安全通信握手协议等核心逻辑。一方面可提升加解密的性能，降低系统CPU利用率; 另一方面增强对不同架构芯片的适配和性能优化能力(比如ARM架构倚天).

## 核心特性

Tongsuo-Java-SDK支持以下核心特性:
1. 常规的国际标准加解密算法和SSL/TLS协议
2. 支持RFC 8998，即在TLS 1.3中使用商用密码算法
3. 支持GB/T 38636-2020 TLCP标准，即安全传输协议
4. 符合JCA/JCE/JSSE接口标准

## 密码认证

由于加解密和安全通信握手协议均由Tongsuo实现, 得益于Tongsuo获得了国家密码管理局商用密码检测中心颁发的商用密码产品认证证书, Tongsuo-Java-SDK也能满足我国商用密码技术合规的要求.

## 项目社区

项目地址: https://github.com/Tongsuo-Project/tongsuo-java-sdk