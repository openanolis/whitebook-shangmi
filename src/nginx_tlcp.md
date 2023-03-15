# Web 服务国密双证书支持

## GMSSL

GmSSL 是一个开放源代码加密工具包，它提供对GM / T 串行标准中指定的中国国家加密算法和协议的第一级支持。作为OpenSSL项目的一个分支，GmSSL提供与OpenSSL的API级别兼容性并维护所有功能。在此感谢贡献，该项目地址：<https://github.com/guanzhi/GmSSL>

## 准备环境

下载 GmSSL 的代码，参考官网方法编译以及配置 PATH 变量：

```shell
# 下载代码
git clone https://github.com/guanzhi/GmSSL

# 编译安装
cd GmSSL
./config --prefix=/usr/local/gmssl
make
make install

# 配置环境变量，使shell能找到的gmssl文件
export PATH=/usr/local/gmssl/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/gmssl/lib:$LD_LIBRARY_PATH
```

`openssl.cnf` 位于 `/usr/local/gmssl/ssl` 下，可以按需修改生成各种文件位置、信息等等，需要注意的是生成证书、私钥的位置、名称要与openssl中保持一致。

openssl.cnf 的部分配置，详细配置可以参考：

```
[ CA_default ]
name        = CA
dir         = /home/gmssl/demoCA    # Where everything is kept
certs       = $dir/certs        # Where the issued certs are kept
crl_dir     = $dir/crl          # Where the issued crl are kept
database    = $sdir/index.txt   # database index file.
#unique_subject = no            # Set to "no' to allow creation of
                                # several certs with same subject.

new_certs_dir = Sdir/newcerts   # default place for new certs.

certificate = $dir/$name.pem    # The CA certificate
serial      = $dir/serial       # The current serial number
crlnumber   = $dir/crlnumber    # the current crl number
                                # must be commented out to leave a V1 CRL
crl         = $dir/crl.pem      # The current CRL
private_key = $dir/private/$name.pem    # The private key
RANDFILE    = $dir/private/.rand        # private random number file

x509_extensions = usr_cert      # The extensions to add to the cert
```

准备主配置文件：

```shell
# 创建根目录
mkdir -p /home/gmssl/demoCA
cd /home/gmssl/demoCA
mkdir certs crl newcerts private

# 在此路径下要创建好/usr/local/gmssl/ssl/openssl.cnf中需要的
# certs， crl ，new_certs_dir和private_key的子目录
# 默认是newcerts和private

touch index.txt

echo '01' > serial
```

## 生成国密证书

TLCP 是双证书协议，因此需要生成根证书，服务端和客户端要分别生成签名证书和加密证书，可以参考如下命令来完成证书生成。

生成根证书：

```sh
gmssl ecparam -genkey -name sm2p256v1 -text \
    -out CA.key -config /usr/local/gmssl/ssl/openssl.cnf

gmssl req -new -key CA.key -out Root.req

gmssl x509 -req -days 3650 -sm3 -in Root.req \
    -signkey Root.key -out RootCA.crt
```

生成服务端证书：

```sh
gmssl ecparam -genkey -name sm2p256v1 -text -out Server.key

gmssl req -new -key Server.key -out Server.csr \
    -subj /C=CN/ST=Beijing/L=Beijing/O=CSG/OU=WangAn (GM)/CN=*.vpn.test.cn \
    -config /usr/local/gmssl/ssl/openssl.cnf

gmssl x509 -req -sm3 -days 3650 -CA RootCA.crt -CAkey Root.key \
    -CAcreateserial -in Server.req -out ServerCA.crt
```

生成客户端证书：

```sh
gmssl ecparam -genkey -name sm2p256v1 -text \
    -out Client.key -config /usr/local/gmssl/ssl/openssl.cnf

gmssl req -new -key Client.key -out Client.req \
    -subj /C=CN/ST=Beijing/L=Beijing/O=CSG/OU=WangAn (GM)/CN=*.vpn.test.cn \
    -config /usr/local/gmssl/ssl/openssl.cnf

gmssl x509 -req -sm3 -days 3650 -CA RootCA.crt \
    -CAkey demoCA/private/Root.key \
    -CAcreateserial -in Client.req -out Client.crt
```

## 配置 Nginx 支持双证书协议

无缝 nginx 国密改造，支持 nginx 1.6+ 版本，本例以 nginx 1.8.0 为例说明。

1. 下载 nginx 代码
2. 编辑`auto/lib/openssl/conf`，将全部`$OPENSSL/.openssl/`修改为`$OPENSSL/`并保存
3. 按如下指令编译安装

    ```sh
    ./configure \
        --without-http_gzip_module \
        --with-http_ssl_module \
        --with-http_stub_status_module \
        --with-http_v2_module \
        --with-file-aio \
        --with-openssl="/usr/local/gmssl" \
        --with-cc-opt="-I/usr/local/gmssl/include" \
        --with-ld-opt="-lm"

    make
    make install
    ```

    注：可能需要使用 `yum install pcre-devel` 安装 pcre-devel 依赖包。

安装成功后，根据情况选择以下的配置后，即可启动支持国密双证书的 Web 服务。

配置示例 (国密单向)

```nginx
server {
    listen 0.0.0.0:443 ssl;

    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:AES128-SHA:DES-CBC3-SHA:ECC-SM4-CBC-SM3:ECC-SM4-GCM-SM3;
    ssl_verify_client off;

    ssl_certificate /usr/local/nginx/conf/demo1.sm2.sig.crt.pem;
    ssl_certificate_key /usr/local/nginx/conf/demo1.sm2.sig.key.pem;

    ssl_certificate /usr/local/nginx/conf/demo1.sm2.enc.crt.pem;
    ssl_certificate_key /usr/local/nginx/conf/demo1.sm2.enc.key.pem;

    location / {
        root html;
        index index.html index.htm;
    }
}
```

配置示例 (国密双向)

```nginx
server {
    listen 0.0.0.0:443 ssl;

    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:AES128-SHA:DES-CBC3-SHA:ECC-SM4-CBC-SM3:ECC-SM4-GCM-SM3;
    ssl_client_certificate /usr/local/nginx/conf/demo1.sm2.trust;
    ssl_verify_client on;

    ssl_certificate /usr/local/nginx/conf/demo1.sm2.sig.crt.pem;
    ssl_certificate_key /usr/local/nginx/conf/demo1.sm2.sig.key.pem;

    ssl_certificate /usr/local/nginx/conf/demo1.sm2.enc.crt.pem;
    ssl_certificate_key /usr/local/nginx/conf/demo1.sm2.enc.key.pem;

    location / {
        root html;
        index index.html index.htm;
    }
}
```

配置示例 (国密 / RSA 单向自适应)

```nginx
server {
    listen 0.0.0.0:443 ssl;

    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:AES128-SHA:DES-CBC3-SHA:ECC-SM4-CBC-SM3:ECC-SM4-GCM-SM3;
    ssl_verify_client off;

    ssl_certificate /usr/local/nginx/conf/demo1.rsa.crt.pem;
    ssl_certificate_key /usr/local/nginx/conf/demo1.rsa.key.pem;
    ssl_certificate /usr/local/nginx/conf/demo1.sm2.sig.crt.pem;
    ssl_certificate_key /usr/local/nginx/conf/demo1.sm2.sig.key.pem;
    ssl_certificate /usr/local/nginx/conf/demo1.sm2.enc.crt.pem;
    ssl_certificate_key /usr/local/nginx/conf/demo1.sm2.enc.key.pem;

    location / {
        root html;
        index index.html index.htm;
    }
}
```

{{#template template/footer.md}}
