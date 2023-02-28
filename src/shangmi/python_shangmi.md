# Python国密签名实践（pyopenssl + cryptography + OpenSSL）

## pyopenssl与cryptography python库介绍
### pyopenssl
[pyopenssl](https://github.com/pyca/pyopenssl)是一个基于OpenSSL库的Python wrapper。
- 其中源码目录下crypto.py是[OpenSSL crypto密码学库](https://github.com/openssl/openssl/tree/master/crypto)的wrapper，通过调用crypto.py的函数来实现对应的OpenSSL crypto API的调用 

```
├── src
│   └── OpenSSL
│       ├── crypto.py
│       ├── debug.py
│       ├── __init__.py
│       ├── rand.py
│       ├── SSL.py
│       ├── _util.py
│       └── version.py
```

以crypto.sign(位于pyopenssl/src/OpenSSL/crypto.py中)为例：会调用OpenSSL._util（位于pyopenssl/src/OpenSSL/_util.py中）里面的lib API完成签名

```Python
from OpenSSL._util import (
    ffi as _ffi,
    lib as _lib,
    exception_from_error_queue as _exception_from_error_queue,
    byte_string as _byte_string,
    native as _native,
    path_string as _path_string,
    UNSPECIFIED as _UNSPECIFIED,
    text_to_bytes_and_warn as _text_to_bytes_and_warn,
    make_assert as _make_assert,
)

def sign(pkey, data, digest):
    """
    Sign a data string using the given key and message digest.

    :param pkey: PKey to sign with
    :param data: data to be signed
    :param digest: message digest to use
    :return: signature

    .. versionadded:: 0.11
    """
    data = _text_to_bytes_and_warn("data", data)

    digest_obj = _lib.EVP_get_digestbyname(_byte_string(digest))
    if digest_obj == _ffi.NULL:
        raise ValueError("No such digest method")

    if _lib.OPENSSL_VERSION_NUMBER < 0x30000000 and _lib.EVP_PKEY_id(pkey._pkey) == _lib.EVP_PKEY_EC :
        if _lib.EC_GROUP_get_curve_name(_lib.EC_KEY_get0_group(_lib.EVP_PKEY_get1_EC_KEY(pkey._pkey))) == NID_sm2 :
            _lib.EVP_PKEY_set_alias_type(pkey._pkey, EVP_PKEY_SM2)
    md_ctx = _lib.Cryptography_EVP_MD_CTX_new()
    md_ctx = _ffi.gc(md_ctx, _lib.Cryptography_EVP_MD_CTX_free)

    _lib.EVP_SignInit(md_ctx, digest_obj)
    _lib.EVP_SignUpdate(md_ctx, data, len(data))

    length = _lib.EVP_PKEY_size(pkey._pkey)
    _openssl_assert(length > 0)
    signature_buffer = _ffi.new("unsigned char[]", length)
    signature_length = _ffi.new("unsigned int *")
    final_result = _lib.EVP_SignFinal(
        md_ctx, signature_buffer, signature_length, pkey._pkey
    )
    _openssl_assert(final_result == 1)

    return _ffi.buffer(signature_buffer, signature_length[0])[:]
```

而OpenSSL._util（位于pyopenssl/src/OpenSSL/_util.py中）的lib来源: [cryptography](https://github.com/pyca/cryptography)这个python密码学库

```python
from cryptography.hazmat.bindings.openssl.binding import Binding

binding = Binding()
binding.init_static_locks()
ffi = binding.ffi
lib = binding.lib
```

由此可见，pyopenssl借助于[cryptography](https://github.com/pyca/cryptography)这个python密码学库完成了签名。关于cryptography的介绍详见下问

### cryptography

接着上文的pyopenssl的crypto.sign的调用进行分析，可以看到在cryptography中binding.py的Binding类会调用lib库（cryptography.hazmat.bindings._openssl即`_openssl.so`）
 
```python
from cryptography.hazmat.bindings._openssl import ffi, lib

class Binding(object):
    """
    OpenSSL API wrapper.
    """

    lib = None
    ffi = ffi
    _lib_loaded = False
    _init_lock = threading.Lock()

    def __init__(self):
        self._ensure_ffi_initialized()

    @classmethod
    def _register_osrandom_engine(cls):
        # Clear any errors extant in the queue before we start. In many
        # scenarios other things may be interacting with OpenSSL in the same
        # process space and it has proven untenable to assume that they will
        # reliably clear the error queue. Once we clear it here we will
        # error on any subsequent unexpected item in the stack.
        cls.lib.ERR_clear_error()
        if cls.lib.CRYPTOGRAPHY_NEEDS_OSRANDOM_ENGINE:
            result = cls.lib.Cryptography_add_osrandom_engine()
            _openssl_assert(cls.lib, result in (1, 2))

    @classmethod
    def _ensure_ffi_initialized(cls):
        with cls._init_lock:
            if not cls._lib_loaded:
                cls.lib = build_conditional_library(lib, CONDITIONAL_NAMES)
                cls._lib_loaded = True
                # initialize the SSL library
                cls.lib.SSL_library_init()
                # adds all ciphers/digests for EVP
                cls.lib.OpenSSL_add_all_algorithms()
                                                       
```

除了阅读源码（这里不再继续详细展开，感兴趣的可以阅读对应的源码）看到cryptography调用OpenSSL完成一些密码学操作（比如签名外），还可以通过ldd可以看到这个cryptography.hazmat.bindings._openssl（即`_openssl.so`）对OpenSSL的依赖

```shell
# pwd
/usr/lib64/python2.7/site-packages/cryptography/hazmat/bindings
# ldd _openssl.so
        linux-vdso.so.1 (0x00007fffc81f7000)
        libssl.so.1.1 => /lib64/libssl.so.1.1 (0x00007fccbc400000)
        libcrypto.so.1.1 => /lib64/libcrypto.so.1.1 (0x00007fccbbe00000)
        libpthread.so.0 => /lib64/libpthread.so.0 (0x00007fccbba00000)
        libpython2.7.so.1.0 => /lib64/libpython2.7.so.1.0 (0x00007fccbb600000)
        libc.so.6 => /lib64/libc.so.6 (0x00007fccbb200000)
        libz.so.1 => /lib64/libz.so.1 (0x00007fccbae00000)
        libdl.so.2 => /lib64/libdl.so.2 (0x00007fccbaa00000)
        /lib64/ld-linux-x86-64.so.2 (0x00007fccbcb49000)
        libutil.so.1 => /lib64/libutil.so.1 (0x00007fccba600000)
        libm.so.6 => /lib64/libm.so.6 (0x00007fccba200000)
```

## python的商密签名实践

结合上文介绍，可以看到在签名上，
- python的基础密码学库cryptography依赖OpenSSL
- pyopenssl又在cryptography上进行封装，使得python的签名变得简单

龙蜥社区不仅在OpenSSL社区参与SM2/3/4等国密算法的优化，而且也在cryptography和pyopenssl仓库导出OpenSSL的国密签名来保障python的用户能够正常使用。接下来我们分别以OpenSSL 3.x和OpenSSL 1.1.x两个支持国密版本的OpenSSL为例阐述python的商密签名实践。

### 基于OpenSSL 3.x的python的商密签名(SM2-with-SM3)实践

正常安装OpenSSL 3.x和pyopenssl以及cryptography后（无需修改）后，生成SM2-with-SM3所需要的商密密钥与证书

```shell
openssl ecparam -genkey -name SM2 -out private.pem
openssl req -new -x509 -days 36500 -key private.pem -out cert.crt -sm3 -subj "/C=CN/ST=Zhejiang/L=Hangzhou/O=Alibaba/OU=OS/CN=CA/emailAddress=ca@foo.com"
```

开发商密签名测试程序test_shangmi.py（以python 3为例），注意上文的pyopenssl与cryptography编译安装时与测试程序的python大版本号（同是python 2或者同是python 3）保持一致, 该测试程序功能主要是：
- 加载之前生成好的SM2-with-SM3的私钥和证书
- 调用pyopenssl的sign和verify函数进行SM2-with-SM3的商密签名和验签

```Python3
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

执行测试程序即可(无报错则验签成功，详见pyopenssl对应的验签函数逻辑)
```shell
./test_shangmi.py private.pem cert.crt
```

### 基于OpenSSL 1.1.x的python的商密签名实践
#### OpenSSL 1.1.x的选择
OpenSSL社区1.1.x版本对国密算法的支持能力有限且不支持SM2-with-SM3这种组合算法。龙蜥社区移植了部分国密能力（包括SM2-with-SM3这种组合算法）到对应的[龙蜥openssl仓库](https://gitee.com/anolis/openssl/tree/anolis_sm234/)以及[龙蜥RPM仓库](https://gitee.com/src-anolis-os/openssl/tree/a8)，未来Anolis 8.8镜像可以默认安装龙蜥移植后的OpenSSL。当然你也可以使用[Tongsuo](https://github.com/Tongsuo-Project/Tongsuo)(原BabaSSL)作为支持国密算法且兼容OpenSSL 1.1.x的密码库。

除了对国密的支持外，在[OpenSSL社区官方文档](https://github.com/openssl/openssl/blob/OpenSSL_1_1_1/doc/man3/EVP_PKEY_set1_RSA.pod)中指出, 需要使用`EVP_PKEY_set_alias_type(pkey, EVP_PKEY_SM2)` 函数将ECCkey其转换为SM2 算法。所以基于OpenSSL 1.1.x的python的商密签名实践中，也需要对python库cryptography和pyopenssl库进行改造来导出`EVP_PKEY_set_alias_type`函数以及添加对应的逻辑（原有的cryptography和pyopenssl库并不支持这一能力）。

#### cryptography
龙蜥社区经过一些开发和测试在cryptography中提交并合入了导出`EVP_PKEY_set_alias_type`函数的代码，详见https://github.com/pyca/cryptography/pull/7935 ，以便用户使用SM2算法，因此用户在使用cryptography仓库时无需修改。可以看到在安装cryptography（这里以python2为例）后可以在`_openssl.so`（上文已经详细介绍）里看到对应的`EVP_PKEY_set_alias_type`函数符号

```shell
# strings /usr/lib64/python2.7/site-packages/cryptography/hazmat/bindings/_openssl.so | grep EVP_PKEY_set_alias_type
EVP_PKEY_set_alias_type
EVP_PKEY_set_alias_type
_cffi_d_EVP_PKEY_set_alias_type
_cffi_f_EVP_PKEY_set_alias_type
.annobin__cffi_d_EVP_PKEY_set_alias_type.start
.annobin__cffi_d_EVP_PKEY_set_alias_type.end
_cffi_d_EVP_PKEY_set_alias_type
.annobin__cffi_f_EVP_PKEY_set_alias_type.start
.annobin__cffi_f_EVP_PKEY_set_alias_type.end
_cffi_f_EVP_PKEY_set_alias_type
EVP_PKEY_set_alias_type@@OPENSSL_1_1_1
```

#### pyopenssl的修改

至于也需要一些修改，目前龙蜥社区已经在pyopenssl发起了，正在review中，详见https://github.com/pyca/pyopenssl/pull/1172 。在代码合入之前仍需要用户手动打入patch，可参考如下步骤(以python 3为例)进行安装。
```shell
git clone https://github.com/hustliyilin/pyopenssl.git
python3 setup.py sdist
cd dist
pip3 install pyOpenSSL-22.2.0.dev0.tar.gz
```

#### 测试步骤

测试步骤跟基于OpenSSL 3.x的python的商密签名实践一样，在此不做赘述。
