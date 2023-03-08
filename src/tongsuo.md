<div align=center>
<img src="images/tongsuo.png" alt="Tongsuo" style="width: 70%; height: 70%">
</div>

铜锁/Tongsuo是一个提供现代密码学算法和安全通信协议的开源基础密码库，为存储、网络、密钥管理、隐私计算等诸多业务场景提供底层的密码学基础能力，实现数据在传输、使用、存储等过程中的私密性、完整性和可认证性，为数据生命周期中的隐私和安全提供保护能力。

铜锁的主要功能特性有：

  * 技术合规能力
    * 符合GM/T 0028《密码模块安全技术要求》的"软件密码模块安全一级"资质
  * 零知识证明（ZKP）
    * Bulletproofs (Range)
  * 密码学算法
    * 中国商用密码算法：SM2、SM3、SM4、[祖冲之](https://www.yuque.com/tsdoc/ts/copzp3)等
    * 国际主流算法：ECDSA、RSA、AES、SHA等
    * 同态加密算法：[EC-ElGamal](https://www.yuque.com/tsdoc/misc/ec-elgamal)、[Paillier](https://www.yuque.com/tsdoc/misc/rdibad)等
  * 安全通信协议
    * 支持GB/T 38636-2020 TLCP标准，即[双证书国密](https://www.yuque.com/tsdoc/ts/hedgqf)通信协议
    * 支持[RFC 8998](https://datatracker.ietf.org/doc/html/rfc8998)，即TLS 1.3 +[国密单证书](https://www.yuque.com/tsdoc/ts/grur3x)
    * 支持[QUIC](https://datatracker.ietf.org/doc/html/rfc9000) API
    * 支持[Delegated Credentials](https://www.yuque.com/tsdoc/ts/leubbg)功能，基于[draft-ietf-tls-subcerts-10](https://www.ietf.org/archive/id/draft-ietf-tls-subcerts-10.txt)
    * 支持[TLS证书压缩](https://www.yuque.com/tsdoc/ts/df5pyi)


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

{{#template template/footer.md}}
