# B00-HX3 手工构建笔记

## 上游版本

| 项目 | 版本 | 本次构建 commit |
|---|---|---|
| TF-A | v2.12 | 4ec2948fe3f65dba2f19e691e702f7de2949179c |
| U-Boot | v2025.04 | 34820924edbc4ec7803eb89d9852f4b870fa760a |
| Linux | v6.18 | 7d0a66e4bb9081d75c82ec4957c50034cb0ea449 |

官方源码:ARM-software/arm-trusted-firmware,u-boot/u-boot,torvalds/linux.

## 构建机

Debian/Ubuntu 主机安装工具:

```sh
sudo apt update
sudo apt install build-essential git bc bison flex libssl-dev libncurses-dev \
  libelf-dev libgnutls28-dev libpython3-dev python3 python3-setuptools swig \
  device-tree-compiler gcc-aarch64-linux-gnu g++-aarch64-linux-gnu \
  debootstrap qemu-user-static rsync dosfstools e2fsprogs parted util-linux \
  kmod xz-utils
```

设本目录和构建工作目录:

```sh
export BOARD=/path/to/B00-HX3
export WORK="$HOME/b00-hx3-work"
mkdir -p "$WORK"
```

## 1. 获取源码

```sh
git clone --depth 1 --branch v2.12 https://github.com/ARM-software/arm-trusted-firmware.git "$WORK/tf-a"
git clone --depth 1 --branch v2025.04 https://github.com/u-boot/u-boot.git "$WORK/u-boot"
git clone --depth 1 --branch v6.18 https://github.com/torvalds/linux.git "$WORK/linux"
```

## 2. 构建 TF-A 和 U-Boot

先构建 BL31:

```sh
cd "$WORK/tf-a"
make -j"$(nproc)" CROSS_COMPILE=aarch64-linux-gnu- PLAT=sun50i_h616 DEBUG=0 bl31
```

再应用板级 SPL/MMC 补丁并构建 U-Boot:

```sh
cd "$WORK/u-boot"
git apply "$BOARD/u-boot/patches/0001-spl-mmc-single-block-fit-reads.patch"
git apply "$BOARD/u-boot/patches/0002-tf-card-detect-no-physical-cd.patch"
install -m 0644 "$BOARD/u-boot/config/b00_hx3_rev1_6_f_defconfig" configs/b00_hx3_rev1_6_f_defconfig
install -m 0644 "$BOARD/u-boot/dts/sun50i-h618-b00-hx3-rev1-6-f.dts" dts/upstream/src/arm64/allwinner/
make O=build CROSS_COMPILE=aarch64-linux-gnu- b00_hx3_rev1_6_f_defconfig
make O=build CROSS_COMPILE=aarch64-linux-gnu- BL31="$WORK/tf-a/build/sun50i_h616/release/bl31.bin" -j"$(nproc)"
```

组合镜像为 $WORK/u-boot/build/u-boot-sunxi-with-spl.bin.SPL FIT 起始于绝对 LBA 0x60;配置强制 MMC 单块读取以避开本板 CMD18 多块读取故障.

## 3. 构建 Linux 内核和模块

```sh
cd "$WORK/linux"
git apply "$BOARD/kernel/patches/0001-add-b00-hx3-dtb-to-makefile.patch"
git apply "$BOARD/kernel/patches/0002-sun8i-use-all-multicast-when-filter-overflows.patch"
install -m 0644 "$BOARD/kernel/dts/sun50i-h618-b00-hx3-rev1-6-f.dts" arch/arm64/boot/dts/allwinner/
mkdir -p build
cp "$BOARD/kernel/config/kernel-b00-hx3.config" build/.config
make O=build ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- olddefconfig
make O=build ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j"$(nproc)" Image dtbs modules
```

产物为 $WORK/linux/build/arch/arm64/boot/Image 和 dts/allwinner/sun50i-h618-b00-hx3-rev1-6-f.dtb.

配置必须包含 `CONFIG_SUN50I_H6_PRCM_PPU=y`:H616 GPU 的电源域由该驱动提供;漏掉它会使 Panfrost 因依赖未就绪而探测失败.以太网补丁在多播地址超过硬件精确过滤槽时改用 all-multicast 过滤,避免把整块网卡切到混杂接收.

## 4. 准备 headless Debian rootfs

```sh
sudo debootstrap --arch=arm64 --variant=minbase --foreign --include=ca-certificates trixie "$WORK/rootfs" https://deb.debian.org/debian
sudo cp /usr/bin/qemu-aarch64-static "$WORK/rootfs/usr/bin/"
sudo mount -t proc proc "$WORK/rootfs/proc"
sudo mount -t sysfs sys "$WORK/rootfs/sys"
sudo mount --bind /dev "$WORK/rootfs/dev"
sudo mount -t devpts devpts "$WORK/rootfs/dev/pts"
sudo chroot "$WORK/rootfs" /debootstrap/debootstrap --second-stage
sudo chroot "$WORK/rootfs" /bin/bash
```

在 chroot 中安装本次软件包:

```sh
apt-get update
apt-get install systemd systemd-sysv systemd-resolved systemd-timesyncd udev kmod \
  openssh-server iproute2 ethtool sudo vim-tiny e2fsprogs dosfstools \
  cloud-guest-utils locales fio mmc-utils memtester mesa-utils libgl1-mesa-dri \
  mesa-opencl-icd clinfo
exit
```

回到构建机,应用本板 rootfs 覆盖文件并启用 DHCP,SSH host key 和扩容服务:

```sh
sudo cp -a "$BOARD/rootfs-overlay/." "$WORK/rootfs/"
sudo install -m 0644 "$BOARD/rootfs-overlay/etc/fstab.template" "$WORK/rootfs/etc/fstab"
sudo chroot "$WORK/rootfs" usermod -p '' root
sudo truncate -s 0 "$WORK/rootfs/etc/machine-id"
sudo chroot "$WORK/rootfs" systemctl enable systemd-networkd.service systemd-resolved.service \
  systemd-timesyncd.service ssh.service ssh-hostkeys.service h618-expand-rootfs.service
sudo rm -f "$WORK/rootfs"/etc/ssh/ssh_host_*
```

安装内核模块并生成依赖索引:

```sh
make -C "$WORK/linux" O=build ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- \
  INSTALL_MOD_PATH="$WORK/rootfs" INSTALL_MOD_STRIP=1 modules_install
sudo depmod -b "$WORK/rootfs" "$(make -s -C "$WORK/linux" O=build ARCH=arm64 kernelrelease)"
```

rootfs 配置允许 root 空密码 SSH 登录,仅用于 headless 首次接入.设备启动后应设置密码并关闭空密码认证;镜像不应打包个人 SSH 公钥或固定 SSH host key.

## 5. 手工制作可写 TF/eMMC 的整盘镜像

以下命令在构建机 root shell 中手工执行.WORK 中 rootfs,U-Boot 和内核产物路径应与上文一致.分区表为 DOS/MBR:p1 从 16 MiB 开始,大小 256 MiB FAT32;p2 从 272 MiB 开始,占余下空间 ext4;U-Boot 写在偏移 8 KiB.

```sh
ROOTFS_MB=$(du -sm "$WORK/rootfs" | cut -f1)
ROOT_PART_MB=$((ROOTFS_MB * 2 > 2048 ? ROOTFS_MB * 2 : 2048))
IMG_MB=$((16 + 256 + ROOT_PART_MB + 8))
IMG="$WORK/b00-hx3-debian-trixie.img"
truncate -s "${IMG_MB}M" "$IMG"
parted -s "$IMG" mklabel msdos
parted -s "$IMG" unit MiB mkpart primary fat32 16 272
parted -s "$IMG" unit MiB mkpart primary ext4 272 100%
parted -s "$IMG" set 1 boot on
LOOP=$(losetup -Pf --show "$IMG")
partx -a "$LOOP"
udevadm settle --timeout=10
mkfs.vfat -F 32 -n BOOT "${LOOP}p1"
mkfs.ext4 -F -L rootfs "${LOOP}p2"
mkdir -p "$WORK/mnt-root" "$WORK/mnt-boot"
mount "${LOOP}p2" "$WORK/mnt-root"
mount "${LOOP}p1" "$WORK/mnt-boot"
rsync -aHAX --numeric-ids "$WORK/rootfs/" "$WORK/mnt-root/"
install -m 0644 "$WORK/linux/build/arch/arm64/boot/Image" "$WORK/mnt-boot/Image"
install -m 0644 "$WORK/linux/build/arch/arm64/boot/dts/allwinner/sun50i-h618-b00-hx3-rev1-6-f.dtb" "$WORK/mnt-boot/"
mkdir -p "$WORK/mnt-boot/extlinux"
ROOT_PARTUUID=$(blkid -s PARTUUID -o value "${LOOP}p2")
BOOT_PARTUUID=$(blkid -s PARTUUID -o value "${LOOP}p1")
sed -i "s/__ROOT_PARTUUID__/$ROOT_PARTUUID/; s/__BOOT_PARTUUID__/$BOOT_PARTUUID/" "$WORK/mnt-root/etc/fstab"
cat > "$WORK/mnt-boot/extlinux/extlinux.conf" <<EOF
default debian
prompt 0

label debian
    linux /Image
    fdt /sun50i-h618-b00-hx3-rev1-6-f.dtb
    append root=PARTUUID=$ROOT_PARTUUID rootwait rw console=ttyS0,115200n8 earlycon=uart8250,mmio32,0x05000000
EOF
dd if="$WORK/u-boot/build/u-boot-sunxi-with-spl.bin" of="$LOOP" bs=1024 seek=8 conv=notrunc,fsync
sync
umount "$WORK/mnt-boot" "$WORK/mnt-root"
losetup -d "$LOOP"
sha256sum "$IMG"
```

