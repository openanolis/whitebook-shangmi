# Kernel TLS（KTLS）实践

## KTLS 简介

传输层安全协议（TLS）一般是在用户态库中实现，比如OpenSSL和gnutls库都实现了TLS协议，除了与内核的网络交互外，协议自身对内核是透明的。Kernel
TLS（KTLS）从名字可以看到，是把TLS协议的数据平面（相对于处理协议的控制平面来说，也就是真正用于传输负载数据的部分）下沉到了内核中，发送数据时由内核加密后直接发送到网络设备，接收到数据后先解密再交给应用，呈现在用户态的仍然是明文。

![TLS Offload Layers](images/tls-offload-layers.svg)

> 🟢 **优势**

众所周知，`sendfile`在内核提供的重要的优化网络服务的手段，但完全实现在用户态的TLS库是不支持`sendfile`的，因为要把文件内容读入用户态，加密后再交给内核发送到网络设备。

在使用Kernel TLS 时，一定程度上是解决了在TLS中使用`sendfile`的问题，KTLS 是内核提供的又一个重要的优化TLS协议的手段。尤其是在静态文件服务的场景下，对TLS协议的性能提升明显，对基于TLS协议的https流量的性能提升非常有帮助，因为在内核加解密数据会减少数据在内核态和用户态的两次数据拷贝。

## KTLS 国密实践

[RFC 8998](https://datatracker.ietf.org/doc/html/rfc8998)定义了在TLS 1.3协议中使用国密算法套件的规范，KTLS 也支持使用 SM4 CCM/GCM AEAD 算法加密数据。

内核的KTLS跟应用是通过sockopt来交互的，开发者需要明确告诉内核某个套接字要使用KTLS并且传递加密所需要的各种信息，比如密钥，记录序号等。

以下示例演示了在网络应用中使用 SM4 GCM 算法在内核加密数据。注意，内核KTLS依赖TLS协议握手之后得到的密钥数据，但不必实现完整的TLS协议，即便这个密钥不是TLS握手得到的。

> 🟢 **准备套接字**

创建新的TCP套接字，并通过 SOL_TCP 选项明确告诉内核使用KTLS：

```c
sock = socket(AF_INET, SOCK_STREAM, 0);
setsockopt(sock, SOL_TCP, TCP_ULP, "tls", sizeof("tls"));
```

以下内容是在 `linux/tls.h` 头文件中与国密相关的定义，会在之后的代码片断中引用：

```c
/* From linux/tls.h */
struct tls_crypto_info {
        unsigned short version;
        unsigned short cipher_type;
};

struct tls12_crypto_info_sm4_gcm {
	struct tls_crypto_info info;
	unsigned char iv[TLS_CIPHER_SM4_GCM_IV_SIZE];
	unsigned char key[TLS_CIPHER_SM4_GCM_KEY_SIZE];
	unsigned char salt[TLS_CIPHER_SM4_GCM_SALT_SIZE];
	unsigned char rec_seq[TLS_CIPHER_SM4_GCM_REC_SEQ_SIZE];
};

struct tls12_crypto_info_sm4_ccm {
	struct tls_crypto_info info;
	unsigned char iv[TLS_CIPHER_SM4_CCM_IV_SIZE];
	unsigned char key[TLS_CIPHER_SM4_CCM_KEY_SIZE];
	unsigned char salt[TLS_CIPHER_SM4_CCM_SALT_SIZE];
	unsigned char rec_seq[TLS_CIPHER_SM4_CCM_REC_SEQ_SIZE];
};

#define TLS_CIPHER_SM4_GCM				55
#define TLS_CIPHER_SM4_GCM_IV_SIZE			8
#define TLS_CIPHER_SM4_GCM_KEY_SIZE		16
#define TLS_CIPHER_SM4_GCM_SALT_SIZE		4
#define TLS_CIPHER_SM4_GCM_TAG_SIZE		16
#define TLS_CIPHER_SM4_GCM_REC_SEQ_SIZE		8

#define TLS_CIPHER_SM4_CCM				56
#define TLS_CIPHER_SM4_CCM_IV_SIZE			8
#define TLS_CIPHER_SM4_CCM_KEY_SIZE		16
#define TLS_CIPHER_SM4_CCM_SALT_SIZE		4
#define TLS_CIPHER_SM4_CCM_TAG_SIZE		16
#define TLS_CIPHER_SM4_CCM_REC_SEQ_SIZE		8
```

设置 TLS_ULP 后允许我们设置/获取 TLS 套接字选项。 国密算法目前只支持带认证的 GCM/CCM 模式。 TLS 握手完成后，我们拥有将数据平面下沉到内核所需的所有参数。 有一个单独的套接字选项`SOL_TLS`用于将传输和接收下沉到内核中。

KTLS 的发送和接收是单独设置的，设置方法是相同的，发送使用`TLS_TX`，接收使用`TLS_RX`。

```c
struct tls12_crypto_info_sm4_gcm crypto_info;

crypto_info.info.version = TLS_1_3_VERSION;
crypto_info.info.cipher_type = TLS_CIPHER_SM4_GCM;
memcpy(crypto_info.iv, iv_write, TLS_CIPHER_SM4_GCM_IV_SIZE);
memcpy(crypto_info.rec_seq, seq_number_write, TLS_CIPHER_SM4_GCM_REC_SEQ_SIZE);
memcpy(crypto_info.key, cipher_key_write, TLS_CIPHER_SM4_GCM_KEY_SIZE);
memcpy(crypto_info.salt, implicit_iv_write, TLS_CIPHER_SM4_GCM_SALT_SIZE);

setsockopt(sock, SOL_TLS, TLS_TX, &crypto_info, sizeof(crypto_info));
```

> 🟢 **发送 TLS 负载数据**

设置了 `TLS_TX` 套接字选项后，通过该套接字发送的所有应用程序数据都使用 TLS 和套接字选项中提供的参数进行加密。 例如，我们可以发送一条加密的 hello world 记录，如下所示：

```c
const char *msg = "hello world\n";
send(sock, msg, strlen(msg));
```

如果可能的话，内核会使用用户态提供的缓冲区直接加密后发送。

同样可以顺滑的使用sendfile通过TLS发送文件数据：

```c
file = open(filename, O_RDONLY);
fstat(file, &stat);
sendfile(sock, file, &offset, stat.st_size);
```

> 🟢 **接收 TLS 负载数据**

与发送相似，设置 `TLS_RX` 套接字选项后，所有类 recv 的调用都使用提供的 TLS 参数进行解密。 在解密发生之前必须收到完整的 TLS 记录。

```c
char buffer[16384];
recv(sock, buffer, sizeof(buffer));
```

如果接收到的数据足够大，则直接将其解密到用户缓冲区中，并且不会发生额外的分配。 如果用户空间缓冲区太小，数据在内核中解密并复制到用户空间。

> 🟢 **TLS 控制消息**

除了应用程序数据，TLS 还具有控制消息，例如警报消息（记录类型 21）和握手消息（记录类型 22）等。这些消息可以通过 CMSG 提供 TLS 记录类型通过套接字发送。 控制消息是完全符合TLS协议规范的，关于如何通过 KTLS 来发送或者接收 TLS 控制消息，可以参考[内核文档](https://www.kernel.org/doc/html/latest/networking/tls.html)。

{{#template ../template/footer.md}}
