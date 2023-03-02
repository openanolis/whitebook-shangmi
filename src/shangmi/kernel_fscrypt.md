# 文件加密（fscrypt）

通常我们文件作为数据载体，使用磁盘，USB 闪存，SD 卡等存储介质进行数据存储，即便我们已经离线存储，仍然不能保证该存储介质不会丢失，如果丢失那么对于我们来说有可能是灾难性的事件。因此对这些离线存储的重要数据文件进行加密是非常有必要的，本节将介绍如何使用国密算法加密文件系统中的文件。

## fscrypt 简介

内核中的 fscrypt 是一个库，文件系统可以挂钩它以支持文件和目录的透明加密。

与 dm-crypt 不同，fscrypt 在文件系统级别而不是块设备级别运行。 这允许它使用不同的密钥加密不同的文件，并在同一文件系统上拥有未加密的文件。 这对于多用户系统非常有用，在该系统中，每个用户的静态数据都需要与其他用户进行加密隔离。 除了文件名，fscrypt 不加密文件系统的元数据。

与作为堆栈文件系统的 eCryptfs 不同，fscrypt 直接集成到支持的文件系统中，目前是 ext4、F2FS 和 UBIFS。fscrypt 允许读取和写入加密文件，而无需在页面缓存中同时缓存解密和加密页面，从而将使用的内存几乎减半并使其与未加密文件保持一致。 同样，需要一半的 dentry 和 inode。 eCryptfs 还将加密文件名限制为 143 字节，从而导致应用程序兼容性问题； fscrypt 允许完整的 255 个字节 (NAME_MAX)长度的文件名。 最后，与 eCryptfs 不同，fscrypt API 可以由非特权用户使用，无需依赖任何组件。

fscrypt 不支持就地加密文件。 相反，它支持将空目录标记为已加密。 然后，在用户空间提供密钥后，在该目录树中创建的所有常规文件、目录和符号链接都将被透明地加密。

## 支持的加密模式和用法

fscrypt 允许为文件内容指定一种加密模式，为文件名指定一种加密模式。 不同的目录树允许使用不同的加密方式。 目前支持以下几种加密方式对：

* AES-256-XTS 算法用于加密内容，AES-256-CTS-CBC 算法用于加密文件名
* AES-128-CBC 算法用于加密内容，AES-128-CTS-CBC 算法用于加密文件名
* Adiantum 算法同时用于加密文件内容和文件名
* AES-256-XTS 算法用于加密内容，AES-256-HCTR2 算法用于加密文件名（仅限 v2 策略）
* SM4-XTS 算法用于加密内容，SM4-CTS-CBC 算法用于加密文件名（仅限 v2 策略）

AES-128-CBC 仅为具有不支持 XTS 模式的加速器的低功耗嵌入式设备使用。 要使用 AES-128-CBC，必须启用 CONFIG_CRYPTO_ESSIV 和 CONFIG_CRYPTO_SHA256（或其他 SHA-256 实现）以便使用 ESSIV。

Adiantum 是一种基于流密码的模式，即使在没有专用加密指令的 CPU 上也很快。 与 XTS 不同，它也是真正的宽块模式。 它还可以消除派生每个文件加密密钥的需要。 要使用 Adiantum，必须启用 CONFIG_CRYPTO_ADIANTUM。 此外，应启用 ChaCha 和 NHPoly1305 的快速实现，例如 ARM 架构上的 CONFIG_CRYPTO_CHACHA20_NEON 和 CONFIG_CRYPTO_NHPOLY1305_NEON。

AES-256-HCTR2 是另一种真正的宽块加密模式，旨在用于具有专用加密指令的 CPU。 AES-256-HCTR2 具有明文中的位翻转会更改整个密文的属性。 由于初始化向量在目录中重复使用，因此此属性使其成为文件名加密的理想选择。 要使用 AES-256-HCTR2，必须启用 CONFIG_CRYPTO_HCTR2。 此外，应启用 XCTR 和 POLYVAL 的快速实现，例如 用于 ARM64 的 CRYPTO_POLYVAL_ARM64_CE 和 CRYPTO_AES_ARM64_CE_BLK。

最后是 SM4 算法，目前仅在 fscrypt v2 策略中启用。

## 使用 SM4 算法加密文件

> 🟢 **准备工作**

fscrypt 依赖内核配置`CONFIG_FS_ENCRYPTION=y`，这里操作系统选择使用 ANCK 5.10 内核的 Anolis OS，其次，需要支持 fscrypt 特性的文件系统，这里以 ext4 为例，当然，F2FS 或者 UBIFS 也可以。

用户空间是通过 fscrypt API 跟内核完成交互的，对于用户来说，一般是通过`fscryptctl`或者`fscrypt`工具来下达加密策略。

本节内容以 **fscryptctl（<https://github.com/google/fscryptctl>）** 工具为例来演示，目前这是一个第三方工具，需要手工安装，按如下常规流程安装：

```sh
git clone https://github.com/google/fscryptctl.git

cd fscryptctl

make

make install
```

其次，选择一块未用到的磁盘格式化为支持 fscrypt 的文件系统 ext4，并挂载。

```sh
mkfs.ext4 -O encrypt /dev/vdb

mount /dev/vdb /mnt
```

> 🟢 **透明加密文件**

fscrypt 所用的加解密钥是关联在超级块上的，运行时是跟挂载点相关联的，添加删除密钥都是针对挂载点的操作，以下对密钥操作的命令都会带上挂载点。

按如下命令所示设置加密策略：

```shell
# 生成密钥文件，实际环境中应用使用更复杂的密钥
> echo '1234567812345678' > /tmp/keyfile

# 添加该密钥到文件系统，返回密钥ID，之后对密钥的操作都使用这个ID来索引
> fscryptctl add_key /mnt < /tmp/keyfile
23086a13ed81fd75ca5fe9b8f2ff25c7

# 查看密钥状态（不是必需）
> fscryptctl key_status 23086a13ed81fd75ca5fe9b8f2ff25c7 /mnt
Present (user_count=1, added_by_self)

# 创建加密目录 endir，并设置加密策略
# 使用之前添加的密钥和 SM4 算法来加密该目录中的文件和子目录
> mkdir /mnt/endir
> fscryptctl set_policy --contents=SM4-XTS \
        --filenames=SM4-CTS 23086a13ed81fd75ca5fe9b8f2ff25c7 /mnt/endir

# 查看策略是否生效（不是必需）
> fscryptctl get_policy /mnt/endir
Encryption policy for /mnt/endir:
	Policy version: 2
	Master key identifier: 23086a13ed81fd75ca5fe9b8f2ff25c7
	Contents encryption mode: SM4-XTS
	Filenames encryption mode: SM4-CTS
	Flags: PAD_32
```

此时，endir 已经是支持透明加解密的一个目录，可以像正常目录一样创建删除文件，在该目录下进行一些常规的文件操作，可以看到与普通目录没有区别：

```shell
> mkdir /mnt/endir/foo
> echo 'hello' > /mnt/endir/foo/hello

> cp -v /usr/include/curl/* endir
> tree /mnt/endir
/mnt/endir
├── curl.h
├── curlver.h
├── easy.h
├── foo
│   └── hello
├── header.h
├── mprintf.h
├── multi.h
├── options.h
├── stdcheaders.h
├── system.h
├── typecheck-gcc.h
├── urlapi.h
└── websockets.h

1 directories, 13 files
```

> 🟢 **锁定加密目录**

之所以能像普通目录一样操作，是因为密钥已经被添加到了文件系统中。接下来删除密钥后，就能看到目录被锁定，里面的所有路径和内容都是加密状态：

```shell
# 移除密钥
> fscryptctl remove_key 23086a13ed81fd75ca5fe9b8f2ff25c7 /mnt

> fscryptctl key_status 23086a13ed81fd75ca5fe9b8f2ff25c7 /mnt
Absent

# 处于加密状态的目录树
> tree /mnt/endir
/mnt/endir
├── 1H2e0BbS4MGZKAKEu6NVXniaYMWIrWDwbyzX6EVEWEN8tfWcWNgDyw
├── 2otRhm5-MDOSKyICcSyBWdKghJIsrsAl5xMCsCX0nCWQN2mC3gKBCg
├── 5MJXflC0Pf81ZlnV2YKmDg01tkRdKXPsmwZEesS-Q8gCLu-nuGRnSg
├── 5vPk9fvei-vU2RQp3tub8v3uZf_hOKje5kpGMn-qoTDkfhtU9C4Y5g
├── 7F6rs2Zf-ogIMXwQdOi0sKtT2vtq2d7XKOkXJ4PPfx1pAKNXDjYHkw
├── LvYw6Jl0a1jImKKOFPjtpG3hEDxjjuM6YIYqcMeXaWdzKUdaX0YCNQ
├── QBBz8_qGE4MJY6YVzfqVUkr6YeCSqtoQmbvG04BsR0lAr2oLwO0b2g
│   └── wOYdFlMRACjeBa-eSo3LuO4sE55q1YuFv-S_lVU-n498jdMjAt06JA
├── WBtjWd12dIHLd1XQ2fN_VnN8EGP1CrMJgqQLQ6Zt9No34mbNibCGSA
├── cHGhd03URFdNl19DExe26X6w2NsQC2ixUspPNdhU-1nrgKDDwPTMWg
├── eBGQnXyUMzPEOY3sHWVZNSeWKGm6C1NYCyEkO9Nm_dqNI15JAi6MzQ
├── mkzb9jZ5jk8A259-k8U34_4qi64SpJBKQOhwdTEIIiaHG7aLryMOGQ
├── tE1hArEQub5O88_prGmdVoj73W7eb-iqaQ4GEetgI8nEDyVIK4K08Q
└── zoiobWxVG2DLjg8uMXfsVP11159zqQUjozJ8gmt1zyjayJlZ4awOhA

1 directory, 13 files

# 目录被锁定，无法进行常规文件操作，即便拔盘，也不能得到明文内容
> cat /mnt/endir/1H2e0BbS4MGZKAKEu6NVXniaYMWIrWDwbyzX6EVEWEN8tfWcWNgDyw
cat: /mnt/endir/1H2e0BbS4MGZKAKEu6NVXniaYMWIrWDwbyzX6EVEWEN8tfWcWNgDyw: Required key not available
> mkdir /mnt/endir/hello
mkdir: cannot create directory ‘/mnt/endir/hello’: Required key not available
```

> 🟢 **再次解锁加密目录**

要解锁目录也很简单，重新添加密钥即可，文件系统会搜索到正确的密钥并解锁相应目录：

```shell
> fscryptctl add_key /mnt < /tmp/keyfile
23086a13ed81fd75ca5fe9b8f2ff25c7

# 添加密钥后文件内容可正常访问
> cat /mnt/endir/foo/hello
hello
```

## 后记

fscryptctl 是一个相对原生的工具，更接近内核，可以看到，该工具命令比较复杂，使用中需要记住很长一串密钥ID，用户体验并不好。

实际环境中，一般会使用 **[fscrypt](https://github.com/google/fscrypt)** 工具来完成加密策略操作，该工具由 Google 开发，用Go语言写成，通过在用户层面维护了一些元数据来简化用户操作，命令更易于理解，也更接近用户。

{{#template ../template/footer.md}}
