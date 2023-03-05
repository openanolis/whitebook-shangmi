# Web 服务国密双证书支持

## GMSSL

GmSSL 是一个开放源代码加密工具包，它提供对GM / T 串行标准中指定的中国国家加密算法和协议的第一级支持。作为OpenSSL项目的一个分支，GmSSL提供与OpenSSL的API级别兼容性并维护所有功能。在此感谢贡献，该项目地址：<https://github.com/guanzhi/GmSSL>

1. 代码下载及配置变量

```shell
# 通过git下载
$ git clone https://github.com/guanzhi/GmSSL gmssl

# 如果嫌下载速度慢，也可下载代码压缩包
$ wget https://github.com/guanzhi/GmSSL/archive/master.zip$ unzip master.zip
```

GmSSL工具箱相关项目文档

2. 编译与安装

```shell
$ cd 你解压的目录
$ ./config --prefix=/usr/local/gmssl
$ make
$ make install
$ cd /usr/local/gmssl

# tree:显示目录树 -d:只显示目录 -L:选择显示的目录深度 /usr/local/gmssl目录树如下：
$ tree -L 2
.
|-- bin
|   |-- c_rehash
|   |-- gmssl
|   `-- openssl -> gmssl
|-- include
|   `-- openssl
|-- lib
|   |-- engines-1.1
|   |-- libcrypto.a
|   |-- libcrypto.so -> libcrypto.so.1.1
|   |-- libcrypto.so.1.1
|   |-- libssl.a
|   |-- libssl.so -> libssl.so.1.1
|   |-- libssl.so.1.1
|   `-- pkgconfig
|-- share
|   |-- doc
|   `-- man
`-- ssl
    |-- certs
    |-- misc
    |-- openssl.cnf
    |-- openssl.cnf.dist
    `-- private
```

config貌似动态编译、静态编译也会有一定的影响，（no-shared 不去编译动态库，编译出来的 gmssl不再依赖 libssl.so ,避免因缺少动态库出错）

openssl.cnf位于`/usr/local/gmssl/ssl`下，可以按需修改生成各种文件位置、信息等等，需要注意的是生成证书、私钥的位置、名称要与openssl中保持一致。

3. 配置环境变量及检测

  * 如果不配置，就会出现找不到gmssl命令的情况

```shell
# 配置安装路径 在已有的PATH等变量前面添加路径‘：’号为分隔符，这种只是临时生效，关闭shell即失效
PATH=/usr/local/gmssl/bin:$PATH
LD_LIBRARY_PATH=/usr/local/gmssl/lib:$LD_LIBRARY_PATH
```

  * 在/etc/profile文件中添加变量对所有用户生效（永久的）

```shell
vim /etc/profile

#编辑
export PATH=/usr/local/gmssl/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/gmssl/lib:$LD_LIBRARY_PATH
source /etc/profile
```

4. 初始化目录

openssl.cnf的部分配置，详细配置可以参考

![Nginx TLCP](images/nginx_tlcp.png)
 
OpenSSL主配置文件

```shell
# 创建根目录
mkdir -p /home/gmssl/demoCA
cd /home/gmssl/demoCA
mkdir certs crl newcerts private

# 在此路径下要创建好/usr/local/gmssl/ssl/openssl.cnf中需要的
# certs， crl ，new_certs_dir和private_key的子目录
# 默认是newcerts和private
touch index.txtecho '01' > serial
```

5. 生成国密证书

  * 生成根证书

```shell
# 生成私钥key
gmssl ecparam -genkey -name sm2p256v1 -text \
    -out CA.key -config /usr/local/gmssl/ssl/openssl.cnf

# 生成证书签名请求
gmssl req -new -key CA.key -out Root.req

# 生成根证书
gmssl x509 -req -days 3650 -sm3 -in Root.req \
    -signkey Root.key -out RootCA.crt
```

  * 生成服务端证书

```shell
# 生成私钥
> gmssl ecparam -genkey -name sm2p256v1 -text -out Server.key

#证书请求
> gmssl req -new -key Server.key -out Server.csr \
    -subj /C=CN/ST=Beijing/L=Beijing/O=CSG/OU=WangAn (GM)/CN=*.vpn.test.cn \
    -config /usr/local/gmssl/ssl/openssl.cnf

#签发证书
> gmssl x509 -req -sm3 -days 3650 -CA RootCA.crt \
    -CAkey Root.key -CAcreateserial \
    -in Server.req -out ServerCA.crt

# 证书验证
> gmssl verify -CAfile RootCA.crt ServerCA.crt
ServerCA.crt: OK

> tree
.
|-- certs
|-- crl
|-- index.txt
|-- newcerts
|-- private
|   `-- Root.key
|-- RootCA.crt
|-- RootCA.srl
|-- Root.key
|-- Root.req
|-- serial
|-- ServerCA.crt
|-- Server.csr
`-- Server.key
```

  * 生成客户端证书

```sh
# 生成私钥
gmssl ecparam -genkey -name sm2p256v1 -text \
    -out Client.key -config /usr/local/gmssl/ssl/openssl.cnf

# 生成客户证书请求
gmssl req -new -key Client.key -out Client.req \
    -subj /C=CN/ST=Beijing/L=Beijing/O=CSG/OU=WangAn (GM)/CN=*.vpn.test.cn \
    -config /usr/local/gmssl/ssl/openssl.cnf

# 签发证书
gmssl x509 -req -sm3 -days 3650 -CA RootCA.crt \
    -CAkey demoCA/private/Root.key -CAcreateserial \
    -in Client.req -out Client.crt

# 证书验证
gmssl verify -CAfile RootCA.crt Client.crt 
```

6. 生成脚本

```sh
#!/bin/sh

# gen GM certificate files
CurPath=`dirname $(readlink -f $0)`
 
GmsslRootPath=/usr/local/gmssl
LdPath=/usr/local/gmssl/lib
GmsslBin=gmssl
DemoCaDir=${GmsslRootPath}/apps/demoCA/
CertDir=${DemoCaDir}/certs/
KeyDir=${CertDir}/keys/
CrlDir=${DemoCaDir}/crl/
ReqDir=${DemoCaDir}/reqs/
 
CertDays=1500
 
CACertFile=CA.cert.pem
CAKeyFile=CA.key.pem
CA_DN_STRING=" /C=CN/ST=Beijing/L=Beijing/O=CSG/OU=WangAn (GM)/CN=*.vpn.test.cn"
CAReqFile=CA.req
 
SSCertFile=SS.cert.pem
SSKeyFile=SS.key.pem
SS_DN_STRING=" /C=CN/ST=Beijing/L=Beijing/O=CSG/OU=WangAn (GM)/CN=*.vpn.test.cn"
SSReqFile=SS.req
 
SECertFile=SE.cert.pem
SEKeyFile=SE.key.pem
SE_DN_STRING=" /C=CN/ST=Beijing/L=Beijing/O=CSG/OU=WangAn (GM)/CN=*.vpn.test.cn"
SEReqFile=SE.req
 
CSCertFile=CS.cert.pem
CSKeyFile=CS.key.pem
CS_DN_STRING=" /C=CN/ST=Beijing/L=Beijing/O=CSG/OU=vpn.test.cn/CN=WangAn (GM)"
CSReqFile=CS.req
 
CECertFile=CE.cert.pem
CEKeyFile=CE.key.pem
CE_DN_STRING=" /C=CN/ST=Beijing/L=Beijing/O=CSG/OU=WangAn (GM)/CN=*.vpn.test.cn"
CEReqFile=CE.req

if [ ! -d "${GmsslRootPath}" ]; then
    echo "GmSSL path DONOT exist!"
    exit 2
fi

export LD_LIBRARY_PATH=${LdPath}

rm -rf "${GmsslRootPath}/apps/demoCA/"

mkdir -p "${CertDir}"
mkdir -p "${KeyDir}"
mkdir -p "${CrlDir}"
mkdir -p "${ReqDir}"

echo "#### Generate CA certificate file..."

${GmsslBin} ecparam -name sm2p256v1 -out "${DemoCaDir}/SM2.pem"

${GmsslBin} req -config "${GmsslRootPath}/apps/openssl.cnf" \
    -nodes -subj "${CA_DN_STRING}" -keyout "${KeyDir}/${CAKeyFile}" \
    -newkey "ec:${DemoCaDir}/SM2.pem" -new -out "${ReqDir}/${CAReqFile}"

# Sign CA certificate with CAKeyFile

${GmsslBin} x509 -sm3 -req -days ${CertDays} -in "${ReqDir}/${CAReqFile}" \
    -extfile "${GmsslRootPath}/apps/openssl.cnf" -extensions v3_ca \
    -signkey "${KeyDir}/${CAKeyFile}" \
    -CAcreateserial -out "${CertDir}/${CACertFile}"

# Print CA certificate file

${GmsslBin} x509 -in "${CertDir}/${CACertFile}" -noout -text

echo "#### Generate Server sign certificate file..."

${GmsslBin} req -config "${GmsslRootPath}/apps/openssl.cnf" \
    -nodes -subj "${SS_DN_STRING}" -keyout "${KeyDir}/${SSKeyFile}" \
    -newkey "ec:${DemoCaDir}/SM2.pem" -new -out "${ReqDir}/${SSReqFile}"

# Sign SS certificate with CAKeyFile

${GmsslBin} x509 -sm3 -req -days ${CertDays} \
    -in "${ReqDir}/${SSReqFile}" -CA "${CertDir}/${CACertFile}" \
    -CAkey "${KeyDir}/${CAKeyFile}" \
    -extfile "${GmsslRootPath}/apps/openssl.cnf" -extensions v3_req \
    -CAcreateserial -out "${CertDir}/${SSCertFile}"

# Print SS certificate file

${GmsslBin} x509 -in "${CertDir}/${SSCertFile}" -noout -text

echo "#### Generate Server encrypt certificate file..."

${GmsslBin} req -config "${GmsslRootPath}/apps/openssl.cnf" \
    -nodes -subj "${SE_DN_STRING}" \
    -keyout "${KeyDir}/${SEKeyFile}" -newkey "ec:${DemoCaDir}/SM2.pem" \
    -new -out "${ReqDir}/${SEReqFile}"

# Sign SE certificate with CAKeyFile

${GmsslBin} x509 -sm3 -req -days ${CertDays} \
    -in "${ReqDir}/${SEReqFile}" \
    -CA "${CertDir}/${CACertFile}" -CAkey "${KeyDir}/${CAKeyFile}" \
    -extfile "${GmsslRootPath}/apps/openssl.cnf" -extensions v3_req \
    -CAcreateserial -out "${CertDir}/${SECertFile}"

# Print SE certificate file

${GmsslBin} x509 -in "${CertDir}/${SECertFile}" -noout -text

echo "#### Generate Client sign certificate file..."

${GmsslBin} req -config "${GmsslRootPath}/apps/openssl.cnf" \
    -nodes -subj "${CS_DN_STRING}" \
    -keyout "${KeyDir}/${CSKeyFile}" -newkey "ec:${DemoCaDir}/SM2.pem" \
    -new -out "${ReqDir}/${CSReqFile}"

# Sign CS certificate with CAKeyFile

${GmsslBin} x509 -sm3 -req -days ${CertDays} \
    -in "${ReqDir}/${CSReqFile}" \
    -CA "${CertDir}/${CACertFile}" -CAkey "${KeyDir}/${CAKeyFile}" \
    -extfile "${GmsslRootPath}/apps/openssl.cnf" -extensions v3_req \
    -CAcreateserial -out "${CertDir}/${CSCertFile}"

# Print CS certificate file

${GmsslBin} x509 -in "${CertDir}/${CSCertFile}" -noout -text

echo "#### Generate Client encrypt certificate file..."

${GmsslBin} req -config "${GmsslRootPath}/apps/openssl.cnf" \
    -nodes -subj "${CE_DN_STRING}" \
    -keyout "${KeyDir}/${CEKeyFile}" -newkey "ec:${DemoCaDir}/SM2.pem" \
    -new -out "${ReqDir}/${CEReqFile}"

# Sign CE certificate with CAKeyFile

${GmsslBin} x509 -sm3 -req -days ${CertDays} \
    -in "${ReqDir}/${CEReqFile}" \
    -CA "${CertDir}/${CACertFile}" -CAkey "${KeyDir}/${CAKeyFile}" \
    -extfile "${GmsslRootPath}/apps/openssl.cnf" -extensions v3_req \
    -CAcreateserial -out "${CertDir}/${CECertFile}"

# Print CE certificate file

${GmsslBin} x509 -in "${CertDir}/${CECertFile}" -noout -text
```

每次demoCA目录结构都会删除重新创建，下面是/usr/local/gmssl/apps下的目录结构：

```shell
$ tree -L 4
.
|-- demoCA
|   |-- certs
|   |   |-- CA.cert.pem
|   |   |-- CA.srl
|   |   |-- CE.cert.pem
|   |   |-- CS.cert.pem
|   |   |-- keys
|   |   |   |-- CA.key.pem
|   |   |   |-- CE.key.pem
|   |   |   |-- CS.key.pem
|   |   |   |-- SE.key.pem
|   |   |   `-- SS.key.pem
|   |   |-- SE.cert.pem
|   |   `-- SS.cert.pem
|   |-- crl
|   |-- reqs
|   |   |-- CA.req
|   |   |-- CE.req
|   |   |-- CS.req
|   |   |-- SE.req
|   |   `-- SS.req
|   `-- SM2.pem
`-- openssl.cnf
```

7. 生成证书注销列表

脚本代码：

```shell
#certificate crl file
CurPath=`dirname $(readlink -f $0)`
 
GmsslRootPath=/usr/local/gmssl
GmsslBin=gmssl
DemoCaDir=${GmsslRootPath}/apps/demoCA
CertDir=${DemoCaDir}/certs
KeyDir=${CertDir}/keys
CrlDir=${DemoCaDir}/crl
ReqDir=${DemoCaDir}/reqs/

if [ -z "$1" ]; then
    echo "Usage: "`basename "$0"`" cert0 [cert1 cert2 ...]"
fi

touch "${DemoCaDir}/index.txt"
touch "${DemoCaDir}/index.txt.attr"

if [ ! -e "${DemoCaDir}/crlnumber" ]; then
    echo 01 > "${DemoCaDir}/crlnumber"
fi

cd "${DemoCaDir}/.."
 
until [ $# -eq 0 ] do
	${GmsslBin} ca -revoke "${CertDir}/$1" \
        -keyfile "${KeyDir}/CA.key.pem" \
        -cert "${CertDir}/CA.cert.pem"

	shift
done

${GmsslBin} ca -gencrl -keyfile "${KeyDir}/CA.key.pem" \
    -cert "${CertDir}/CA.cert.pem" -out "${CrlDir}/gm_cert.crl"
```

生成之后/usr/local/gmssl/apps的目录：

```shell
$ tree
.
|-- demoCA
|   |-- certs
|   |   |-- CA.cert.pem
|   |   |-- CA.srl
|   |   |-- CE.cert.pem
|   |   |-- CS.cert.pem
|   |   |-- keys
|   |   |   |-- CA.key.pem
|   |   |   |-- CE.key.pem
|   |   |   |-- CS.key.pem
|   |   |   |-- SE.key.pem
|   |   |   `-- SS.key.pem
|   |   |-- SE.cert.pem
|   |   `-- SS.cert.pem
|   |-- crl
|   |   `-- gm_cert.crl
|   |-- crlnumber
|   |-- crlnumber.old
|   |-- index.txt
|   |-- index.txt.attr
|   |-- reqs
|   |   |-- CA.req
|   |   |-- CE.req
|   |   |-- CS.req
|   |   |-- SE.req
|   |   `-- SS.req
|   `-- SM2.pem
`-- openssl.cnf
```

## 配置改造

国密OpenSSL与国密Nginx

gmssl_openssl_1.1_bxx.tar.gz

无缝nginx国密改造，支持nginx1.6+

编译部署（以nginx-1.8.0为例）

1) 下载gmssl_openssl_1.1_bxx.tar.gz到/root/下
2) 解压 tar xzfm gmssl_openssl_1.1_bxx.tar.gz -C /usr/local
3) 下载nginx-1.18.0.tar.gz到/root/下
4) 解压 tar xzfm nginx-1.18.0.tar.gz
5) 进入目录 cd /root/nginx-1.18.0
6) 编辑auto/lib/openssl/conf，将全部$OPENSSL/.openssl/修改为$OPENSSL/并保存
7) 编译配置

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
```

8) 编译安装
make install
9) /usr/local/nginx即为生成的nginx目录
注：可能需要使用yum install pcre-devel需要安装pcre-devel

配置示例(国密单向)

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

配置示例(国密双向)

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

配置示例(国密/RSA单向自适应)

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
