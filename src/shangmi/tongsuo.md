![Tongsuo](images/tongsuo.png)

铜锁/Tongsuo（原BabaSSL）是一个提供现代密码学算法和安全通信协议的开源基础密码库，为存储、网络、密钥管理、隐私计算等诸多业务场景提供底层的密码学基础能力，实现数据在传输、使用、存储等过程中的私密性、完整性和可认证性，为数据生命周期中的隐私和安全提供保护能力。

铜锁派生自OpenSSL，目前最新的版本是 8.3.2，发布于2022.12.12，除了OpenSSL常规功能外，Tongsuo 支持支持如下特有的功能特性：

* 支持RFC 8998，即在TLS 1.3中使用商用密码算法
* 支持GB/T 38636-2020 TLCP标准，即安全传输协议
* 支持QUIC API
* 支持Delegated Credentials功能，基于draft-ietf-tls-subcerts-10
* 支持RFC 8879，即证书压缩
* 支持EC-ElGamal半同态加密算法

铜锁获得了国家密码管理局商用密码检测中心颁发的商用密码产品认证证书，助力用户在国密改造、密评、等保等过程中，更加严谨地满足我国商用密码技术合规的要求。可在[此处](https://www.yuque.com/tsdoc/misc/st247r05s8b5dtct)下载资质原始文件。

<div align=center>
<span>
<img src="images/tongsuo_android.png" alt="Tongsuo" style="width: 32%; height: 32%">
</span>
<span>
<img src="images/tongsuo_ios.png" alt="Tongsuo" style="width: 32%; height: 32%">
</span>
<span>
<img src="images/tongsuo_linux.png" alt="Tongsuo" style="width: 32%; height: 32%">
</span>
</div>

## 铜锁社区

铜锁是完全以社区开发和运营的中立化开源项目，铜锁社区由三部分组成，分别是：铜锁技术委员会（Tongsuo Technical Committee, TTC），代码贡献者（Contributor）以及用户（User）。

虽然铜锁诞生于阿里系内部并率先在各项业务上得到了广泛的应用，但自从2020年10月开源之后，我们一直致力于将铜锁的产品发展的权力交给社区，因此成立了铜锁技术委员会（以下简称TTC）。

铜锁技术委员会作为铜锁开源社区的最高决策机构，采用投票的方式对铜锁相关开源项目的各项事宜进行决策。具体来说TTC有如下职责：

* 制定铜锁项目整体的发展方向和路线
* 负责铜锁项目的特性准入评审
* 负责铜锁项目的代码评审
* 负责铜锁开源社区运营过程中的相关事宜，包括但不限于宣传、推广、用户拓展等方面
* 负责决定技术委员会中新成员的加入和现有成员的移除

TTC的成员可以是个人也可以是组织。

向铜锁贡献代码需要遵守代码贡献者协议（CLA），并得到铜锁技术委员会成员的至少两个Approval后，可以合并到铜锁代码仓库。

所有使用铜锁社区中相关项目的使用者，即是铜锁的用户。铜锁的用户可以通过Github向铜锁社区进行咨询和问题反馈。

本节会重点介绍铜锁项目的特色功能。

{{#template ../template/footer.md}}
