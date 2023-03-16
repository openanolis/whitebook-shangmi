# TLS 1.3 使用商密套件（RFC 8998）

TLS 1.3 协议在 `RFC 8998` 规范中定义：<https://datatracker.ietf.org/doc/html/rfc8998>

## 重要说明

在tls1.3的标准中，现有的任意算法套件均不强制使用某一个签名算法，而为了国密算法的统一性，我们在RFC8998的定义中明确要求了tls1.3+国密的两个套件：`TLS_SM4_GCM_SM3` 和 `TLS_SM4_CCM_SM3`，需要强制使用SM2作为签名算法(即server端需要配置国密证书)。 然而出于兼容性考虑，现有的大量软件和设备并没有部署国密证书，为保证大家能尽快体验到更快更便捷的国密流程，我们的实现中并不完全强制签名算法为SM2，而是提供了一个开关来由用户选择是否要开启这种强制措施(默认关闭)，在本手册后续会介绍相关用法。

## 签发 SM2 证书

此例使用Tongsuo签发一个自签名的证书：

```sh
# 生成 SM2 私钥
openssl ecparam -genkey -name SM2 -out sm2.key

# 生成证书签名请求
openssl req -new -key sm2.key -out sm2.csr \
    -sm3 -sigopt "sm2_id:1234567812345678"

# 签名证书
openssl x509 -req -in sm2.csr -signkey sm2.key \
    -out sm2.crt -sm3 -sm2-id 1234567812345678 \
    -sigopt "sm2_id:1234567812345678"
```

## 特性使用（s_client/s_server 工具验证）

server端：命令行输入

```sh
openssl s_server \
    -cert test_certs/sm2_cert/sm2-second.crt \
    -key test_certs/sm2_cert/sm2-second.key \
    -accept 127.0.0.1:4433
```

client端：命令行输入

```sh
openssl s_client -connect 127.0.0.1:4433 \
    -tls1_3 -ciphersuites TLS_SM4_GCM_SM3
```

## 在你的client/server中使用（相关api使用）

server端：和标准tls的server一样，无需做任何修改，仅需要注意是否强制签名算法使用SM2(虽然我们在标准中定义了tls1.3+国密的算法套件必须强制使用sm2签名算法，但由于国密证书还不够普及，所以我们在实现上暂时放宽了这个限制，而是提供了一个开关)，通过下面的代码开启/关闭：

```c
/* enable */
SSL_CTX_enable_sm_tls13_strict(ctx);

/* disable */
SSL_CTX_disable_sm_tls13_strict(ctx);
```

client端：client无需做修改，只需要强制指定算法套件为**TLS_SM4_GCM_SM3/TLS_SM4_CCM_SM3**（国密套件的默认优先级低于现有的tls1.3算法套件），通过这种方式设定：

```c
SSL_CTX_set_ciphersuites(ctx, "TLS_SM4_GCM_SM3");
```

{{#template template/footer.md}}
