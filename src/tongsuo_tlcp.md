# TLCP 安全传输协议

## 编译 NTLS 功能

NTLS 在 Tongsuo 的术语中代指符合 GM/T 0024 SSL VPN 和 TLCP 协议的安全通信协议，其特点是采用加密证书/私钥和签名证书/私钥相分离的方式。
在编译 Tongsuo 的时候，需要显式的指定编译参数方可开启 NTLS 的支持：

```sh
./config enable-ntls
```

## 签发 SM2 双证书

双证书的签发流程是：根CA > 中间CA > 客户端/服务端证书，根CA证书签发中间CA证书，中间CA证书签发客户端和服务端的证书，包括签名证书和加密证书，这些证书的公钥算法都是SM2。

签发 SM2 双证书的详细流程参考官方文档：<https://www.yuque.com/tsdoc/ts/sulazb>

## 特性使用（s_server/s_client工具验证）

测试用证书在`test_certs/double_cert`目录下

server 端：命令行输入

```sh
openssl s_server -accept 127.0.0.1:4433 \
    -enc_cert test_certs/double_cert/SE.cert.pem \
    -enc_key test_certs/double_cert/SE.key.pem \
    -sign_cert test_certs/double_cert/SS.cert.pem \
    -sign_key test_certs/double_cert/SS.key.pem \
    -enable_ntls
```

client 端(测试 ECC-SM2-WITH-SM4-SM3 套件)：命令行输入

```sh
openssl s_client -connect 127.0.0.1:4433 \
    -cipher ECC-SM2-WITH-SM4-SM3 -enable_ntls -ntls
```

client 端(测试 ECDHE-SM2-WITH-SM4-SM3 套件)：命令行输入

```sh
openssl s_client -connect 127.0.0.1:4433 \
    -cipher ECDHE-SM2-WITH-SM4-SM3 \
    -sign_cert test_certs/double_cert/CS.cert.pem \
    -sign_key test_certs/double_cert/CS.key.pem \
    -enc_cert test_certs/double_cert/CE.cert.pem \
    -enc_key test_certs/double_cert/CE.key.pem \
    -enable_ntls -ntls
```

## NTLS API 使用

server 端示例代码如下：

```c
SSL_CTX *ctx;
const char *sign_key_file = "/path/to/sign_key_file";
const char *sign_cert_file = "/path/to/sign_cert_file";
const char *enc_key_file = "/path/to/enc_key_file";
const char *enc_cert_file = "/path/to/enc_cert_file";

/* 生成上下文 */
ctx = SSL_CTX_new(NTLS_server_method());

/* 允许使用国密双证书功能 */
SSL_CTX_enable_ntls(ctx);

/* 设置签名证书 */
SSL_CTX_use_sign_PrivateKey_file(ctx, sign_key_file, SSL_FILETYPE_PEM);
SSL_CTX_use_sign_certificate_file(ctx, sign_cert_file, SSL_FILETYPE_PEM);

/* 设置加密证书 */
SSL_CTX_use_enc_PrivateKey_file(ctx, enc_key_file, SSL_FILETYPE_PEM);
SSL_CTX_use_enc_certificate_file(ctx, enc_cert_file, SSL_FILETYPE_PEM);

/* 后续同标准 tls 流程 */
con = SSL_new(ctx);
```

client端示例代码如下：

```c
SSL_CTX *ctx = NULL;

/* 生成上下文 */
ctx = SSL_CTX_new(NTLS_client_method());

/* 设置算法套件为 ECC-SM2-WITH-SM4-SM3 或者 ECDHE-SM2-WITH-SM4-SM3
 * 这一步并不强制，默认 ECC-SM2-WITH-SM4-SM3 优先
 */
SSL_CTX_set_cipher_list(ctx, "ECC-SM2-WITH-SM4-SM3");

/* 设置签名证书，加密证书，这一步参考服务端相关代码
 * 仅ECDHE-SM2-WITH-SM4-SM3套件需要这一步
 */
```

## 说明

由于国密双证书的握手流程和协议版本号与标准tls流程存在一定的不同，因此我们选择将双证书的实现(代码里命名为 ntls )同现有的 tls 状态机拆分开来，然后在入口处通过对请求的版本号进行识别，然后使其进入正确的状态机。

{{#template template/footer.md}}
