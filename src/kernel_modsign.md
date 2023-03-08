# 内核模块签名

## 内核模块签名简介

Linux内核支持只加载认证了的内核模块，这是一个重要的安全机制，开启这个功能时，可以阻止来源不明和没有认证的内核模块加载到内核，用以保护系统的安全性，这个认证也是通过数字签名来保证的，不同于IMA的签名，Linux内核模块的签名是以特定的数据格式追加在文件结尾的，使用的算法通常是国际算法。

涉及到签名的场景就可以使用商密算法来替换掉国际算法，要使内核模块签名支持使用商密算法，也需对签名工具和内核本身做修改，签名工具和内核同时需要支持解析使用商密算法的PKCS#7数字签名。

## 内核模块签名国密实践

> 🟢 **准备环境**

开始之前，我们仍然需要准备与IMA类似的环境，生成商密密钥和证书，使用新生成的根证书编译安装内核。不过这次生成的密钥是用于签名内核模块，在内核ko模块文件签名时我们选择使用根证书ca.cert直接签名，因此可以不需要生成sm2.cert证书。

参考IMA章节的如下两小节准备ko模块签名的环境：

* 生成商密密钥和证书（可以不用生成sm2.cert）
* 编译并安装新内核

> 🟢 **sign-file签名ko文件**

接下来我们使用内核提供的`sign-file`工具，给一个内核模块ko文件加上商密签名，命令如下

```sh
<kernel_src>/scripts/sign-file sm3 ca.key ca.crt <file>.ko <file>.ko.signed
```

通过`tail <file>.ko.signed` 看到`~Module signature appended~`字样说明模块已经有了签名。

> 🟢 **内核验证验名**

内核会在加载模块时自动执行ko文件签名验证流程，当验证失败时，内核会根据配置来决定是拒绝加载还是发出告警信息，默认配置是验签失败时告警。

`insmod`和`modprobe`工具用于加载模块。`/proc/modules`文件中记录了已加载模块的状态，包括是否通过了签名验证的状态，这里关心以下两个标记：

- O: Out-of-tree module has been loaded
- E: Unsigned module has been loaded

如果你的模块签名正确，可以看到对应的模块**没有**`E`标记，以Out-of-tree的hello.ko为例，正确签名后是这样的：

```shell
# insmod hello.ko

# cat /proc/modules
hello 262144 0 - Live 0xffff000003520000 (O)
```

如果你的模块验签失败或者未签名，可以看到`E`标记， 比如：

```shell
# modprobe nft_fib_inet.ko

# cat /proc/modules
nft_fib_inet 262144 1 - Live 0xffff0000034a0000 (E)
```

这时候可以通过`dmesg`查看详细的错误原因，比如`PKCS#7 signature not signed with a trusted key`是指使用了错误的私钥来签名。

{{#template template/footer.md}}
