# B00-HX3-REV1.6-F

- SoC:Allwinner H618;DRAM:4 GiB DDR3.
- PCB 丝印:`B00-HX3-REV1.6-F`.
- 以太网:RTL8211F 千兆 PHY,rootfs 固定接口名 `end0` 和 MAC `02:00:04:48:99:cb`;Linux 下已建立 1000 Mb/s 全双工链路,无需混杂模式;最终检查错误为 0,有 1 个启动后 RX dropped 且 10 秒未增长.
- 存储:TF 卡和 Biwin eMMC;板子启动优先级为 TF 卡优先,eMMC 其次.
- USB 端口按 USB Host 使用;GPU 走主线 Panfrost,并已用 Mesa Rusticl 在 Mali-G31 上实机运行 OpenCL 向量加法.
- Debian 13 arm64,Linux 6.18,U-Boot 2025.04,TF-A 2.12 的当前组合已启动.

## 目录

- `kernel/`:Linux 主线设备树,内核配置和新增 DTB 目标补丁.
- `u-boot/`:本板 SPL/启动配置,U-Boot 设备树和 MMC 读取补丁.
- `rootfs-overlay/`:headless Debian 的网络,SSH 和主机名配置.
- `BUILD.md`:从固定上游版本到可写 TF/eMMC 的手工构建教程.

镜像首次从 TF 或 eMMC 启动时,会用 `cloud-guest-utils` 扩展第 2 分区,再在线扩展 ext4 rootfs;成功后服务和 helper 脚本会自行移除. TF 与 eMMC 可写入同一整盘镜像,设备容量不同也会按各自介质扩展.

## 其他

- 板子默认DHCP,无HDMI输出,root默认空密码,自行修改并做安全配置.
- 板子串口并不可用.
- 实际设备图查看`https://github.com/ophub/fnnas/issues/500`
- 实际镜像看Relase下载.