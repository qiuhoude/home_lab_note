# home_lab_note
折腾openwrt , nas笔记

## openwrt docker折腾
现在的需求是根据别人编译好的openwrt固件,img文件制作成docker镜像, 比如eSir的固件

#### 将img转成 openwrt系统文件的压缩包
```bash
fdisk openwrt.img
# 1. 查看镜像分区,使用fdisk工具
# 进入fdisk交互界面
Command (m for help): p
Disk openwrt.img: 516.51 MiB, 541589504 bytes, 1057792 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x145b0744

Device       Boot Start     End Sectors  Size Id Type
openwrt.img1 *      512   33279   32768   16M 83 Linux
openwrt.img2      33792 1057791 1024000  500M 83 Linux
# 可以查看有两个分区,第二个分区才是openwrt的内容,开始位置是 33792
# 扇区的单位是   Units: sectors of 1 * 512 = 512 bytes ,后面挂在需要用到,

# 2. 挂载img文件
sudo mount -o loop,offset=$((33792*512)) openwrt.img /mnt/openwrt

# 3. 将挂在文件夹进行打包
cd /mnt/openwrt
sudo tar -czvf ~/opewrt/openwrt.tar.gz ./*
# 打包完卸载/mnt/openwrt
sudo unmount /mnt/openwrt
```

#### 制作openwrt docker镜像
准备Dockerfile文件

```Dockerfile
FROM scratch
ADD openwrt.tar.gz /
ENTRYPOINT ["/sbin/init"]
```

```bash
# 构建 Dockerfile,openwrt.tar.gz在同一个目录下
docker build . -t houde/openwrt
# 重新制作可以删除掉原来dangling镜像
docker rmi $(docker images | grep "none" | awk '{print $3}')
```

#### 以旁路由的模式运行openwrt
```bash
#  https://docs.docker.com/network/macvlan/ macvlan文档
#  添加macvlan网络, 绑定到eth0上
docker network create -d macvlan --subnet=192.168.10.0/24 --gateway=192.168.10.1 -o parent=eth0 openwrtvlan
docker run --restart unless-stopped -d --name=OpenWRT --network openwrtvlan --privileged=true houde/openwrt
```
