# OpenAnolis 国密算法库

密码算法库是操作系统的基础组件，在系统安全领域的作用不言而喻。操作系统默认已经内置了大量的密码学库，比如OpenSSL，libgcrypt，gnulib，nettle 是被默认集成到基础操作系统的，它们有一些重复的功能，但也各有侧重的领域，是操作系统不可或缺的安全基石。

本小节会介绍一些支持国密算法的、非常主流的密码算法库，提供给开发者和用户更多的选择。

## OpenSSL

官网：<https://www.openssl.org>

OpenSSL 是一个通用的、强大的、商业级的、功能齐全的工具包，用于通用加密和安全通信。

OpenSSL 的重要性众所周知，这里重点强调一下版本问题。

> 🟢 **1.1.1 稳定版**

目前主流发行版使用的仍然是 1.1.1 版本，这个版本在国密的支持上有一些固有的缺陷：

* 不支持 SM2 的签名验签，因为基于可辨别用户ID的Za值计算在这个版本中未实现
* 国外主流的发行版的包默认没有编译国密 SM2、SM4 模块，CentOS上就是如此

由于技术上和兼容性的原因，这个版本目前很难升级到最新的社区版本，因此在主流的发行版本中基本是无缘使用国密的。

> 🟢 **3.0.x 稳定版**

社区最新的稳定版本是 3.0，这个版本对国密的支持已经比较完善，并且支持了国密的指令集优化。用户如果自行编译可以完整使能国密的能力。

> 🟢 **龙蜥社区 1.1.1 版本**

从目前情况来看，对于一个操作系统发行版，要完全从 1.1.1 切换到 3.0 还需要较长的时间，因此龙蜥社区在 1.1.1 版本的基础上，在保证兼容性和稳定性的前提下，补全了国密能力上的缺陷，并且做为操作系统默认库在 Anolis OS 8.8 中集成发布，详细信息可参考[**Anolis OS
国密开发指南**](anolisos_guide.html)。

## libgcrypt

官网：<https://www.gnupg.org/software/libgcrypt/index.html>

不像 OpenSSL 还包括了安全协议，libgcrypt 是一个纯粹的密码算法库，就国密算法的性能来说，libgcrypt 的国密算法优化是做的最充分的，是目前国密算法的天花板，Linux 内核国密算法的部分优化也是先在 libgcrypt 实现后才移植到内核的。

Libgcrypt 是一个通用密码库，最初基于 GnuPG 的代码。 它为几乎所有的密码提供支持：

* 对称密码算法 (AES、Arcfour、Blowfish、Camellia、CAST5、ChaCha20 DES、GOST28147、Salsa20、SEED、Serpent、Twofish、SM4)
* 模式 (ECB、CFB、CBC、OFB、CTR、CCM） ,GCM,OCB,POLY1305,AESWRAP)
* 哈希算法 (MD2, MD4, MD5, GOST R 34.11, RIPE-MD160, SHA-1, SHA2-224, SHA2-256, SHA2-384, SHA2-512, SHA3-224 , SHA3-256, SHA3-384, SHA3-512, SHAKE-128, SHAKE-256, TIGER-192, Whirlpool, SM3)
* MAC (HMAC 用于所有哈希算法, CMAC 用于所有密码算法, GMAC-AES, GMAC-CAMELLIA, GMAC-TWOFISH、GMAC-SERPENT、GMAC-SEED、Poly1305、Poly1305-AES、Poly1305-CAMELLIA、Poly1305-TWOFISH、Poly1305-SERPENT、Poly1305-SEED)
* 公钥算法 (RSA、Elgamal、DSA、ECDSA、EdDSA、ECDH、SM2)
* 大整数函数、随机数和大量的支持函数

libgcrypt 是很多基础组件依赖的密码库，比如 gpg，systemd，qemu，postgresql，还有许多桌面环境的库，音视频组件，蓝牙都依赖于 libgcrypt 提供的密码安全机制，还有部分会选择依赖libgcrypt，比如 curl，cryptsetup 等会选择依赖 OpenSSL，libgcrypt 算法库，用户需要自行构建来选择不同的密码库。

libgcrypt 从 1.9.0 版本开始陆续支持了国密算法和国密的指令集优化。

## GmSSL

项目地址：<https://github.com/guanzhi/GmSSL>

GmSSL 是一个开源密码工具包，为 GM/T 系列标准中规定的中国国家密码算法和协议提供一级支持。 作为 OpenSSL 项目的一个分支，GmSSL 提供了与 OpenSSL 的 API 级兼容性并保持了所有的功能。 现有项目（例如 Apache Web 服务器）可以轻松地移植到 GmSSL，只需进行少量修改和简单的重建。

自2014年底首次发布以来，GmSSL已入选开源中国六大推荐密码项目之一，并获得2015年中国Linux软件大奖。

该密码库的特点：

* 支持中国GM/T密码标准。
* 支持中国厂商的硬件密码模块。
* 具有商业友好的开源许可证。
* 由北京大学密码学研究组维护。

GmSSL 将支持以下所有 GM/T 加密算法：

* SM3 (GM/T 0004-2012)：具有 256 位摘要长度的密码哈希函数。
* SM4（GM/T 0002-2012）：密钥长度为128位，块大小为128位的块密码，也称为SMS4。
* SM2（GM/T 0003-2012）：椭圆曲线密码方案，包括数字签名方案、公钥加密、（认证）密钥交换协议和一种推荐的256位素数域曲线sm2p256v1。
* SM9（GM/T 0044-2016）：基于配对的密码方案，包括基于身份的数字签名、加密、（认证）密钥交换协议和一条256位推荐BN曲线。
* ZUC（GM/T 0001-2012）：流密码，采用128-EEA3加密算法和128-EIA3完整性算法。
* SM1和SSF33：密钥长度为128位，块大小为128位的块密码，没有公开说明，只随芯片提供。

GmSSL 支持许多有用的加密算法和方案：

* 公钥方案：Paillier、ECIES（椭圆曲线集成加密方案）
* 基于配对的密码学：BF-IBE、BB1-IBE
* 块密码和模式：Serpent、Speck
* 块密码模式：FPE（格式保护加密）
* 基于SM3/SM4的OTP（一次性密码）（GM/T 0021-2012）
* 编码：Base58

ECDSA、RSA、AES、SHA-1 等 OpenSSL 算法在 GmSSL 中仍然可用。

## nettle

官网：<http://www.lysator.liu.se/~nisse/nettle>

Nettle 是一个相对低层的加密库，旨在轻松适应各种工具包和应用程序。它开始于2001年的lsh的低级加密函数的集合。自2009年6月以来，Nettle是GNU软件包。

Nettle 的定位跟 libgcrypt 有点类似，是很多基础组件选择依赖的一个密码学库。

从提供的 API 上来看，Nettle 没有对算法做更高层次的抽象，每个不同的算法都有一套更易理解的接口，开发者也会更容易上手。

Nettle 从 3.8 版本开始支持了 SM3 算法，最新的开发分支已经合入了 SM4 算法，会在下一个 release 版本会发布。

## gnulib

官网：<https://www.gnu.org/software/gnulib>

从名字可以看出，gnulib 并不是一个纯密码算法的库，它的定位是 GNU 的公共代码库，旨在 GNU 包的源代码级别之间共享。

之所以在这里提 gnulib，是因为这个库里面实现了常用的哈希算法，也包括SM3算法，gnulib 里的 哈希算法主要是为 coreutils 包里的 sha*sum, md5sum 系列工具提供支持的，当然开发者也可以基于 gnulib 构建自己的程序。

gnulib 是在 2017 年 10 月支持了 SM3 算法，由阿里巴巴张佳贡献。

## coreutils

coreutils 支持了大量的计算哈希的工具，比如 cksum，md5sum，b2sum，sha*sum 等，这些工具是紧密依赖于 gnulib 库的。

2017 年 10 月，在 gnulib 库支持了 SM3 之后，我们便向 coreutils 社区提交了 sm3sum 工具的支持，coreutils 社区却迟迟不愿接收，因为 SM3 算法的IV向量没有明确的来历说明，社区对算法的安全性有质疑，虽然披时 SM3 已经是 ISO 的国际标准算法。社区人员认为 SM3 在 gnulib 中作为库提供给开发者是没有问题的，因为开发者具备也应该具备判断一个算法是否安全的能力，但是在 coreutils 中提供一个 sm3sum 的工具提供给终端用户会引起用户的误导，用户可能误认为算法安全性是得到保证的，尤其是在 SM3 算法安全性被质疑的前提下。

直到四年后的 2021 年 9 月，在包括Linux 内核，libgcrypt，OpenSSL 等主流的密码算法社区都支持了SM3算法后，在龙蜥的几次推动下，coreutils 社区终于不再质疑 SM3 的安全性问题，但是社区也不愿意再多引入一个工具，应该把这个哈希算法整合为一个工具，因为类似 *sum 的工具太多了。

因此，社区提出一个 `cksum -a [algo]` 的方案，通过给 cksum 工具添加一个算法参数，整合了目前 coreutils 中支持的所有哈希算法，为了兼容考虑，之前的 *sum 工具也继续保留了，SM3 是唯一仅在 cksum 工具中支持的算法，当然这并不是优点，使用习惯上也会有一些差异，用户需要通过 `cksum -a sm3` 来计算 SM3
哈希，除这个区别外，其它用法跟 md5sum 类似。

coreutils 从 9.0 版本开始支持 SM3 的哈希计算。

## RustCrypto

这是一个纯 Rust 编写的密码算法库，供 Rust 开发者使用。

该项目维护着数十个流行的 crate，都提供密码算法的纯 Rust 实现，主要包括以下算法：

* 非对称加密：椭圆曲线、rsa
* 加密编码格式：const-oid、der、pem-rfc7468、pkcs8
* 数字签名：dsa、ecdsa、ed25519、rsa
* 椭圆曲线：k256、p256、p384
* 哈希函数：blake2、sha2、sha3、sm3
* 密钥派生函数：hkdf、pbkdf2
* 消息认证码：hmac
* 密码哈希：argon2、pbkdf2、scrypt
* Sponge 函数：ascon、keccak
* 对称加密：aes-gcm、aes-gcm-siv、chacha20poly1305、sm4
* Traits：aead、密码、摘要、密码哈希、签名

该算法库目前支持 SM3 和 SM4 算法。

## Intel IPP

项目地址：<https://github.com/intel/ipp-crypto>

Intel Integrated Performance Primitives (Intel IPP) Cryptography 是一个安全、快速且轻量级的密码学库，针对各种 Intel CPU 进行了高度优化。

该库提供了一套全面的常用于加密操作的函数，包括：

* 对称密码学原语函数：

    - AES（ECB、CBC、CTR、OFB、CFB、XTS、GCM、CCM、SIV）
    - SM4（ECB、CBC、CTR、OFB、CFB、CCM）
    - TDES（ECB、CBC、CTR、OFB、CFB）
    - RC4

* 单向哈希原语：

    - SHA-1、SHA-224、SHA-256、SHA-384、SHA-512
    - MD5
    - SM3

* 数据认证原语函数：

    - HMAC
    - AES-CMAC

* 公钥加密函数：

    - RSA、RSA-OAEP、RSA-PKCS_v15、RSA-PSS
    - DLP、DLP-DSA、DLP-DH
    - ECC（NIST 曲线）、ECDSA、ECDH、EC-SM2

* 多缓冲区 RSA、ECDSA、SM3、x25519
* 有限域算术函数
* 大整数算术函数
* PRNG/TRNG 和质数生成

使用英特尔 IPP 密码术的原因：

* 安全性（秘密处理功能的恒定时间执行）
* 专为小尺寸设计
* 针对不同的 Intel CPU 和指令集架构进行了优化（包括硬件加密指令 SSE 和 AVX 的支持）
* 可配置的 CPU 分配以获得最佳性能
* 内核模式兼容性
* 线程安全设计

{{#template template/footer.md}}
