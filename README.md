# ImmortalWrt 云编译 · CMCC RAX3000M NAND

基于 [padavanonly/immortalwrt-mt798x-6.6](https://github.com/padavanonly/immortalwrt-mt798x-6.6) 的 **CMCC RAX3000M NAND 版**固件云编译工程(GitHub Actions),内置 MTK **闭源高功率 mt_wifi 驱动**,产物兼容设备内置的**旧版 bl-mt798x U-Boot**。

- 源码基线:ImmortalWrt v24.10 + 内核 6.6(分支 `openwrt-24.10-6.6`)
- 产物格式:`squashfs-sysupgrade.bin`(tar 格式,≈36MB)
- 实测 SHA256:`c9800d2c2e9585e914a05ea5bb8ce56d42aaefb88a01bd4d3ed2f17faa3e40e3`

---

## 一、固件特性

| 项目 | 说明 |
|---|---|
| 无线驱动 | MTK 闭源 `kmod-mt_wifi` + `kmod-warp` + `kmod-mediatek_hnat`(高功率) |
| 硬件加速 | TurboACC MTK 版(HW NAT / SFE 开关) |
| 镜像格式 | tar `sysupgrade.bin`(legacy bl-mt798x U-Boot 兼容,**与官方 ubootmod 不通用**) |
| U-Boot | 无需更换,直接 LuCI/命令行刷入;另附 `nand-preloader.bin` / `nand-bl31-uboot.fip` 备用 |
| NAND 分区 | bl2(0x0) / u-boot-env(0x100000) / factory(0x180000) / fip(0x380000) / ubi(0x580000~) |

### 内置应用(全部带中文)

| 类别 | 应用 |
|---|---|
| 管理基础 | LuCI 全套、firewall、package-manager、ttyd(系统菜单)、argon 主题 + argon-config |
| MTK 特色 | mtwifi-cfg(无线配置)、turboacc-mtk(硬件加速)、eqos-mtk(网速限制) |
| 科学上网 | openclash(自动换装 dnsmasq-full + kmod-tun) |
| 应用商店 | iStore(luci-app-store + taskd 全家) |
| 系统工具 | diskman(磁盘管理,系统菜单)、upnp、ddns、watchcat(定时重启)、wolplus(远程唤醒)、usb-printer(USB 打印)、cifs-mount(挂载 NAS/Samba,含 cifsmount 用户态工具)、vlmcsd(KMS)、wrtbwmon(流量统计) |

---

## 二、目录结构

```
.
├── .github/workflows/build.yml          # GitHub Actions 云编译流水线
├── config/
│   └── cmcc_rax3000m.seed.config        # 设备配置种子(仅 cmcc_rax3000m 单设备)
├── patches/
│   ├── 001-cmcc_rax3000m-legacy-uboot-image.patch  # 镜像格式改造(核心)
│   └── upstream-commit.txt              # 锚定的上游源码 commit
└── packages/                            # 第三方包源码(随仓库携带,保证可复现)
    ├── leigod-acc / luci-app-leigod-acc # 雷神加速器
    ├── luci-app-store / taskd / luci-lib-taskd / luci-lib-xterm  # iStore 全家
    └── luci-app-wolplus                 # WOL 增强版
```

### 核心补丁说明(`001-cmcc_rax3000m-legacy-uboot-image.patch`)

padavanonly 上游的 `cmcc_rax3000m` 默认输出 `sysupgrade.itb`(FIT 外置 rootfs 格式),适配 OpenWrt 官方 ubootmod U-Boot;而 RAX3000M NAND 出厂内置的旧版 bl-mt798x U-Boot 只认 **tar 格式 sysupgrade.bin**。本补丁在 `Device/cmcc_rax3000m` 定义内覆盖镜像生成链:

```make
KERNEL = kernel-bin | lzma | fit lzma $$(KDIR)/image-$$(firstword $$(DEVICE_DTS)).dtb
IMAGES := sysupgrade.bin
IMAGE/sysupgrade.bin := sysupgrade-tar | append-metadata
```

即:内核(LZMA 压缩 FIT)+ rootfs 以 UBI 卷形式打进 tar 包,与设备分区表完全吻合,刷机不换 U-Boot、不动分区。

---

## 三、使用方法

### 云编译(GitHub Actions)

1. Fork 或使用本仓库
2. **Actions → Build RAX3000M NAND → Run workflow**
   - `upload_release` 选 `true` 可同时发布到 Releases
3. 等待约 2~3 小时(首次全量),产物在 Artifact / Release 下载

配置变更:直接修改 `config/cmcc_rax3000m.seed.config`(make menuconfig 本地导出或手改),push 后自动触发编译。

### 本地编译(Ubuntu,WSL2 实测可用)

```bash
git clone --depth 1 -b openwrt-24.10-6.6 https://github.com/padavanonly/immortalwrt-mt798x-6.6
cd immortalwrt-mt798x

# 应用镜像补丁 + 第三方包
git apply 001-cmcc_rax3000m-legacy-uboot-image.patch
cp -r /path/to/packages/* package/custom/

# feeds + 配置
cat feeds.conf.default > feeds.conf
./scripts/feeds update -a && ./scripts/feeds install -a
rm -rf package/feeds/packages/perl   # 见「已知问题」
cp cmcc_rax3000m.seed.config .config && make defconfig

# 编译
make download -j$(nproc)
make -j$(nproc) || make -j1 V=s
```

### 刷机(RAX3000M NAND,旧 U-Boot)

1. LuCI → 系统 → 备份/升级 → 刷写新固件,选 `*squashfs-sysupgrade.bin`
2. **不勾选「保留配置」**(跨固件升级)
3. 等待 3~5 分钟自动重启,地址恢复 `192.168.1.1`
4. 命令行方式:`sysupgrade -v /tmp/*.bin`
5. 救砖:拔电按住 reset 上电,旧 U-Boot TFTP 恢复模式可用

---

## 四、引用与来源

| 来源 | 用途 | 许可 |
|---|---|---|
| [padavanonly/immortalwrt-mt798x-6.6](https://github.com/padavanonly/immortalwrt-mt798x-6.6) | 基础源码(ImmortalWrt 24.10 + 内核 6.6 + MTK 闭源 mt_wifi) | GPL-2.0 |
| [shiyu1314](https://github.com/shiyu1314) 的 immortalwrt-mt798x 分支 | legacy U-Boot 镜像方案(`sysupgrade-tar` 生成链)参考 | GPL-2.0 |
| [linkease/istore](https://github.com/linkease/istore) | iStore 应用商店(`packages/` 内 4 组件) | GPL-3.0 |
| [sundaqiang/openwrt-packages](https://github.com/sundaqiang/openwrt-packages) | luci-app-wolplus | GPL-2.0 |
| [miaoermua/luci-app-leigod-acc](https://github.com/miaoermua/luci-app-leigod-acc) | 雷神加速器 | GPL-2.0 |
| [vernesong/OpenClash](https://github.com/vernesong/OpenClash) | openclash(来自 immortalwrt luci feed) | GPL-3.0 |
| [ImmortalWrt](https://github.com/immortalwrt/immortalwrt) / [OpenWrt](https://github.com/openwrt/openwrt) | 上游发行版 | GPL-2.0 |

本文档与工程由编译实测过程整理;所有第三方包版权归原作者所有,遵循原许可协议分发。

---

## 五、已知问题与注意事项

1. **perl 必须剥离**:`feeds/packages/lang/perl` 在本配置下无任何包依赖,但其交叉构建(miniperl 自检 `File/Glob.pm` 失败)必然报错并阻断 `make`。workflow 与本地步骤均已移除。
2. **不要用 Ubuntu 25.x(uutils coreutils)做编译机**:`/usr/bin/install` 是 Rust 实现,OpenWrt prereq 检测不认。修复:`sudo apt install --allow-remove-essential coreutils-from-gnu coreutils-from-uutils-`。
3. **openclash 会自动将 dnsmasq 替换为 dnsmasq-full**(kconfig 冲突处理),属正常现象。
4. **WSL2 编译保活**:长编译需配合 Windows 计划任务定期执行 `wsl` 命令防止 distro 空闲回收。
5. 镜像格式与官方 `ubootmod` 方案 **互不兼容**,刷错不会启动;老 U-Boot 设备用本工程产物即可。

---

## 六、编译产物一览

| 文件 | 说明 |
|---|---|
| `immortalwrt-mediatek-filogic-cmcc_rax3000m-squashfs-sysupgrade.bin` | 主固件(36MB,刷这个) |
| `immortalwrt-mediatek-filogic-cmcc_rax3000m-nand-preloader.bin` | BL2 引导(备用) |
| `immortalwrt-mediatek-filogic-cmcc_rax3000m-nand-bl31-uboot.fip` | ATF+U-Boot(备用) |
| `immortalwrt-mediatek-filogic.manifest` | 固件内完整包清单 |
