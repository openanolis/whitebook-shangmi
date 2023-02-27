# TLCP 安全传输协议

## 编译 NTLS 功能

NTLS 在 Tongsuo 的术语中代指符合 GM/T 0024 SSL VPN 和 TLCP 协议的安全通信协议，其特点是采用加密证书/私钥和签名证书/私钥相分离的方式。
在编译 Tongsuo 的时候，需要显式的指定编译参数方可开启 NTLS 的支持：

```sh
./config enable-ntls
```

## 签发 SM2 双证书

根CA > 中间CA > 客户端/服务端证书，根CA证书签发中间CA证书，中间CA证书签发客户端和服务端的证书，包括签名证书和加密证书，这些证书的公钥算法都是SM2。

```sh
export PATH=/opt/tongsuo/bin:$PATH

rm -f *.key *.csr *.crt
rm -rf {newcerts,db,private,crl}
mkdir {newcerts,db,private,crl}
touch db/{index,serial}
echo 00 > db/serial

# sm2 ca
openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:sm2 -out ca.key

openssl req -config ca.cnf -new -key ca.key -out ca.csr -sm3 -nodes -subj "/C=AA/ST=BB/O=CC/OU=DD/CN=root ca"

openssl ca -selfsign -config ca.cnf -in ca.csr -keyfile ca.key -extensions v3_ca -days 3650 -notext -out ca.crt -md sm3 -batch

# sm2 middle ca
openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:sm2 -out subca.key

openssl req -config ca.cnf -new -key subca.key -out subca.csr -sm3 -nodes -subj "/C=AA/ST=BB/O=CC/OU=DD/CN=sub ca"

openssl ca -config ca.cnf -extensions v3_intermediate_ca -days 3650 -in subca.csr -notext -out subca.crt -md sm3 -batch

cat ca.crt subca.crt > chain-ca.crt

# server sm2 double certs
openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:sm2 -out server_sign.key

openssl req -config subca.cnf -key server_sign.key -new -out server_sign.csr -sm3 -nodes -subj "/C=AA/ST=BB/O=CC/OU=DD/CN=server sign"

openssl ca -config subca.cnf -extensions server_sign_req -days 3650 -in server_sign.csr -notext -out server_sign.crt -md sm3 -batch

openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:sm2 -out server_enc.key

openssl req -config subca.cnf -key server_enc.key -new -out server_enc.csr -sm3 -nodes -subj "/C=AA/ST=BB/O=CC/OU=DD/CN=server enc"

openssl ca -config subca.cnf -extensions server_enc_req -days 3650 -in server_enc.csr -notext -out server_enc.crt -md sm3 -batch

# client sm2 double certs
openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:sm2 -out client_sign.key

openssl req -config subca.cnf -key client_sign.key -new -out client_sign.csr -sm3 -nodes -subj "/C=AA/ST=BB/O=CC/OU=DD/CN=client sign"

openssl ca -config subca.cnf -extensions client_sign_req -days 3650 -in client_sign.csr -notext -out client_sign.crt -md sm3 -batch

openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:sm2 -out client_enc.key

openssl req -config subca.cnf -key client_enc.key -new -out client_enc.csr -sm3 -nodes -subj "/C=AA/ST=BB/O=CC/OU=DD/CN=client enc"

openssl ca -config subca.cnf -extensions client_enc_req -days 3650 -in client_enc.csr -notext -out client_enc.crt -md sm3 -batch
```

其中，ca.cnf 文件内容如下：

```
[ ca ]
# `man ca`
default_ca = CA_default

[ CA_default ]
# Directory and file locations.
dir               = ./
certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/db/index
unique_subject    = no
serial            = $dir/db/serial
RANDFILE          = $dir/private/random

# The root key and root certificate.
private_key       = $dir/ca.key
certificate       = $dir/ca.crt

# For certificate revocation lists.
crlnumber         = $dir/crlnumber
crl               = $dir/crl/ca.crl.pem
crl_extensions    = crl_ext
default_crl_days  = 30

# SHA-1 is deprecated, so use SHA-2 instead.
default_md        = sha256

name_opt          = ca_default
cert_opt          = ca_default
default_days      = 365
preserve          = no
policy            = policy_strict

[ policy_strict ]
# The root CA should only sign intermediate certificates that match.
# See the POLICY FORMAT section of `man ca`.
countryName             = optional
stateOrProvinceName     = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ policy_loose ]
# Allow the intermediate CA to sign a more diverse range of certificates.
# See the POLICY FORMAT section of the `ca` man page.
countryName             = optional
stateOrProvinceName     = optional
localityName            = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ req ]
# Options for the `req` tool (`man req`).
default_bits        = 2048
distinguished_name  = req_distinguished_name
string_mask         = utf8only

# SHA-1 is deprecated, so use SHA-2 instead.
default_md          = sha256

# Extension to add when the -x509 option is used.
x509_extensions     = v3_ca

req_extensions = v3_req

[ req_distinguished_name ]
# See <https://en.wikipedia.org/wiki/Certificate_signing_request>.
countryName                     = optional
stateOrProvinceName             = optional
localityName                    = optional
0.organizationName              = optional
organizationalUnitName          = optional
commonName                      = optional
emailAddress                    = optional

[ v3_req ]
# Extensions to add to a certificate request
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = localhost

[ v3_ca ]
# Extensions for a typical CA (`man x509v3_config`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ v3_intermediate_ca ]
# Extensions for a typical intermediate CA (`man x509v3_config`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true, pathlen:0
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ crl_ext ]
# Extension for CRLs (`man x509v3_config`).
authorityKeyIdentifier=keyid:always

[ ocsp ]
# Extension for OCSP signing certificates (`man ocsp`).
basicConstraints = CA:FALSE
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, digitalSignature
extendedKeyUsage = critical, OCSPSigning
```

其中，subca.cnf 文件内容如下：

```
[ ca ]
# `man ca`
default_ca = CA_default

[ CA_default ]
# Directory and file locations.
dir               = ./
certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/db/index
unique_subject    = no
serial            = $dir/db/serial
RANDFILE          = $dir/private/random

# The root key and root certificate.
private_key       = $dir/subca.key
certificate       = $dir/subca.crt

# For certificate revocation lists.
crlnumber         = $dir/crlnumber
crl               = $dir/crl/ca.crl.pem
crl_extensions    = crl_ext
default_crl_days  = 30

# SHA-1 is deprecated, so use SHA-2 instead.
default_md        = sm3

name_opt          = ca_default
cert_opt          = ca_default
default_days      = 365
preserve          = no
policy            = policy_strict

[ policy_strict ]
# The root CA should only sign intermediate certificates that match.
# See the POLICY FORMAT section of `man ca`.
countryName             = optional
stateOrProvinceName     = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ policy_loose ]
# Allow the intermediate CA to sign a more diverse range of certificates.
# See the POLICY FORMAT section of the `ca` man page.
countryName             = optional
stateOrProvinceName     = optional
localityName            = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ req ]
# Options for the `req` tool (`man req`).
default_bits        = 2048
distinguished_name  = req_distinguished_name
string_mask         = utf8only

# SHA-1 is deprecated, so use SHA-2 instead.
default_md          = sha256

# Extension to add when the -x509 option is used.
x509_extensions     = v3_ca

req_extensions = v3_req

[ req_distinguished_name ]
# See <https://en.wikipedia.org/wiki/Certificate_signing_request>.
countryName                     = optional
stateOrProvinceName             = optional
localityName                    = optional
0.organizationName              = optional
organizationalUnitName          = optional
commonName                      = optional
emailAddress                    = optional

[ v3_req ]

# Extensions to add to a certificate request

basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = localhost

[ v3_ca ]
# Extensions for a typical CA (`man x509v3_config`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ v3_intermediate_ca ]
# Extensions for a typical intermediate CA (`man x509v3_config`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true, pathlen:0
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ crl_ext ]
# Extension for CRLs (`man x509v3_config`).
authorityKeyIdentifier=keyid:always

[ ocsp ]
# Extension for OCSP signing certificates (`man ocsp`).
basicConstraints = CA:FALSE
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, digitalSignature
extendedKeyUsage = critical, OCSPSigning

[ server_sign_req ]
# Extensions to add to a certificate request
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature
# Add more items here, for instance:
# subjectAltName = @alt_names

[ server_enc_req ]
# Extensions to add to a certificate request
basicConstraints = CA:FALSE
keyUsage = keyAgreement, keyEncipherment, dataEncipherment
# Add more items here, for instance:
# subjectAltName = @alt_names

[ client_sign_req ]
# Extensions to add to a certificate request
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature

[ client_enc_req ]
# Extensions to add to a certificate request
basicConstraints = CA:FALSE
keyUsage = keyAgreement, keyEncipherment, dataEncipherment
```

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
openssl s_client -connect 127.0.0.1:4433 -cipher ECC-SM2-WITH-SM4-SM3 -enable_ntls -ntls
```

client 端(测试 ECDHE-SM2-WITH-SM4-SM3 套件)：命令行输入

```sh
openssl s_client -connect 127.0.0.1:4433 -cipher ECDHE-SM2-WITH-SM4-SM3 \
    -sign_cert test_certs/double_cert/CS.cert.pem \
    -sign_key test_certs/double_cert/CS.key.pem \
    -enc_cert test_certs/double_cert/CE.cert.pem \
    -enc_key test_certs/double_cert/CE.key.pem \
    -enable_ntls -ntls
```

## 在你的 client/server 中使用（相关 api 使用）

server 端

```c
int main()
{
    //变量定义
    const SSL_METHOD *meth = NULL;
    SSL_CTX *ctx = NULL;
    const char *sign_key_file = "/path/to/sign_key_file";
    const char *sign_cert_file = "/path/to/sign_cert_file";
    const char *enc_key_file = "/path/to/enc_key_file";
    const char *enc_cert_file = "/path/to/enc_cert_file";

    //双证书相关server的各种定义
    meth = NTLS_server_method();
    //生成上下文
    ctx = SSL_CTX_new(meth);
    //允许使用国密双证书功能
    SSL_CTX_enable_ntls(ctx);

    //加载签名证书，加密证书
    if (sign_key_file) {
        if (!SSL_CTX_use_sign_PrivateKey_file(cctx->ctx, sign_key_file,
                                              SSL_FILETYPE_PEM))
            goto err;
    }

    if (sign_cert_file) {
        if (!SSL_CTX_use_sign_certificate_file(cctx->ctx, sign_cert_file,
                                               SSL_FILETYPE_PEM))
            goto err;
    }

    if (enc_key_file) {
        if (!SSL_CTX_use_enc_PrivateKey_file(cctx->ctx, enc_key_file,
                                             SSL_FILETYPE_PEM))
            goto err;
    }

    if (enc_cert_file) {
        if (!SSL_CTX_use_enc_certificate_file(cctx->ctx, enc_cert_file,
                                              SSL_FILETYPE_PEM))
            goto err;
    }

    //...后续同标准tls流程
    con = SSL_new(ctx);
}
```

client端

```c
int main()
{
    //变量定义
    const SSL_METHOD *meth = NULL;
    SSL_CTX *ctx = NULL;
    const char *sign_key_file = "/path/to/sign_key_file";
    const char *sign_cert_file = "/path/to/sign_cert_file";
    const char *enc_key_file = "/path/to/enc_key_file";
    const char *enc_cert_file = "/path/to/enc_cert_file";

    //双证书相关client的各种定义
    meth = NTLS_client_method();
    //生成上下文
    ctx = SSL_CTX_new(meth);
    //允许使用国密双证书功能
    SSL_CTX_enable_ntls(ctx);

    //设置算法套件为ECC-SM2-WITH-SM4-SM3或者ECDHE-SM2-WITH-SM4-SM3
    //这一步并不强制编写，默认ECC-SM2-WITH-SM4-SM3优先
    if(SSL_CTX_set_cipher_list(ctx, "ECC-SM2-WITH-SM4-SM3") <= 0)
        goto err;

    //加载签名证书，加密证书，仅ECDHE-SM2-WITH-SM4-SM3套件需要这一步,
    //该部分流程用...begin...和...end...注明
    // ...begin...
    if (sign_key_file) {
        if (!SSL_CTX_use_sign_PrivateKey_file(cctx->ctx, sign_key_file,
                                              SSL_FILETYPE_PEM))
            goto err;
    }

    if (sign_cert_file) {
        if (!SSL_CTX_use_sign_certificate_file(cctx->ctx, sign_cert_file,
                                               SSL_FILETYPE_PEM))
            goto err;
    }

    if (enc_key_file) {
        if (!SSL_CTX_use_enc_PrivateKey_file(cctx->ctx, enc_key_file,
                                             SSL_FILETYPE_PEM))
            goto err;
    }

    if (enc_cert_file) {
        if (!SSL_CTX_use_enc_certificate_file(cctx->ctx, enc_cert_file,
                                              SSL_FILETYPE_PEM))
            goto err;
    }
    // ...end...

    //...后续同标准tls流程
    con = SSL_new(ctx);
}
```

## 说明

由于国密双证书的握手流程和协议版本号与标准tls流程存在一定的不同，因此我们选择将双证书的实现(代码里命名为 ntls )同现有的 tls 状态机拆分开来，然后在入口处通过对请求的版本号进行识别，然后使其进入正确的状态机。

{{#template ../template/footer.md}}
