# 内核完整性度量架构（IMA）

## IMA 简介

IMA是`Integrity Measurement Architecture`的缩写，它是Linux内核中完整性子系统的一部分。目前Linux内核的完整性子系统支持EVM和IMA，前者用于保护文件的扩展属性。这篇文章讨论的是后者。

IMA所能做到的事情包括：

*  能够对正在打开的文件进行完整性评估。
*  能够对正在执行exec的文件（可执行程序）进行完整性评估。
*  能够对正在执行mmap(PROT_EXEC)的文件（共享库）进行完整性评估。
*  能够对正在加载中的kernel模块和固件进行完整性评估。

所谓的完整性评估指的是对内核对文件客体在执行特定的内核操作时，会主动对文件的内容进行完整性检查。为此，IMA子系统会借用内核的security子系统在open(), execve(), mmap()等系统调用中下的hook来执行自己的代码。

## IMA 原理

IMA在进行完整性验证时，会通过事先存储在文件系统中的文件扩展属性security.ima进行。具体来说，借用IMA签名工具evmctl，在系统部署的时候，管理员以特权用户身份将文件的完整性信息写入文件扩展属性security.ima中。

在系统运行时，IMA子系统会从该扩展属性中读取出文件的完整性信息，同时与实际计算出的完整性信息进行比较。如果结果一致，证明该文件没有遭到过篡改，则允许执行后续的操作；如果结果不一致，证明文件内容遭到了篡改，则后续操作禁止执行。

因此，即使攻击者通过密码破解拿到了本地特权，或者利用安全漏洞拿到了本地特权，但是在准备运行恶意程序或植入了后门的程序的时候，因为在没有IMA私钥的情况下是无法构造出合法的IMA签名的，因此导致恶意程序或被篡改了的程序均无法运行。即使带有IMA保护的存储设备受到离线攻击（比如把存储设备从主机上取下，拿到另一台机器上进行修改，然后再重新安装到主机上），被篡改的文件或攻击者植入的恶意软件在运行时依旧无法运行，这在一定程度上能够抑制类似Dirty Cow这样的内核漏洞所带来的危害。

## IMA 商密化实践

所谓IMA商密化，就是在IMA整个签名验证流程中，使用商密算法SM3代替国际常用的哈希算法SHA256，SHA512等，用SM2算法的签名验签取代RSA算法。

本文中用到的主要是以下公开的商密算法：

* SM2：基于椭圆曲线密码（ECC）的公钥密码算法标准，提供数字签名，密钥交换，公钥加密，用于替换RSA/ECDSA/ECDH等国际算法
* SM3：消息摘要算法，哈希结果为256 bits，用于替换MD5/SHA1/SHA256等国际算法

首先，安装实践IMA必要的工具包：

```sh
yum install -y keyutils ima-evm-utils
```

> 🟢 **生成商密密钥和证书**

为了使用IMA功能，先要准备以下密钥和证书：

* CA根证书：为了便于实验，这里选择自签名的证书，作为信任根内置到内核里
* IMA私钥：与IMA证书对应的SM2私钥，用于签名文件
* IMA证书：由CA根证书签名，系统启动后动态导入内核

```sh
# 创建签名证书请求使用的配置文件genkey.conf
cat > genkey.conf << EOF
[ req ]
distinguished_name = req_distinguished_name
prompt = no
string_mask = utf8only
x509_extensions = v3_ca

[ req_distinguished_name ]
O = IMA-test
CN = IMA test key
emailAddress = ima@test.com

[ v3_ca ]
basicConstraints=critical,CA:FALSE
keyUsage=digitalSignature
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid:always
EOF

# 生成一个自签名的根证书ca.cert，作为CA证书
openssl ecparam -genkey -name SM2 -text -out ca.key
openssl req -verbose -new -days 10000 -x509 \
    -sm3 \
    -sigopt "distid:1234567812345678" \
    -config genkey.conf \
    -key ca.key \
    -out ca.cert

# 生成SM2私钥sm2.key
openssl ecparam -genkey -name SM2 -text -out sm2.key

# 从私钥生成证书请求
openssl req -verbose -new \
    -sm3 \
    -sigopt "distid:1234567812345678" \
    -config genkey.conf \
    -key sm2.key \
    -out sm2.csr

# 使用CA证书给SM2证书请求签名，生成IMA要使用的商密证书sm2.cert
openssl x509 -req -days 10000 \
    -sm3 \
    -sigopt "distid:1234567812345678" \
    -vfyopt "distid:1234567812345678" \
    -CA ca.cert -CAkey ca.key -CAcreateserial \
    -extfile genkey.conf -extensions v3_ca \
    -in sm2.csr \
    -out sm2.cert
```

> 🟢 **编译并安装新内核**

为了测试和验证IMA特性，我们需要把CA根证书内置到内核，这需要使用新的CA根证书重新编译内核。

按如下步骤依次下载内核源代码，编译，安装内核后并重启系统：

```sh
# 下载Anolis OS的ANCK源代码，使用最新的5.10分支即可
git clone https://gitee.com/anolis/cloud-kernel.git -b devel-5.10

# 安装编译依赖
yum install -y bison flex elfutils-libelf-devel bc make gcc

# 用上一步生成好的ca.cert作为内核信任的根证书，使用默认配置编译内核
cp -f ca.cert <kernel_src>/certs/

# 进入内核源码目录，使用默认配置编译内核
cd <kernel_src>

# 如果您是arm的镜像，请将 arch/arm64/configs/anolis_defconfig 作为.config；
# 如果是x86的，请将 arch/x86/configs/anolis_defconfig 作为.config。以x86为例
cp -f arch/x86/configs/anolis_defconfig .config

# 配置系统的可信根证书
sed -i 's/CONFIG_SYSTEM_TRUSTED_KEYS=\"\"/CONFIG_SYSTEM_TRUSTED_KEYS=\"certs\/ca.cert\"/' .config

# 编译
make -j<nproc>

# 安装modules
make modules_install

# 安装内核，这一步也会自动生成initramfs并更新grub.cfg
make install

# 查看vmlinuz
ls -l /boot/vmlinuz*

# 将新内核设置为缺省的启动内核，这里根据实际情况自行调整
grubby --set-default /boot/vmlinuz-5.10.<minor version>

# 重启
reboot
```

重启机器后，通过`/proc/keys`或者`keyctl`可以看到我们的SM2根证书已经内置到了内核中：

```shell
# cat /proc/keys | grep sm2
02a32516 I------     1 perm 1f030000     0     0 asymmetri: IMA-test: bc08a9e6e43c30fd421ebc6fec7bb9063a089aca: X509.sm2 3a089aca []

# keyctl show %:.secondary_trusted_keys
Keyring
 371108982 ---lswrv      0     0  keyring: .secondary_trusted_keys
 945861859 ---lswrv      0     0   \_ keyring: .builtin_trusted_keys
 575535217 ---lswrv      0     0       \_ asymmetric: Build time autogenerated kernel key: 60d20efc1951f58c8bba2b04d937e4fa2a6cd62a
  44246294 ---lswrv      0     0       \_ asymmetric: IMA-test: bc08a9e6e43c30fd421ebc6fec7bb9063a089aca
```

> 🟢 **导入IMA证书到内核**

为了使用IMA功能，我们需要把前面用CA证书签名的IMA证书sm2.cert导入到内核，之后才能用该证书正确验签IMA私钥签名的文件，因为内核已经集成了CA根证书，也只有CA签名的证书才能成功导入内核。

内核只支持导入DER格式的证书，因此我们需要先将pem的证书转换为der格式，命令如下

```sh
# 内核只支持导入DER格式的证书
openssl x509 -in sm2.cert -outform der -out sm2.cert.der
```

用keyctl导入证书，注意**系统重启后会失效**

```sh
# 非持久性导入, 重启后失效
keyctl padd asymmetric "IMA" %:.ima < sm2.cert.der
```

IMA证书导入成功后，我们可以从`/proc/keys`看到证书信息
 
```sh
# IMA证书导入成功后，我们可以从/proc/keys看到证书信息：
cat /proc/keys | grep sm2
03cd0857 I------     1 perm 1f030000     0     0 asymmetri: IMA-test: 10024aa19b7b8d73f4b3b413f70bcf94806f9ca2: X509.sm2 806f9ca2 []
069ced19 I--Q---     1 perm 39010000     0     0 asymmetri IMA: X509.sm2 604c5d8c []
```

> 🟢 **IMA签名**

接下来给系统中需要IMA验证的文件加上SM2的签名，这里简单粗暴的给常用目录下文件全部签名，如果文件比较多的话，这个过程会持续几分钟。

```sh
# 使用SM3哈希算法，sm2.key是签名用的私钥，给系统主要目录下所有文件做IMA签名
evmctl ima_sign -a sm3 -s -k /path/to/sm2.key -r -t f /bin
evmctl ima_sign -a sm3 -s -k /path/to/sm2.key -r -t f /sbin
evmctl ima_sign -a sm3 -s -k /path/to/sm2.key -r -t f /usr
evmctl ima_sign -a sm3 -s -k /path/to/sm2.key -r -t f /lib
evmctl ima_sign -a sm3 -s -k /path/to/sm2.key -r -t f /lib64
evmctl ima_sign -a sm3 -s -k /path/to/sm2.key -r -t f /etc
evmctl ima_sign -a sm3 -s -k /path/to/sm2.key -r -t f /home

# 通过getfattr可以查看添加到文件扩展属性中的签名数据（不是必需）
yum install -y attr
getfattr -n security.ima /path/to/file
```

> 🟢 **导入IMA策略**

IMA策略是使能IMA的必备步骤，策略的内容可以根据实际需要进行定制。为了方便说明问题，这里以用户imatest来测试，在下面的规则示例中，当以imatest身份运行可执行程序和共享库时，会进行IMA度量以及apprase检查。

```sh
# 创建用户imatest
useradd imatest

# 用户ID可以从/etc/passwd中看到，这里是1001，用户ID会在IMA规则中用到
tail /etc/passwd | grep imatest
# imatest:x:1001:1001::/home/imatest:/bin/bash
```

将下面的文件内容另存为`ima.policy`文件：

```
appraise appraise_type=imasig uid=1001 func=BPRM_CHECK
measure uid=1001 func=BPRM_CHECK
appraise appraise_type=imasig uid=1001 func=MMAP_CHECK
measure uid=1001 func=MMAP_CHECK
```

上面的规则表示：

* 度量以uid=1001身份运行的程序和共享库，并将度量值记录在/sys/kernel/security/ima/ascii_runtime_measurements文件中。
* 评估以uid=1001身份运行的程序和共享库，如果程序的完整性被破坏，程序将被拒绝运行。

然后写入IMA规则到内核：

```sh
cat ima.policy > /sys/kernel/security/ima/policy
```

> 🟢 **IMA验证**

IMA策略写入内核后，IMA特性就已经在内核生效了，此时可以以imatest用户身份执行一些操作

```sh
su imatest

... ...
```

通过`ascii_runtime_measurements`我们可以看到IMA运行时度量的信息，这些都是通过IMA验证的文件，以下是部分度量日志：

```shell
# cat /sys/kernel/security/ima/ascii_runtime_measurements
10 bcb0e518b79de0d7f2cd20b8a29a7092e65db7ef ima-sig sha1:0000000000000000000000000000000000000000 boot_aggregate
10 d20bcf8ea6f3f8c03dd475766dfe18ccc14d264b ima-sig sm3:66acf6555ad2f6e5b89e1a80644eae4fd0015e5b811be63d1a58415e2f4f2b0e /usr/bin/bash 030211604c5d8c00473045022100ce91713b9aa37a2f8ce16f95acc00a7f072d7d3cc90ad8fae916ffc6072b10f4024
10 b829e4761f7a717dde6daab8089c8bac15fbb139 ima-sig sm3:1bcd4a4cb2a2c4418aa7075f5c012de03a383f3af172579132e5ad21bd5752c5 /usr/lib64/ld-2.17.so 030211604c5d8c0046304402203db3c575be22d35abe49d6e9f7fde0be22be02fdd9211feebf7ae172b8c66
10 be220df2bc614980a06eff27f7286305c7afbdc2 ima-sig sm3:66acf6555ad2f6e5b89e1a80644eae4fd0015e5b811be63d1a58415e2f4f2b0e /usr/bin/bash 030211604c5d8c00473045022100e66e7f2d356668c1779cfbe05838979c789d9f81c8b5cd0e19022b8255ea13f1024
10 d262c29452b0d575a81786c49453fe6f14c0969d ima-sig sm3:1bcd4a4cb2a2c4418aa7075f5c012de03a383f3af172579132e5ad21bd5752c5 /usr/lib64/ld-2.17.so 030211604c5d8c004730450220432ee49d0fe84c36633365afdd1fc138316e5ea2cb854c5dc5b0c9d03619f
10 26bf0fc75828783a0013b9f52045c9a56e8b4853 ima-sig sm3:18781f4c910453fdbcb36a359092128df86a660e230db4bb52765e32e209cf94 /usr/lib64/libtinfo.so.5.9 030211604c5d8c00473045022100fb46c2b15aef6f1005fea3f96d23d95a2d036d5da761d593b76891
10 28ee84f368f892180efaefb3d9b68c90ded73268 ima-sig sm3:1c568b3aa88a2bafe543d4aa1db95c12e7e062f54de5c55c50729fa3ac786ab2 /usr/lib64/libdl-2.17.so 030211604c5d8c00483046022100930e63c0325013bfdbdbe098bc13ac514703af2ff7998fe447675910
10 88120ffffa110b3621947f937117739f46cc8a46 ima-sig sm3:a53099f1946d13df1cf2561be1989cafbf4f1e4908294b315f80b8b07936c6d8 /usr/lib64/libc-2.17.so 030211604c5d8c00473045022010f571c748201ddc229f5b9cbbc4d471fb892b98653f8f73dc614eacfa2
10 1c83184c6b11f31ca8f1090aafc4ee5be8e336b0 ima-sig sm3:a591bc1f9eb1d1b89f0a1a83f6236302132a973515edebec1664499e32e42f12 /usr/lib64/libnss_files-2.17.so 030211604c5d8c00483046022100b51c03e638f2d71f6a4368592d8e055c231441962b1fb0989
... ...
```
注意期中第一条日志，boot_aggregate是系统启动阶段TPM PCR的汇聚值，这个值是汇聚TPM设备对应PCR bank的值做一个digest，所使用的摘要算法是根据TPM设备版本以及支持的PCR bank决定，默认是SHA1，当然这个摘要也可以通过配置为默认优先SM3算法，例子中没有TPM设备，所以这里值是0。

接下来，我们再构造一个没有经过签名的可执行文件并执行：

```sh
su imatest
cd ~

echo 'int main(){}' > dummy.c
gcc -o dummy dummy.c
```

运行dummy可执行文件，提示没有权限，这是符合预期的，同样，如果文件的签名是错误的，没有通过签名验证，是会被拒绝执行的。

```shell
# ./dummy
bash: ./dummy: Permission denied

# dmesg
audit: type=1800 audit(1631788187.296:8): pid=1426 uid=1001 auid=0 ses=4 op=appraise_data cause=IMA-signature-required comm="bash" name="/home/imatest/dummy" dev="vda2" ino=668226 res=0
```

我们看到，通过对这些软件栈的改造，可以平滑迁移到商密算法，并且完全基于商密算法构建出IMA的安全机制，而这些机制在以前都是完全且只能构建在国际标准的算法之上的。

{{#template ../template/footer.md}}
