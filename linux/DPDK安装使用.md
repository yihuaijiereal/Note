# DPDK的安装和使用
## 1.DPDK简介
DPDK(Data Plane Development Kit)是一个快速包处理的驱动和库的集合。
### 1.1主要的库
1）多核框架

2）huge page 内存

3）ring buffers

4）轮询模式的驱动
### 1.2使用
1）在最小的CPU周期数内接受和发送数据包

2）开发快速的包扑捉算法

3）运行第三方的快速通道
### 1.3误解
DPDK不是一个网络通信的协议栈，不提供类似第三层（网络层）的转发等功能，也没有IPSec，防火墙等功能，但是我们可以用DPDK的来轻松的开发他们。
## 2.安装准备
### 2.1huge page支持
#### 2.1.1 check
查看是否支持hugepages，运行

grep -i huge /boot/config-3.13.0-49-generic

如果出现

CONFIG_HUGETLBFS=y

CONFIG_HUGETLB_PAGE=y

则表示支持hugepages

查看系统当前的hugepages信息，运行

grep -i huge /proc/meminfo

#### 2.1.2配置hugepages（必须以root用户运行，sudo并没有用）
单节点主机运行

echo 1024 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages

支持NUMA的主机运行(运行grep -i numa /var/log/dmesg查看是否支持UNMA，如果返回No NUMA configuration found，说明不支持，也可以查看/sys/devices/system/node目录下的node数目)

echo 1024 > /sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages

echo 1024 > /sys/devices/system/node/node1/hugepages/hugepages-2048kB/nr_hugepages
#### 2.1.3使用hugepages
运行

mkdir /mnt/huge

mount -t hugetlbfs nodev /mnt/huge

是挂载点永久化，在/etc/fstab末尾加入以下行

nodev /mnt/huge hugetlbfs defaults 0 0
#### 2.1.4查看修改后的信息
运行
grep -i huge /proc/meminfo

结果如下

AnonHugePages:    286720 kB

HugePages_Total:    2048

HugePages_Free:     2048

HugePages_Rsvd:        0

HugePages_Surp:        0

Hugepagesize:       2048 kB
## 3.安装
### 3.1获取安装包
wget http://dpdk.org/browse/dpdk/snapshot/dpdk-2.2.0.tar.gz

tar -zxvf dpdk-2.2.0.tar.gz

cd dpdk-2.2.0
### 3.2安装
安装，运行

make install T=x86_64-native-linuxapp-gcc

加载模块，运行

sudo modprobe uio_pci_generic

sudo modprobe vfio-pci
### 3.3绑定网卡
查看网卡绑定，运行（如果出现OSError: [Errno 2] No such file or directory，尝试安装pciutils）

./tools/dpdk_nic_bind.py --status

得到

Network devices using DPDK-compatible driver

============================================

< none >

Network devices using kernel driver

===================================

0000:81:00.0 '82599ES 10-Gigabit SFI/SFP+ Network Connection' if=em1 drv=ixgbe unused=vfio-pci,uio_pci_generic *Active*

0000:81:00.1 '82599ES 10-Gigabit SFI/SFP+ Network Connection' if=em2 drv=ixgbe unused=vfio-pci,uio_pci_generic

Other network devices

=====================

将em2绑定到DPDK驱动，运行

sudo ./tools/dpdk_nic_bind.py --bind=uio_pci_generic em2

再运行./tools/dpdk_nic_bind.py --status查看状态得到

Network devices using DPDK-compatible driver

============================================

0000:81:00.1 '82599ES 10-Gigabit SFI/SFP+ Network Connection' drv=uio_pci_generic unused=vfio-pci

Network devices using kernel driver

===================================

0000:81:00.0 '82599ES 10-Gigabit SFI/SFP+ Network Connection' if=em1 drv=ixgbe unused=vfio-pci,uio_pci_generic *Active*

Other network devices

=====================

< none >

## 4.运行helloworld例子
cd examples/helloworld/

export RTE_SDK=$HOME/dpdk-2.2.0

export RTE_TARGET=x86_64-native-linuxapp-gcc

make

sudo ./helloworld -c 3 -n 2
