# SPDK安装
## 1.需求
### 1.1依赖
Fodera/Centos:

    sudo dnf install -y gcc libpciaccess-devel CUnit-devel libaio-devel
dnf是新一代的包管理器，已经作为Fodera22的默认包管理器，Centos上可以安装与yum共存。

    yum install epel-release
    yum install dnf

Ubuntu/Debian:

    sudo apt-get install -y gcc libpciaccess-dev make libcunit1-dev libaio-dev
### 1.2安装DPDK
参考
  [DPDK安装](https://github.com/yihuaijiereal/Note/blob/master/linux/DPDK%E5%AE%89%E8%A3%85%E4%BD%BF%E7%94%A8.md)

# 2.安装
获取SPDK：

    wget https://github.com/spdk/spdk
安装：

    cd spdk
    make DPDK_DIR=~/dpdk-2.2.0/x86_64-native-linuxapp-gcc
# 3.绑定
需要将I/OAT和NVMe从内核态绑定到用户态，并分配hugepages

    sudo scripts/configure_hugepages.sh
    sudo scripts/setup.sh
