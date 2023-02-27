# 委托凭证（Delegated Credentials）

## 前言

在HTTPS通信的场景下，通常在Web服务器上部署证书和密钥，用于身份认证，同时保证消息的完整性。而加密通信的安全性主要取决于加密算法的安全性和密钥的安全性。如果使用了不安全的加密算法，即使攻击者不知道密钥，在只知道密文的前提下，依然可以通过对密文进行分析来实现攻击。例如2020年3月底，研究者公开了Zoom的重大安全漏洞，其中包括，Zoom使用了不安全的加密模式（ECB加密模式），导致了多个知名的公司、教育机构和政府组织宣布，出于信息安全的考虑，禁止使用Zoom，转而使用其他替代产品。

同时，随着近些年量子计算的快速发展，对现代密码学的安全性提出了新的挑战。目前广泛使用的公钥加密算法以RSA和基于椭圆曲线的密码算法为主，而这些算法所基于的数学难题对于经典计算机来说很难，但是对于一个功能足够强大的量子计算机来说可以轻松解决。后量子密码学（PQC）标准化组织致力于研究新的加密算法，以抵抗量子计算，在不久的未来，当新的算法来临时，我们该如何快速迁移？

在加密算法安全的前提下，如何保证密钥的安全性，就成为了关键。尤其是对于部署大量边缘服务器的企业或云计算厂商，将密钥部署在每一台边缘服务器上，如果有一台服务器被黑客入侵，就会导致服务所有站点的密钥泄露，影响范围大，后果非常严重。

## Delegated Credentials介绍

Delegated Credentials 的意思就是委托凭证，使用服务端（或客户端）证书签发DC，意味着就可以使用DC来代理服务端证书进行TLS握手。而DC本质上就是“迷你证书”，DC公钥的功能对等于证书，DC私钥对等于证书的私钥。

### 签发DC

X.509标准禁止终端实体证书作为签发者再去签发其他证书，但是只要证书的KeyUsage扩展中包含digitalSignature，就可以用来签发其他签名的对象，而这里对应的就是DC。

X.509标准定义了新的证书扩展，DelegationUsage，表明服务端证书包含这个扩展，才具备签发DC的能力。同时需要终端实体证书的KeyUsage扩展中包含digitalSignature。

### 服务端认证

1. ClientHello消息中包含delegated_credential扩展，表明支持DC；
2. 服务端看到delegated_credential扩展后，如果也支持DC，则在Certificate消息中，发送证书链的同时，在终端实体证书的扩展中携带delegated credential；
3. 客户端收到Certificate消息后，校验delegated credential，包括签名、有效时间等；
4. 服务端使用dc的私钥进行签名，然后发送CertificateVerify消息；
5. 客户端收到CertificateVerify消息后，使用dc公钥来验证签名，完成对服务端的身份认证，同时保证消息的完整性。

## 对比 keyless 和 Delegated Credentials 方案

keyless方案，通过将密钥部署在远程的Back-End（例如KeyServer），在TLS握手的时候，请求Back-End使用密钥进行签名。通过将密钥和边缘服务器隔离，提升密钥的安全性，降低密钥泄露风险。

```
Client            Front-End            Back-End
     |----ClientHello--->|                    |
     |<---ServerHello----|                    |
     |<---Certificate----|                    |
     |                   |<---remote sign---->|
     |<---CertVerify-----|                    |
     |        ...        |                    |
```

而DC方案，同样在Back-End上部署证书和密钥，提前签发DC后并部署在边缘服务器上（Front-End）。

和keyless相比的优点：

* keyless需要改造TLS握手处理过程，修改密码库（例如OpenSSL）支持keyless，而DC不需要
* keyless在TLS握手时请求Back-End服务器，增加了握手的延时，而DC签发、部署可以异步离线执行

```
Client            Front-End            Back-End
     |                   |<--DC distribution->|
     |----ClientHello--->|                    |
     |<---ServerHello----|                    |
     |<---Certificate----|                    |
     |<---CertVerify-----|                    |
     |        ...        |                    |
```

## Delegated Credentials优势

### 自主签发

通常，服务端证书或客户端证书是由证书认证机构（CA）来签发，一般有效期不超过1年，在证书过期之前需要找CA重新签发证书。如果想要签发短期的证书，意味着要与CA机构进行频繁的交互，等待CA审核通过后，下载证书重新部署。而短期的证书意味着证书的有效期更短，如果CA相应不及时，可能导致证书过期的风险。

引入Delegated Credentials后，就可以使用服务端（或客户端）证书来签发DC，而不需要依赖于外部CA。通过DC来代理终端实体证书，完成TLS通信。

### 使用更安全的算法

正是因为自主签发，我们可以选择更安全的签名算法，例如Ed25519，而CA通常不支持。

每次签发DC时，都可以选择使用最安全的签名算法，可以及时淘汰老旧的、不安全的算法，保证通信的安全性。

### 周期短

DC的有效期最长不能超过7天，更短的周期意味着频繁的轮转，降低密钥泄露的风险，提升安全性。

注意，因为周期短，DC不支持吊销。

## Delegated Credentials支持现状

DC既需要服务端支持，也需要客户端支持，目前支持DC的浏览器并不多，以Firefox为主。

支持DC的开源密码库包括Google的BoringSSL、Firefox的NSS、蚂蚁的BabaSSL，目前OpenSSL还不支持DC。

BabaSSL发布的8.2.0版本支持Delegated Credentials，且已经合并到开源BabaSSL，欢迎大家使用内部RPM包或开源版本。

## 实战Delegated Credentials

### 签发DC

注意：需要使用BabaSSL才能签发dc，开源OpenSSL并不支持。

完整的脚本请参考BabaSSL开源代码库，<https://github.com/BabaSSL/BabaSSL/blob/master/test/recipes/80-test_dc_sign_data/sign_dc.sh>

```sh
# 创建dc密钥
openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:P-256 -out dc-ecc-server-longterm.key

# 签发dc
openssl delecred -new -server -sec 604800 -dc_key dc-ecc-server-longterm.key -out dc-ecc-server-longterm.dc -parent_cert dc-ecc-leaf.crt -parent_key dc-ecc-leaf.key -expect_verify_md sha256 -sha256
```

### 使用DC通信

认证服务端：

```sh
# server端
openssl s_server -accept 127.0.0.1:4433 -cert dc-ecc-leaf.crt -dc_pkey dc-ecc-server-longterm.key -dc dc-ecc-server-longterm.dc -enable_sign_by_dc

# client端
openssl s_client -connect 127.0.0.1:4433 -enable_verify_peer_by_dc -verifyCAfile dc-ecc-chain-ca.crt -verify_return_error
```

双向认证：

```sh
# server端
openssl s_server -accept 127.0.0.1:4433 -cert dc-ecc-leaf.crt -dc_pkey dc-ecc-server-longterm.key -dc dc-ecc-server-longterm.dc -enable_sign_by_dc -enable_verify_peer_by_dc -Verify 1 -verifyCAfile dc-ecc-chain-ca.crt -verify_return_error

# client端
openssl s_client -connect 127.0.0.1:4433 -cert dc-ecc-leaf-clientUse.crt -dc_pkey dc-ecc-client-longterm.key -dc dc-ecc-client-longterm.dc -enable_verify_peer_by_dc -enable_sign_by_dc -verifyCAfile dc-ecc-chain-ca.crt -verify_return_error
```

### 应用基于BabaSSL集成DC

```c
SSL *s;
SSL_CTX *ctx;
DELEGATED_CREDENTIAL *dc = NULL;
ctx = SSL_CTX_new(TLS_server_method());
if (ctx == NULL) {
    // error
}

// 设置证书
if (!SSL_CTX_use_certificate_file(ctx, cert_file, SSL_FILETYPE_PEM)) {
    // error
}
// 设置证书的密钥
if (!SSL_CTX_use_PrivateKey_file(ctx, key_file, SSL_FILETYPE_PEM)) {
    // error
}

// 加载DC文件，注意：必须先加载服务端（或客户端）证书，再加载DC
if (!SSL_CTX_use_dc_file(ctx, cert_file, 0)) {
    // error
}

// 加载DC的密钥
if (!SSL_CTX_use_dc_PrivateKey_file(ctx, key_file, SSL_FILETYPE_PEM)) {
    // error
}

//功能：开启dc签名功能，server在开启该功能并收到dc请求时才会选择使用dc进行签名
SSL_CTX_enable_sign_by_dc(ctx);
...
    
s = SSL_new(ctx);

...
```

{{#template ../template/footer.md}}
