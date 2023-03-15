# Python国密签名实践

## cryptography 和 pyopenssl

[cryptography](https://github.com/pyca/cryptography) 是使用 Python 开发的 OpenSSL wrapper，依赖于 OpenSSL C 库来进行所有密码操作。

[pyopenssl](https://github.com/pyca/pyopenssl) 是另一个 OpenSSL wrapper，它提供更高层次的抽象，主要用于操作 SSL/TLS 协议，其它算法部分主要通过调用 cryptography 库来实现功能。

## python 商密签名实践

龙蜥社区不仅在OpenSSL社区参与SM2/3/4等国密算法的优化，而且也在cryptography和pyopenssl项目支持OpenSSL的国密签名来保障python的用户能够正常使用国密。接下来分别以 OpenSSL 3.x 和 OpenSSL 1.1.x 两个支持国密版本的OpenSSL为例阐述python的商密签名实践。

### 基于 OpenSSL 3.x 的 python 的商密签名

正常安装 OpenSSL 3.x 和 pyopenssl 以及 cryptography 后（无需修改）后，生成SM2-with-SM3所需要的商密密钥与证书

```sh
openssl ecparam -genkey -name SM2 -out private.pem

openssl req -new -x509 -days 36500 \
    -key private.pem -out cert.crt -sm3 \
    -subj "/C=CN/ST=Zhejiang/L=Hangzhou/O=Alibaba/OU=OS/CN=CA/emailAddress=ca@foo.com"
```

开发商密签名测试程序test_shangmi.py（以python 3为例），注意上文的 pyopenssl 与 cryptography 编译安装时与测试程序的python大版本号（同是python 2或者同是python 3）保持一致, 该测试程序功能主要是：

- 加载之前生成好的SM2-with-SM3的私钥和证书
- 调用pyopenssl的sign和verify函数进行SM2-with-SM3的商密签名和验签

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import sys
import base64

from OpenSSL.crypto import load_certificate, load_privatekey
from OpenSSL.crypto import FILETYPE_PEM
from OpenSSL.crypto import (
    sign,
    verify,
)

if __name__ == "__main__":
    if len(sys.argv) != 3:
        print("Usage: ./test_shangmi.py [private_key_file] [cert_file]")
        exit()

    private_key_file = open(sys.argv[1], 'rb')
    if not private_key_file:
        print("fail to open private_key_file")
        exit()
    private_key = private_key_file.read()

    cert_file = open(sys.argv[2], 'rb')
    if not cert_file:
        print("fail to open cert_file")
        exit()
    cert = cert_file.read()

    priv_key = load_privatekey(FILETYPE_PEM, private_key)
    cert = load_certificate(FILETYPE_PEM, cert)
    content = (
        b"It was a bright cold day in April, and the clocks were striking "
        b"thirteen. Winston Smith, his chin nuzzled into his breast in an "
        b"effort to escape the vile wind, slipped quickly through the "
        b"glass doors of Victory Mansions, though not quickly enough to "
        b"prevent a swirl of gritty dust from entering along with him."
    )
    sig = sign(priv_key, content, "sm3")
    print("sig", sig)
    print(base64.b64encode(sig))
    print("len(sig)", len(sig))
    verify(cert, sig, content, "sm3") 
```

执行测试程序即可 (无报错则验签成功，详见pyopenssl对应的验签函数逻辑)

```sh
./test_shangmi.py private.pem cert.crt
```

### 基于 OpenSSL 1.1.x 的 python 的商密签名

#### OpenSSL 1.1.x 的选择

这里选择使用 [**Anolis OS 国密开发指南**](anolisos_guide.html) 章节描述的OpenSSL 1.1.1版本，当然也可以选择使用 [Tongsuo](https://github.com/Tongsuo-Project/Tongsuo) 作为底层密码库。

除了对国密的支持外，在[OpenSSL社区官方文档](https://github.com/openssl/openssl/blob/OpenSSL_1_1_1/doc/man3/EVP_PKEY_set1_RSA.pod)中指出, 在检测到使用SM2密钥时，需要调用 `EVP_PKEY_set_alias_type(pkey, EVP_PKEY_SM2)` 函数显式使用 SM2 算法。所以在基于OpenSSL 1.1.x 的 python 库中，也需要做对应的支持，因此需要对 cryptography 和 pyopenssl 库进行改造，导出 `EVP_PKEY_set_alias_type` 函数以及添加对应的逻辑（原有的 cryptography 和 pyopenssl 库并不支持这一能力）。

#### cryptography

龙蜥社区经过一些开发和测试在 cryptography 中提交并合入了导出 `EVP_PKEY_set_alias_type` 函数的[代码](https://github.com/pyca/cryptography/pull/7935)，以便用户使用SM2算法，因此用户在使用 cryptography 仓库时无需修改。

#### pyopenssl 的修改

pyopenssl 也需要一些修改，龙蜥社区已经在 pyopenssl 社区提交了[PR](https://github.com/pyca/pyopenssl/pull/1172)。在代码合入之前仍需要用户手动打入patch，可参考如下步骤(以python 3为例)进行安装。

```sh
git clone https://github.com/hustliyilin/pyopenssl.git
python3 setup.py sdist
cd dist
pip3 install pyOpenSSL-22.2.0.dev0.tar.gz
```

#### 测试步骤

测试步骤跟基于OpenSSL 3.x 的 python 的商密签名实践一样，在此不做赘述。

{{#template template/footer.md}}
