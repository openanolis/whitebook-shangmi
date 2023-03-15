# Anolis OS 国密开发指南

龙蜥社区一直致力于在操作系统中集成国密的全部能力，以开箱即用作为目标，让用户可以如丝般顺滑的从国际算法切换到国密算法的生态上，为此，龙蜥社区在诸多的基础软件中做了很多的兼容和优化工作。Linux 内核和 OpenSSL 作为其中最重要的项目做了全面的国密适配，本章重点介绍龙蜥社区在 OpenSSL 适配国密上的进展以及用户使用指南。

## OpenSSL 1.1 内置完整国密能力

众所周知，OpenSSL 是操作系统中最基础的密码学工具库，几乎被默认集成到所有基于Linux和BSD的发行版中，为操作系统提供最基本的基于密码学的信息安全能力。

OpenSSL整个软件包大概可以分成三个主要的功能部分：密码算法库，SSL/TLS协议库以及应用程序。作为一个基于密码学的安全开发包，OpenSSL提供的功能相当强大和全面，囊括了主要的密码算法、常用的密钥和证书封装管理功能以及 SSL/TLS 协议，并提供了丰富的应用程序供测试或其它目的使用。

> 🟢 **挑战**

OpenSSL 3.0
是OpenSSL社区的最新版本,于2020年4月发布，到目前为止已经发布了几个稳定版本，得益于社区开发者的持续贡献，这个版本已经支持了完整的国密能力，包括SM2、SM3、SM4算法以及使用这些算法的密钥和证书体系，甚至是SM3、SM4算法的优化也有了部分架构的支持。

相较于基础软件较长的升级迭代周期来说，OpenSSL 3.0 仍然是一个很新的版本，一个主要原因是OpenSSL 3.0 发布时间还较短，其次OpenSSL 3.0 在软件架构上做了较大的变动，部分原来被广泛使用的函数被标记为了过时函数，对于算法的使用方式也有了新的规则，各发行版出于稳定性和兼容性的考虑，目前系统默认内置的依然是 1.1.1 版本。用户若是想体验 OpenSSL 3.0 的功能需要使用第三方仓库或者是下载源码自行编译。

Anolis OS 23 系列的发行版已率先集成 OpenSSL 3.0 作为默认的密码学库，并提供与社区版本一致的体验，用户可以参考 OpenSSL 社区手册使用完整的国密能力。

> 🟢 **Anolis 8.8 内置国密**

但目前主流的 Anolis OS 8.x 系列，默认预装的还是 OpenSSL 1.1.1，一直以来，OpenSSL 1.1.1 都是国密的一个遗憾，也是一个痛点，这个版本的 SM2 签名验签能力是有缺陷的，也不支持国密标准的可辨别标识，但这个版本又是主流的发行版所使用的版本，甚至在某些发行版中，比如CentOS中，SM3和SM4算法都是被删掉的。鉴于以上原因，在 Anolis OS 8.8 中，在保证兼容性和稳定性的前提下，龙蜥社区在系统默认集成的OpenSSL 1.1.1 版本上支持了完整的SM2签名和验签能力，并做了开源，发行版本rpm也默认开启了SM3和SM4算法，带给用户开箱即用的国密使用体验。

**_代码仓库：<https://github.com/openanolis/openssl/tree/anolis_sm234>_**

**_Anolis OS RPM：<https://gitee.com/src-anolis-os/openssl/tree/a8>_**

> 🟢 **用户手册**

根据规范 `GM/T 0003.2-2012` 和 `GM/T 0003.5-2012` 的定义，SM2 算法签名时需要一个计算了 Za 的哈希值，这个Za是通过固定的椭圆曲线参数、公钥数据以及一个用户输入的可辨别ID生成的。因此，就必须在用户界面添加一个指定用户ID的参数才能支持完整的SM2签名和验签能力，在龙蜥社区的OpenSSL库中，通过特殊参数向OpenSSL内部传递该参数。

因此，龙蜥社区为签名工具扩展了`sigopt`参数，为验签工具添加了`vfyopt`参数来支撑用户界面的工具，SM2的可辨别标识通过`distid`子参数指定，比如可以使用`-sigopt "distid:1234567812345678"`在签名时指定SM2的可辨别标识值，终端用户可以通过命令行工具openssl来调用SM2的完整能力。

除此之外，其它完全兼容社区的OpenSSL 1.1.1，以下是使用这两个参数的具体例子，`1234567812345678` 是规范推荐使用的用户ID，用户可根据需要自行指定。

* 生成一个自签名的SM2根证书

    ```bash
    # Generate a self signed root certificate
    
    openssl ecparam -genkey -name SM2 -text -out ca.key
    
    openssl req -verbose -new -days 10000 -x509 -sm3 \
        -sigopt "distid:1234567812345678" \
        -config genkey.conf -key ca.key -out ca.cert
    ```

* 从私钥生成SM2算法的证书请求并验证

    ```bash
    # Generate a certificate request from private key and verify it
    
    openssl ecparam -genkey -name SM2 -text -out sm2.key
    
    openssl req -verbose -new -sm3 \
        -sigopt "distid:1234567812345678" \
        -config genkey.conf -key sm2.key -out sm2.csr
    
    openssl req -verbose -verify \
        -vfyopt "distid:1234567812345678" \
        -config genkey.conf -noout -text -in sm2.csr
    ```

* 使用SM2算法签名证书请求并验证
    
    ```bash
    # Sign a SM2 certificate request using the CA certificate and verify it
    
    openssl x509 -req -days 10000 -sm3 \
        -sigopt "distid:1234567812345678" \
        -vfyopt "distid:1234567812345678" \
        -CA ca.cert -CAkey ca.key -CAcreateserial \
        -extfile genkey.conf -extensions v3_ca \
        -in sm2.csr -out sm2.cert
    
    openssl x509 -in sm2.cert -outform DER -out sm2.der
    
    openssl verify -verbose -show_chain \
        -x509_strict -CAfile ca.cert sm2.cert
    ```

其中 genkey.conf的内容如下：

```
[ req ]
distinguished_name = req_distinguished_name
prompt = no
string_mask = utf8only
x509_extensions = v3_ca

[ req_distinguished_name ]
O = OpenAnolis-CA
CN = OpenAnolis certificate signing key
emailAddress = ca@openanolis-ca

[ v3_ca ]
basicConstraints=CA:TRUE
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid:always,issuer

[ skid ]
basicConstraints=CA:TRUE
subjectKeyIdentifier=12345678
authorityKeyIdentifier=keyid:always,issuer
```

> 🟢 **开发指南**

为了支持SM2的签名验签能力，龙蜥社区为 OpenSSL 内部的验签添加了三个新的API：

```c
int ASN1_item_verify_ctx(const ASN1_ITEM *it, X509_ALGOR *a,
                         ASN1_BIT_STRING *signature, void *asn, EVP_PKEY *pkey,
                         EVP_MD_CTX *ctx);

int X509_verify_ctx(X509 *a, EVP_PKEY *r, EVP_MD_CTX *ctx);

int X509_REQ_verify_ctx(X509_REQ *a, EVP_PKEY *r, EVP_MD_CTX *ctx);
```

开发者通过他们可以调用完整的国密签名验签能力。这几个API扩展了原来不带 ctx 后缀的API能力，通过ctx可以预先指定额外的参数，这里主要是指SM2算法的用户可辨别标识，具体用法可参考apps目录里工具的调用方法或者sign系列的类似函数。

**_X509_sign 参考：<https://www.openssl.org/docs/man1.1.1/man3/X509_sign_ctx.html>_**

{{#template template/footer.md}}
