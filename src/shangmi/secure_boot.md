# 国密安全启动

## UEFI安全启动简介

Secure Boot是UEFI的一个子规范，只是UEFI的一个部分。两者的关系是局部与整体的关系。

Secure Boot的目的，是防止恶意软件侵入。它的做法就是采用密钥。UEFI规定，主板出厂的时候，可以内置一些可靠的公钥。然后，任何想要在这块主板上加载的操作系统或者硬件驱动程序，都必须通过这些公钥的认证。也就是说，这些软件必须用对应的私钥签署过，否则UEFI固件拒绝加载。由于恶意软件不可能通过认证，因此就没有办法感染Boot。

这个设想是好的。但是，UEFI没规定哪些公钥是可靠的，也没规定谁负责颁发这些公钥，都留给硬件厂商自己决定。

(国密支持的安全启动的价值和意义，地缘冲突 / 俄乌战争 / 国密CA自我掌控）

## 基于国密的安全启动支持

### 依赖与工具（Dependencies）
Secure Boot需要固件，bootloader的支持以及用户态的工具集用于签名和密钥管理。
* 固件 
  - edk2：固件开发规范，具备基础的国密算法支持，主要是SM2签名验签。
* bootloader 
  - shim：功能上会注册MOK secure boot协议，该协议用来劫持UEFI secure boot认证机制，以便能够使用MOK来验证下一阶bootloader以及内核。目前的shim版本已经具备。
  - grub2：支持利用shim注册的MOK secure boot协议来验证内核。主要是SM2算法的PKCS#7格式签名验签。
* 工具集 
  - sbsigntools：签名工具，允许用户使用该工具对bootloader和内核进行签名。目前版本 0.9.3。
  - mokutils：MOK secure boot密钥管理工具。mokutils的rpm在shim-signed中提供。
  - efitools：UEFI secure boot密钥管理工具。目前版本 1.9.2。

### 国密证书初始化与部署（Initial Provisioning of SM2 Certificates and Hashes）

### 更新与修改

* 更新 PK
* 更新 KEK
* 更新 DB 与 DBX
* 更新 MOK 与 MOKX

### 镜像签名与验证

### 国密 BootLoader 支持

* grub
* shim

### UEFI 安全启动策略加强 / 定制化 / 创新（Mike）
