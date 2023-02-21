# 磁盘加密（LUKS）

类似于文件加密，磁盘加密很重要的一点也是为了解决因存储介质丢失而导致的敏感数据泄露问题。磁盘加密是以磁盘为加密对象来保护重要数据，磁盘之上的文件甚至文件系统对磁盘加密来说是透明的。

## dm-crypt 和 LUKS 简介

`dm-crypt`是 Linux 内核提供的一个磁盘加密功能，负责对块设备进行加解密操作。dm-crypt 是 device-mapper 构架中用于块设备加密的一个模块。dm-crypt 通过 dm 框架虚拟一个块设备，并在BIO转发的时候将数据加密后存储来实现块设备的加密，而这些对于应用层是透明的。

> 🟢 **dm-crypt 的特点**

* 支持多种加密格式

    目前 dm-crypt 支持如下几种加密模式：

    1. LUKS（Linux Unified Key Setup）：这是 dm-crypt 最常用的一种模式，本节也是以 LUKS 为主展开。
    2. Plain：Plain 模式使用单个无salt的哈希值逐个扇区进行加密。
    3. loop-AES：loop-AES 是一款比较陈旧的 Linux 磁盘加密工具。dm-crypt 提供了对它的支持。
    4. TCRYPT：在 cryptsetup 的 1.6.0 版本之后，开始提供对 TrueCrypt 加密盘的支持。`TCRYPT` 是 `TrueCrypt` 的缩写。在该模式下，可以打开 TrueCrypt 和 VeraCrypt 的加密盘，并对盘中的文件进行读写。

* 无需额外安装软件

    由于 dm-crypt 早已被整合到 Linux Kernel 中。因此，无需额外安装它。至于它的命令行前端（cryptsetup），大部分主流的发行版都会内置 cryptsetup 的软件包。

* 可以跟 LVM 无缝整合

    LVM（Logical Volume Manager）是 Linux 内核提供的另一个很有用的工具。比如用它来创建分区，将来可以随时调整分区大小；比如现有的硬盘空间用完了，可以另外加一块硬盘并且新加硬盘可以用来扩展现有分区。LVM 和 dm-crypt 都是基于 Linux 内核的 device mapper 机制。因此两者可以很好地整合。

> 🟢 **cryptsetup**

`cryptsetup` 是与 dm-crypt 交互的命令行工具，用于创建、访问和管理加密设备，主流的发行版已经内置了该工具。

从原理上来说，cryptsetup 其实是一种设备的映射关系，我们用它来把一个设备映射成另外一个设备，然后对这个新的设备进行操作，并进行加密，这样就不会使我们的原设备直接被使用，从而达到一种安全的效果。

> 🟢 **LUKS**

LUKS （Linux Unified Key Setup）是 Linux 硬盘加密的标准。 通过提供标准的磁盘格式，它不仅可以促进发行版之间的兼容性，还可以提供对多个用户密码的安全管理。 与现有解决方案相比，LUKS 将所有必要的设置信息存储在分区信息首部中，使用户能够无缝传输或迁移其数据。

## 使用 SM4 算法加密 LUKS 磁盘

⚠️  **选择一个不用的磁盘或者磁盘分区，该操作会清空设备上的所有数据，请谨慎操作。**

这里选择使用 vda4 作为实验分区，使用 LUKS 格式格式化要加密的磁盘分区，加密算法是SM4 XTS：

```sh
# 根据提示输入大写的 YES 和密码完成格式化操作
> cryptsetup --cipher sm4-xts-plain64 --key-size=256 --hash sm3 luksFormat /dev/vda4

# 打开该加密分区，密码正确后，会创建代表该分区的透明设备 /dev/mapper/diskluks
# 该设备展示给用户的是一个未加密的普通的分区设备，可以对它进行任何针对分区的操作
> cryptsetup luksOpen /dev/vda4 diskluks

# 使用status子命令可以看到加密分区的状态信息
> cryptsetup status diskluks
/dev/mapper/diskluks is active.
  type:    LUKS2
  cipher:  sm4-xts-plain64
  keysize: 256 bits
  key location: keyring
  device:  /dev/vda4
  sector size:  512
  offset:  32768 sectors
  size:    14645248 sectors
  mode:    read/write

# 此时，diskluks表现为一个普通分区，我们可以格式化为任意支持的文件系统，并挂载它
> mkfs.ext4 /dev/mapper/diskluks
> mount /dev/mapper/diskluks /mnt/

# 在磁盘树中可以观察到加密的设备和代表它的透明未加密设备diskluks
> lsblk -f
NAME         FSTYPE      FSVER LABEL   UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
vda
├─vda1
├─vda2       ext4        1.0   rootfs  e958d21c-63aa-46d7-a215-be264ccb02d5   23.2G    16% /
├─vda3       ext4        1.0   fscrypt 0463916b-cd52-40f3-9c95-290bf4839d8e
└─vda4       crypto_LUKS 2             b5338a30-6974-45f4-81da-98f1cb0cab72
  └─diskluks ext4        1.0           683a0dc2-0755-4759-b94c-54687c706dd5    6.4G     0% /mnt

# 取消挂载
> umount /mnt

# 锁定加密分区，diskluks设备被删除，此时没有密码的用户看不到明文数据
> cryptsetup close diskluks
```

## 后记

SM4 的密钥长度是 128 位，因为 XTS 模式使用两个密钥，所以这里需要显式指定密钥长度是 256 位。

除了密码之外，还可以选择使用密钥文件解密硬盘，也就是相当于一个密钥，当然也可以只使用密钥文件或者同时使用密码与密钥文件，用户需要使用子命令 `cryptsetup luksAddKey` 和 `cryptsetup luksRemoveKey` 来管理密钥。

{{#template ../template/footer.md}}
