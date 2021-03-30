# KVM使用virt-install创建虚拟机

#### 安装前准备

依赖包安装

权限调整

重启libvirtd服务

镜像文件下载

安装虚拟机

出现的问题和解决

附录：virt-install参数说明

主机系统：CentOS7.6

#### 依赖包安装

```
[root@Dell ~]# yum install -y qemu-kvm libvirt virt-install bridge-utils
```



#### 权限调整

将user和group前面的#去掉，让root用户可以操作

```
[root@Dell ~]# vim /etc/libvirt/qemu.conf

user = "root"

group = "root"
```

#### 重启libvirtd服务

```
[root@Dell ~]# systemctl daemon-reload    //重载配置

[root@Dell ~]# systemctl restart libvirtd      //重启libvirtd服务

[root@Dell ~]# systemctl status libvirtd      //查看libvirtd服务状态
```

#### 镜像文件下载

```
[root@Dell ~]# mkdir images
[root@Dell ~]# cd images/
https://mirrors.tuna.tsinghua.edu.cn/centos/7/isos/x86_64/
下载minimal镜像
```

#### 网桥

使用brctl查看网桥信息，没有则创建

#### 安装虚拟机

本文将用于安装的镜像文件下载到/root/images文件夹中，虚拟机磁盘文件放于/root下。

```
[root@Dell ~]# virt-install \
--connect qemu:///system \
--virt-type=kvm \
--name=test1 \
--vcpus=2 \
--memory=4096 \
--location=/root/images/CentOS-7-x86_64-Minimal-1810.iso \
--disk path=/root/test1.qcow2,size=10,format=qcow2 \
--network bridge=br0 \
--graphics none \
--extra-args='console=ttyS0'
```

ttyS0，注意'S'大写

正常运行之后，就会进入命令行安装模式，跟装普通的CentOS系统一样的操作，这里就不赘述了，打感叹号！的是必须选择的，打×的是已经选好了的。

需要配置IP 时区 等信息后进行安装，安装成功后可修改常用优化参数，装机愉快！

时间有点长，请耐心等待！！！

最后附上安装完成的显示：

CentOS Linux 7 (Core)

Kernel 3.10.0-957.el7.x86_64 on an x86_64

localhost login: root

Password: 

[root@localhost ~]# 

##### 设置ip

```
cd /etc/sysconfig/network-scripts/

vim ifcfg-eth0
```

##### 初始化系统

```
cat >> /etc/security/limits.conf <<EOF
*      soft    nofile  65535
*      hard    nofile  65535
*      soft    nproc   65535
*      hard    nproc   65535
EOF

cat >> /etc/systemd/system.conf <<EOF
DefaultLimitNOFILE=65535
DefaultLimitNPROC=65535
EOF

sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
setenforce 0

systemctl stop firewalld
systemctl disable firewalld
```

##### 配置阿里云仓库

```
curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
curl -o /etc/yum.repos.d/epel.repo https://mirrors.aliyun.com/repo/epel-7.repo
```

##### 安装基础包

```
yum -y install vim wget git lrzsz net-tools tree nmap dos2unix nc lsof tcpdump htop iftop iotop sysstat nethogs gcc gcc-c++

vim:
wget: 
git:
lrzsz:
rz: 从windows传到linux)
sz + 文件名 : 从linux传到windows
net-tools: netstat
tree: tree以树形结构显示文件和目录
nmap: nmap扫描端口的工具
dos2unix: 转换脚本格式的工具
nc: 文件传输,端口检查
lsof: 反查端口进程,以及服务开发文件工具
tcpdump: 抓包,监听等重要排错工具
htop: 系统进程相关信息查看工具
iftop: 查看主机网卡带宽工具
iotop:
sysstat: 含有sar,iostat等重要系统性能查看工具
nethogs: 显示进程的网络流量
```

安装完成之后，关闭虚拟机，可以将此虚拟机作为模板

将该主机关机，并将硬盘文件/root/test1.qcow2拷贝到/home/vms/template/centos7作为模板备用。

#### 批量生成主机

此处生成172.22.3.171-177 共7台主机，171作为安装机。172-174作为master节点+node节点+etcd节点。175-177为node节点。

```shell
cp /home/vms/template/centos7 /home/vms/172.22.3.171

virt-install --connect qemu:///system --name 172.22.3.171 --ram 4096 --vcpu=1 --virt-type kvm --disk path=/home/vms/172.22.3.171,bus=virtio --network bridge=br0,model=virtio --import --graphics none   
#--graphics none参数一定要有，否则会在启动阶段退出
```

参考上述命令生成其他六台虚拟机。

```
cp /home/vms/template/centos7 /home/vms/172.22.3.172
...
```



### 常用命令

```
# virsh --help                                     #查看命令帮忙
# virsh list                                       #显示正在运行的虚拟机
# virsh list --all                                 #显示所有的虚拟机
# virsh start vm-node1                             #启动vm-node1虚拟机
# virsh shutdown vm-node1                          #关闭vm-node1虚拟机
# virsh destroy vm-node1                           #虚拟机vm-node1强制断电
# virsh suspend vm-node1                           #挂起vm-node1虚拟机
# virsh resume vm-node1                            #恢复挂起的虚拟机
# virsh undefine vm-node1                          #删除虚拟机，慎用
# virsh dominfo vm-node1                           #查看虚拟机的配置信息
# virsh domiflist                                  #查看网卡配置信息
# virsh domblklist vm-node1                        #查看该虚拟机的磁盘位置
# virsh edit vm-node1                              #修改vm-node1的xml配置文件
# virsh dumpxml vm-node1                           #查看KVM虚拟机当前配置
# virsh dumpxml vm-node1 > vm-node1.bak.xml        #备份vm-node1虚拟机的xml文件，原文件默认路径/etc/libvirt/qemu/vm-node1.xml
# virsh autostart vm-node1                         #KVM物理机开机自启动虚拟机，配置后会在此目录生成配置文件/etc/libvirt/qemu/autostart/vm-node1.xml
# virsh autostart --disable vm-node1               #取消开机自启动
```



### 给虚拟机添加磁盘

##### 虚拟磁盘

```
qemu-img create -f qcow2 /var/lib/libvirt/images/server-vdb.qcow2 50G 创建50G磁盘文件
```



使用` virsh edit <vm_name>` 来加载磁盘，，添加如下内容

```
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2' cache='none'/>
      <source file='/var/lib/libvirt/images/server-vdb.qcow2'/>
      <target dev='vdb' bus='virtio'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x08' function='0x0'/>
    </disk>
```

保持退出

##### 物理磁盘

使用 virsh edit <vm_name> 来加载磁盘，添加如下内容

```
   <disk type='block' device='disk'>
      <driver name='qemu' type='raw' cache='none' io='native'/>
      <source dev='/dev/sdb1'/>
      <target dev='sda' bus='sata'/>
      <address type='drive' controller='0' bus='0' target='0' unit='0'/>
    </disk>
```

#### 验证

```
virsh shutdown vm_name
virsh start vm_name
```

进入虚拟机，注意：没有物理磁盘，物理磁盘只是举例

```
[root@localhost ~]# lsblk
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
vda    253:0    0  10G  0 disk 
├─vda1 253:1    0   1G  0 part /boot
├─vda2 253:2    0   1G  0 part [SWAP]
└─vda3 253:3    0   8G  0 part /
vdb    253:16   0  50G  0 disk 
```

