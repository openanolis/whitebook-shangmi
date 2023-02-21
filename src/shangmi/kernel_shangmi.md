# 内核国密算法

Linux 内核上游社区从 5.10 版本开始已经陆续支持了国密算法，到目前为止，x86和arm64架构上的国密优化也支持的比较完善。

龙蜥社区 ANCK 5.10 内核已经全部支持了上游的这些国密和算法优化，通过 Anolis OS 提供给用户。

以下几张表格总结了目前内核已经实现的国密相关算法的一些具体情况，包括算法优先级（优先级越大性能越高，同一个算法内核会优先选择优先级高的实现），依赖指令（具体架构上依赖的CPU SIMD指令），内部驱动（可以认为是更详细的算法名称，可以通过这个名称引用该实现）。

类型字段是算法的类型，目前跟国密相关的几个类型如下：

* akcipher: 非对称算法
* shash: 哈希算法
* cipher: 对称算法，单个分组的加解密
* skcipher: 对称算法，与具体模式相结合
* aead: 对称算法，带认证

> 🟢 **软件实现**

国密的软件实现是最早被引用内核的国密算法实现，软件实现不依赖任何特殊指令，适用于任何架构，但效率较低，在不支持优化的平台上可以选择使用软件实现的国密方案。

SM2 是非对称算法，目前在内核中主要用于验签和完整性检查，由于算法自身输入数据量小且调用频度不高，优化带来的收益不大，因此，Linux 内核中的SM2算法目前只有软件实现。

| 算法 | 类型     | 内部驱动    | 优先级 | 模块名      |
| ---- | -------- | ----------- | -----: | ----------- |
| SM2  | akcipher | sm2-generic |    100 | sm2-generic |
| SM3  | shash    | sm3-generic |    100 | sm3-generic |
| SM4  | cipher   | sm4-generic |    100 | sm4-generic |

> 🟢 **x86架构指令集优化**

在x86架构上，主要是使用 AVX/AVX2 指令集对国密算法做的优化。

| 算法    | 类型     | 内部驱动           | 优先级 | 模块名                | 依赖指令   |
| ------- | -------- | ------------------ | -----: | --------------------- | ---------- |
| SM3     | shash    | sm3-avx            |    300 | sm3-avx-x86_64        | avx/bmi2   |
| SM4-ECB | skcipher | ecb-sm4-aesni-avx  |    400 | sm4-aesni-avx-x86_64  | avx/aesni  |
| SM4-CBC | skcipher | cbc-sm4-aesni-avx  |    400 | sm4-aesni-avx-x86_64  | avx/aesni  |
| SM4-CFB | skcipher | cfb-sm4-aesni-avx  |    400 | sm4-aesni-avx-x86_64  | avx/aesni  |
| SM4-CTR | skcipher | ctr-sm4-aesni-avx  |    400 | sm4-aesni-avx-x86_64  | avx/aesni  |
| SM4-ECB | skcipher | ecb-sm4-aesni-avx2 |    500 | sm4-aesni-avx2-x86_64 | avx2/aesni |
| SM4-CBC | skcipher | cbc-sm4-aesni-avx2 |    500 | sm4-aesni-avx2-x86_64 | avx2/aesni |
| SM4-CFB | skcipher | cfb-sm4-aesni-avx2 |    500 | sm4-aesni-avx2-x86_64 | avx2/aesni |
| SM4-CTR | skcipher | ctr-sm4-aesni-avx2 |    500 | sm4-aesni-avx2-x86_64 | avx2/aesni |

> 🟢 **arm64架构指令集优化**

arm64架构上的国密优化最完整，效果也最明显，比如SM4算法，除了x86架构上的四个模式外，还对 CTS/XTS 模式，AEAD模式 CCM/GCM 以及带密钥的哈希算法做了深度优化，这主要得益于armv8开始支持了SM3/SM4算法的Crypto Extensions扩展。

| 算法    | 类型     | 内部驱动       | 优先级 | 模块名        | 依赖指令   |
| ------- | -------- | -------------- | -----: | ------------- | ---------- |
| SM3     | shash    | sm3-neon       |    200 | sm3-neon      | NEON       |
| SM3     | shash    | sm3-ce         |    400 | sm3-ce        | CE-SM3     |
| SM4     | cipher   | sm4-ce         |    300 | sm4-ce-cipher | CE-SM4     |
| SM4-ECB | skcipher | ecb-sm4-neon   |    200 | sm4-neon      | NEON       |
| SM4-CBC | skcipher | cbc-sm4-neon   |    200 | sm4-neon      | NEON       |
| SM4-CFB | skcipher | cfb-sm4-neon   |    200 | sm4-neon      | NEON       |
| SM4-CTR | skcipher | ctr-sm4-neon   |    200 | sm4-neon      | NEON       |
| SM4-ECB | skcipher | ecb-sm4-ce     |    400 | sm4-ce        | CE-SM4     |
| SM4-CBC | skcipher | cbc-sm4-ce     |    400 | sm4-ce        | CE-SM4     |
| SM4-CFB | skcipher | cfb-sm4-ce     |    400 | sm4-ce        | CE-SM4     |
| SM4-CTR | skcipher | ctr-sm4-ce     |    400 | sm4-ce        | CE-SM4     |
| SM4-CTS | skcipher | cts-cbc-sm4-ce |    400 | sm4-ce        | CE-SM4     |
| SM4-XTS | skcipher | xts-sm4-ce     |    400 | sm4-ce        | CE-SM4     |
| CMAC-SM4   | shash | cmac-sm4-ce    |    400 | sm4-ce        | CE-SM4     |
| XCBC-SM4   | shash | xcbc-sm4-ce    |    400 | sm4-ce        | CE-SM4     |
| CBCMAC-SM4 | shash | cbcmac-sm4-ce  |    400 | sm4-ce        | CE-SM4     |
| SM4-CCM | aead     | ccm-sm4-ce     |    400 | sm4-ce-ccm    | CE-SM4     |
| SM4-GCM | aead     | gcm-sm4-ce     |    400 | sm4-ce-gcm  | CE-SM4/PMULL |

{{#template ../template/footer.md}}
