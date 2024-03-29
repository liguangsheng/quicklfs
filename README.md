### 制作内核
在kernel.org下载内核源码并解压，进入源码目录编译内核
```
# 使用环境变量声明linux内核版本
export KERNEL_VERSION=6.4.5
# 下载linux内核源代码
wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-${KERNEL_VERSION}.tar.xz
# 解压源代码
tar Jxvf linux-${KERNEL_VERSION}.tar.xz
# 使用默认配置编译内核
(cd linux-${KERNEL_VERSION} && make defconfig && make bzImage -j4)
```

编译好的内核在arch/x86_64/boot/bzImage，可以尝试qemu虚拟机启动这个内核
应该可以正常启动成功，但是最后会报错，这是正常情况

```
qemu-system-x86_64 -kernel linux-${KERNEL_VERSION}-arch/x86_64/boot/bzImage
```

### 制作initramfs
先准备一个busybox的二进制，可以直接下载编译好的二进制，也可以自己下载源码编译，编译的时候最好选择静态链接以免运行时找不到动态链接库
```
# 下载busybox
wget https://busybox.net/downloads/binaries/1.35.0-x86_64-linux-musl/busybox
# 为busybox添加可执行权限
chmod +x busybox
```

建立initramfs目录用来制作initramfs


可以选择直接把busybox直接释放到bin目录，也可以在init脚本里释放，直接释放会多占点空间，init里释放多占点时间

```
# 新建initramfs文件夹，并新建各种子文件夹
mkdir initramfs
(cd initramfs && mkdir -p bin dev etc lib mnt proc sbin sys tmp var)
# 复制busybox到initramfs里面
cp busybox initramfs/bin/
```

简简单单写个脚本到initramfs/init，用来测试一下
```
#!/bin/busybox sh

/bin/busybox --install -s /bin/
/bin/busybox --install -s /sbin/

mount -t proc proc /proc
mount -t sysfs sysfs /sys

exec /bin/sh
```

别忘了给init加上可执行权限
```
chmod +x initramfs/init
```

把initramfs制作成镜像文件
```
(cd initramfs && find . | cpio -ov --format=newc | gzip -9 > ../initramfz)
```

此时可以用qemu启动，顺利执行到init脚本进入到busybox的sh了
```
qemu-system-x86_64 -kernel linux-${KERNEL_VERSION}/arch/x86_64/boot/bzImage --initrd initramfz
```

### 制作rootfs
新建磁盘镜像
```
qemu-img create -f qcow2 lfs.qcow2 64G
```

把磁盘镜像连接到/dev/nbd0
```
modprobe nbd max_part=8
qemu-nbd --connect=/dev/nbd0 lfs.qcow2
```

磁盘分区，分了个16M作为BIOS启动，剩下的直接一个分区
```
# fdisk -l /dev/nbd0
Disk /dev/nbd0：64 GiB，68719476736 字节，134217728 个扇区
单元：扇区 / 1 * 512 = 512 字节
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节
磁盘标签类型：gpt
磁盘标识符：CE5EA8BA-9D01-7D41-8751-E92B26266EFB

设备         起点      末尾      扇区 大小 类型
/dev/nbd0p1  2048     34815     32768  16M BIOS 启动
/dev/nbd0p2 34816 134215679 134180864  64G Linux 文件系统
```

磁盘格式化
```
sudo mkfs.ext4 /dev/nbd0p2
mke2fs 1.46.5 (30-Dec-2021)
丢弃设备块： 完成
创建含有 16772608 个块（每块 4k）和 4194304 个inode的文件系统
文件系统UUID：16dcad35-f3c1-40e1-82b2-0360c2b56510
超级块的备份存储于下列块：
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
	4096000, 7962624, 11239424

正在分配组表： 完成
正在写入inode表： 完成
创建日志（65536 个块）完成
写入超级块和文件系统账户统计信息： 已完成
```

挂载磁盘
```
mkdir mnt
mount -o sync /dev/nbd0p2 mnt
sudo mkidr mnt/boot
```

完善一下initramfs里的init脚本
```
#!/bin/busybox sh

/bin/busybox --install -s /bin/
/bin/busybox --install -s /sbin/
mkdir /usr
ln -s /bin /usr/bin
ln -s /sbin /usr/sbin

# 禁止kernel的日志输出到屏幕上
echo 0 > /proc/sys/kernel/printk

# 清屏
clear

# 创建各种设备节点
mknod -m 640 /dev/console c 5 1
mknod -m 664 /dev/null c 1 3
mdev -s

# 挂载运行时需要的几个目录
mkdir -p /{dev,proc,sys,run}
mount -n -t devtmpfs devtmpfs /dev
mount -n -t proc     proc     /proc
mount -n -t sysfs    sysfs    /sys
mount -n -t tmpfs    tmpfs    /run

# 挂载磁盘和rootfs
mkdir -p /rootfs
mount /dev/sda2 /rootfs

# 尝试使用rootfs里的/sbin/init脚本作为init进程
init="/sbin/init"
if [[ -x "/rootfs/${init}" ]] ; then
    umount /sys /proc
    exec switch_root /rootfs "${init}"
fi

# 如果执行init进程失败，回退到启动busybox的sh
echo "Failed to switch_root, dropping to a shell"
exec /bin/sh
```


重新打包initramfz和之前的bzImage一起复制到mnt/boot目录里去，还有把busybox复制到rootfs里面去
```
sudo cp linux-${KERNEL_VERSION}/arch/x86_64/boot/bzImage mnt/boot/
(cd initramfs && find . | cpio -ov --format=newc | gzip -9 > ../initramfz)
sudo cp initramfz mnt/boot/

sudo mkdir mnt/bin
sudo cp busybox mnt/bin
./busybox --install -s mnt/bin
```

安装grub
```
sudo grub-install --target=i386-pc --boot-directory=./mnt/boot --root-directory=./mnt /dev/nbd0
```

简单写个配置到mnt/boot/grub2/grub.cfg
```
set default=0
set timeout=5

set root=(hd0,gpt2)

menuentry "mylinux" {
        linux   /boot/bzImage ro console=tty0 selinux=0 root=/dev/sda
	initrd	/boot/initramfz
}
```

简单写个init脚本到mnt/sbin/init
```
#!/bin/busybox sh

NC='\033[0m'
BLACK='\033[0;30m'
RED='\033[0;31m'
GREEN='\033[0;32m'
ORANGE='\033[0;33m'
BLUE='\033[0;34m'
PURPLE='\033[0;35m'
CYAN='\033[0;36m'
LIGHT_GRAY='\033[0;37m'

DARK_GRAY='\033[1;30m'
LIGHT_RED='\033[1;31m'
LIGHT_GREEN='\033[1;32m'
YELLOW='\033[1;33m'
LIGHT_BLUE='\033[1;34m'
LIGHT_PURPLE='\033[1;35m'
LIGHT_CYAN='\033[1;36m'
WHITE='\033[1;37m'

echo -e "Welcome to ${LIGHT_GREEN}My Linux From Scratch${NC}!"
echo -e "Welcome to ${LIGHT_PURPLE}My Linux From Scratch${NC}!"
echo -e "Welcome to ${LIGHT_CYAN}My Linux From Scratch${NC}!"

mkdir -p /dev /proc /sys /run
mount -n -t devtmpfs devtmpfs /dev
mount -n -t proc     proc     /proc
mount -n -t sysfs    sysfs    /sys
mount -n -t tmpfs    tmpfs    /run
mdev -s

exec /bin/busybox sh
```

别忘了给init加上可执行权限
```
sudo chmod +x mnt/sbin/init
```

这次从设备启动qemu，应该可以看到三种颜色的Welcome了，你可以从上面的颜色变量里挑你自己喜欢的颜色修改
```
sudo qemu-system-x86_64 -hda /dev/nbd0
```

这里有个`can't access tty; job control turned off`的提示，表示现在没法使用Ctrl-C停止进程和Ctrl-Z后台挂起进程，可以先不用管

取消挂载之后可以直接从qcow2镜像启动
```
sudo umount mnt
sudo qemu-nbd --disconnect /dev/nbd0
qemu-system-x86_64 -hda lfs.qcow2
```
