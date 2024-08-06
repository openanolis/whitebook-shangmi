# 密码加速器 - 海光密码技术（HCT）

## 简介

海光密码技术HCT（Hygon Cryptographic Technology）是基于海光芯片密码协处理器以及密码指令集特性，自主设计研发的一套密码算法加速软件开发套件。

官方发布的 HCT 密码计算套件位于 gitee 的 `hygon-devkit` 仓库，地址：<https://gitee.com/anolis/hygon-devkit.git>

```sh
git clone https://gitee.com/anolis/hygon-devkit.git
```

HCT密码计算套件的目录结构示意图如下：

```
hygon-devkit/
  ├─ hct
  ├──pkg
  │  ├── hct_x.x.x.xxxxxxxx_alpha
  │  └── hct_x.x.x.xxxxxxxx_release
  ├── readme
  ├── sample
  └── script
```

* pkg目录：内含各版本hct密码计算套件（上述示意图中的x.x.x.xxxxxxxx代表版本信息）。
* sample目录：内含使用hct密码套件的示例程序代码。
* script目录：内含一些工具脚本。
* readme文件：有关HCT的简单情况及使用说明。

## 测试与开发

### 环境配置

安装支持HCT的内核，参考[安装开源OS镜像并编译软件页面](https://conf.hygon.cn/pages/viewpage.action?pageId=76487872 )进行内核的编译与安装。完成内核安装后，重启系统，选择刚才安装的内核启动。

按如下流程必要的模块跟依赖：

```sh
# 1. 检测并安装必须的内核模块 
modprobe vfio vfio-pci vfio_iommu_type1 mdev vfio_mdev

# 2. 安装需要依赖的软件库

# numa 库
yum install numactl -y

# openssl 库
wget https://www.openssl.org/source/old/1.1.1/openssl-1.1.1c.tar.gz
tar –zxvf openssl-1.1.1c.tar.gz
cd openssl-1.1.1c
./config
make
sudo make install

# Tips：HCT套件中的部分工具脚本需要用到uuidgen工具，安装方式如下。
yum install util-linux -y

# 3. 安装HCT开发套件

cd hct/pkg/hct_1.0.0.20230224_rc
./config 
sudo make install
```

### 测试

* 功能测试

  ```sh
  cd /opt/hygon/hct/bin/hct/test
  ./hct_test
  ```

* 性能测试

  ```sh
  cd /opt/hygon/hct/bin
  ./hct_speed -elapsed -engine hct –multi 128 –seconds 60 sm2enc
  ./hct_speed -elapsed -engine hct –multi 128 –seconds 60 sm2sign
  ./hct_speed -elapsed -engine hct –multi 128 –seconds 60 –bytes 1024 -evp sm3
  ./hct_speed -elapsed -engine hct –multi 128 –seconds 60 –bytes 1024 -evp sm4
  ```

## hctconfig 服务命令

HCT套件成功安装后，会在系统中增加一个hctconfig服务命令。通过hctconfig服务可以方便的管理CCP协处理器的绑定情况：

1. 使用默认配置，将 CCP 协处理器绑定至最佳性能状态：`service hctconfig start`
2. 利用status参数查看HCT的相关状态信息及CCP协处理的绑定情况：`service hctconfig status`
3. 利用rebond命令对CCP协处理器进行重新绑定：`service hctconfig rebond`

***Tips：start命令和rebond的命令都可以带参数，详情可参考 usage 命令***：`service hctconfig usage`

## 开发

HCT通过 openssl 标准接口（EVP）向应用提供接口，使用HCT套件时，通过e=ENGINE_by_id（"hct"）选择密码引擎为hct，然后直接通过openssl标准（EVP）接口即可完成对hct密码计算套件的调用。

## 卸载HCT套件

在对应HCT版本包内执行make uninstall即可卸载HCT套件

```sh
hct/pkg/hct_1.0.0.20230224_rc
make uninstall
```

## 其它注意事项

如果BIOS支持，可以将PSP CCP配置给C86使用，BIOS设置步骤如下：

1. 进入BIOS，选择“HYGON CBS”，进入“Moksha Common Options”
2. 在“Available PSP CCP VQ Count”中输入4

**说明：海光CPU芯片中包含两类CCP协处理器，一类为PSP CCP，一类为NTB CCP，通常BIOS只会将NTB CCP配置给用户使用，PSP CCP一般用于可信计算场景。如果BIOS支持将PSP CCP配置给用户（C86）使用，那么HCT就可以获得更多的协处理器资源，表现出更高的性能。**

hct的正确执行依赖内核的iommu功能，所以需要使能内核的iommu功能

hygon CPU上启动内核的命令行参数参考如下：

```
amd_iommu=on iommu=pt
```

{{#template template/footer.md}}
