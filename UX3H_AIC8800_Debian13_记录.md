# 6.12 内核适配变更摘要（基于本仓库 diff）

- `rwnx_radar.c`：对 `cfg80211_radar_event()` 的调用在 `>= 6.12` 时新增了一个参数，当前以 `0` 作为占位，确保与 6.12 之后的函数签名匹配，避免编译错误。

（其余改动多为代码风格与安装流程的优化，不是 6.12 特定兼容点。）

# Mercury UX3H（AX300 / AIC8800）Debian 安装指南（离线优先）

本指南基于仓库 `aic8800fdrvpackage`（`bk1d/aic8800fdrvpackage`）的实际安装逻辑整理：它的 `.deb` 在安装时会执行 `postinst`，自动 `make && make install && insmod`，因此**即使你已经有 deb 包**，系统里仍然需要具备编译环境与内核头文件，否则会出现“缺少 make/gcc”或“/lib/modules/.../build 不存在”等错误。

## 0. 你现在的文件位置（双系统/WSL 注意）

你把仓库克隆在 Windows 盘上（WSL 路径是）：`/mnt/d/Personal/utils/tmp/aic8800fdrvpackage`

重启进原生 Debian 后，Windows 分区的挂载点可能不是 `/mnt/d`。你可以用下面命令找出挂载点再进入对应目录：

```bash
lsblk -f
```

## 1. 安装前检查（强烈建议先做）

### 1.1 检查是否已安装编译工具链

```bash
dpkg -s build-essential >/dev/null 2>&1 && echo "build-essential: 已安装" || echo "build-essential: 未安装"
```

如果你看到类似官方说明里“`make: 未找到命令` / `gcc-xx: not found`”，就是这里缺了。

### 1.2 检查当前内核是否有 headers（关键）

驱动 Makefile 默认用：`/lib/modules/$(uname -r)/build` 作为内核构建目录。

```bash
test -e "/lib/modules/$(uname -r)/build" && echo "kernel build dir: OK" || echo "kernel build dir: 缺失"
```

如果出现官方截图那种：

- `"/lib/modules/xxx/build: 没有那个文件或目录"`

基本就是没有装 `linux-headers-$(uname -r)`（或者装了但内核版本对不上）。

## 2. 离线安装依赖（Debian 13 DVD）

离线 DVD 安装的 Debian，**不一定**默认带 `build-essential` 和 `linux-headers-*`；是否有取决于你安装时是否勾选了“开发工具”等组件。

建议做法是：把安装 DVD 配置成 apt 源，然后用 `apt` 装依赖（不需要联网）。

### 2.1 把 DVD 作为 apt 源（推荐）

把 DVD 挂载后（例如 `/media/cdrom` 或 `/mnt/cdrom`），执行：

```bash
sudo apt-cdrom add
sudo apt update
```

然后安装依赖：

```bash
sudo apt install -y build-essential linux-headers-$(uname -r) eject
```

如果 `linux-headers-$(uname -r)` 找不到，通常是：

- 你当前运行的内核不是 DVD 仓库里对应版本（例如你装了更新/定制内核）。

此时可选方案：

- 先换回发行版自带内核版本（与 DVD 匹配）再装 headers
- 或者接网配置官方源/镜像源后安装对应 headers

## 3. 打包并安装驱动（Deb 方式）

在仓库根目录执行（文件名可按你内核与架构调整）：

```bash
dpkg-deb -b src aic8800fdrvpackage_linux_$(uname -r)_$(dpkg --print-architecture).deb
```

安装：

```bash
sudo dpkg -i aic8800fdrvpackage_linux_*.deb
```

安装脚本会做的关键动作（便于你理解报错点）：

- 复制 udev 规则到 `/etc/udev/rules.d/aic.rules` 并 reload
- 复制固件目录到 `/lib/firmware/aic8800DC`
- 进入 `/AIC8800/drivers/aic8800/` 执行 `make && make install`
- `modprobe cfg80211`
- `insmod aic_load_fw.ko`、`insmod aic8800_fdrv.ko`

## 4. 安装后验证

```bash
lsmod | grep -E 'aic8800|aic_load_fw' || true
ip a
dmesg -T | tail -200
```

另外确认设备是否识别（USB）：

```bash
lsusb
```

（此包的 udev 规则里是 `idVendor==a69c`、`idProduct==5721`。）

## 5. 常见问题对照（对应你贴的官方截图）

### 5.1 “缺少 make / gcc”

现象：

- `make: 未找到命令`
- `gcc-12: not found`（或其他 gcc 版本）

处理：

```bash
sudo apt install -y build-essential
```

### 5.2 “/lib/modules/$(uname -r)/build 不存在”

现象：

- `.../build: 没有那个文件或目录`

处理：

```bash
sudo apt install -y linux-headers-$(uname -r)
```

如果找不到对应 headers：

- 先确认 `uname -r` 的内核是不是 Debian 官方仓库/安装介质提供的版本
- 必要时切换到发行版内核或联网安装对应 headers

### 5.3 Secure Boot（双系统常见）

现象（安装过程中或 `insmod` 时）：

- `Required key not available`

处理：

- 在 BIOS/UEFI 里关闭 Secure Boot，或对模块进行签名（更麻烦，一般不建议）

### 5.4 “编译看似正常，但 postinst 最后返回 1（安装失败）”

这种情况很常见：前面的 `make` 都成功了，但 **postinst 后半段某一步命令返回非 0**，`dpkg` 就会把安装判为失败。

建议你在 Debian 里这样定位失败点（直接看最后几行错误最有效）：

```bash
sudo dpkg -i aic8800fdrvpackage_linux_*.deb
```

如果你想精确定位是哪条命令失败，可以看 dpkg 落地的脚本并用 `bash -x` 跑一遍：

```bash
sudo sed -n '1,120p' /var/lib/dpkg/info/aic8800fdrvpackage.postinst
sudo bash -x /var/lib/dpkg/info/aic8800fdrvpackage.postinst configure
```

一个“看起来像内核不兼容但其实不是”的典型坑：`aicrf_test` 目录的 `make install` 里用了 `sudo cp ...`。在最小化 Debian/离线环境下经常**没有安装 sudo**，于是最后一步失败，导致 `postinst` 返回 1（即使驱动模块已经编译成功）。

解决办法：

- 简单：`sudo apt install -y sudo`（离线可用 ISO 作为源）
- 更干净：把 `aicrf_test/Makefile` 的 `sudo cp` 改成 `install`（我已经在本地仓库里做了这个修复，重新打包再装即可）

另一个会导致“看上去只是 warning，最后却失败”的点：某些发行版/构建配置会把 `-Wimplicit-fallthrough` 之类的告警当成错误（`-Werror`），这时像 `// no break` 这种注释不会被 GCC 识别为“刻意 fallthrough”，从而直接失败。

我已把驱动里的两处 `// no break` 改成内核常用的 `fallthrough;`，以避免在较新 GCC/内核构建配置下触发 `implicit-fallthrough` 的致命错误。

## 7. 失败证据采集（强烈建议：先采集再改代码）

你遇到的现象通常是：

- 编译输出里有一堆 warning，看上去“好像没事”
- 但最后 `dpkg` 报 `post-installation script subprocess returned error status 1`
- 或看到类似 `Makefile.build:xxx: ... 错误 2`

**注意**：`错误 2` 只是 “make 收到子命令失败” 的汇总信息，真正的原因在它前面那条 `error:`（或模块插入的 `insmod` 错误）。

### 7.1 抓 dpkg 安装完整日志

```bash
sudo dpkg -i aic8800fdrvpackage_linux_*.deb 2>&1 | tee /tmp/aic8800_install.log
```

看最后 200 行：

```bash
tail -200 /tmp/aic8800_install.log
```

快速定位第一条真正的编译错误：

```bash
rg -n "error:" /tmp/aic8800_install.log | head -50
```

如果是模块插入失败，重点查这些关键词：

```bash
rg -n "insmod|Invalid module format|vermagic|Unknown symbol|Required key not available|Lockdown" /tmp/aic8800_install.log | head -200
```

**更详细日志（推荐）**：本仓库的 `postinst` 已支持额外调试输出。

```bash
sudo AIC_DEBUG=1 dpkg -i aic8800fdrvpackage_linux_*.deb
```

调试日志会写入：

```
/var/log/aic8800_install.log
```

如果你想自定义日志路径：

```bash
sudo AIC_DEBUG=1 AIC_LOG=/tmp/aic8800_install_verbose.log dpkg -i aic8800fdrvpackage_linux_*.deb
```

### 7.2 抓内核日志（模块加载相关）

```bash
dmesg -T | tail -200 > /tmp/aic8800_dmesg_tail.log
tail -200 /tmp/aic8800_dmesg_tail.log
```

重点查这些关键词（Secure Boot / 符号不匹配 / 格式不对）：

```bash
rg -n "aic|cfg80211|lockdown|Required key|Unknown symbol|vermagic|Invalid module" /tmp/aic8800_dmesg_tail.log
```

### 7.3 关键信息一并提供（便于快速判断）

```bash
uname -r
dpkg --print-architecture
mokutil --sb-state 2>/dev/null || true
```

### 7.4 常见“证据 → 结论”

- `Required key not available` / `Lockdown`：Secure Boot/内核锁定导致模块拒载（与“内核太高”无关）
- `Invalid module format` + `vermagic`：headers/内核版本对不上（常见于 headers 装错版本）
- `Unknown symbol ...`：内核导出符号/API 变更或依赖模块缺失（需要根据具体 symbol 适配代码或补依赖）
- 日志里出现 `warning: this statement may fall through` 后就失败：可能是 `-Werror` 把告警当错误（本仓库我已把两处 `// no break` 改成 `fallthrough;` 以规避）

## 6. 卸载（如需）

```bash
sudo dpkg -r aic8800fdrvpackage
```

它会尝试：

- `rmmod aic8800_fdrv`、`rmmod aic_load_fw`
- `make uninstall` 并清理固件目录等



## 8. Debian 13（kernel 6.12）编译失败的代码级修复点

如果你的安装日志里出现下面这些 **编译错误（error）**，就不是“sudo/依赖缺失”，而是 **cfg80211 API 变更导致代码不兼容**：

- `error: too many arguments to function 'cfg80211_ch_switch_notify'`
- `error: too many arguments to function 'cfg80211_ch_switch_started_notify'`
- `error: 'struct cfg80211_csa_settings' has no member named 'punct_bitmap'`
- `error: ... start_radar_detection ... incompatible pointer type`（cfg80211_ops 回调签名变了）
- `error: too few arguments to function 'cfg80211_cac_event'`

本仓库已做的适配（重新打包 deb 再装即可）：

- `src/AIC8800/drivers/aic8800/aic8800_fdrv/rwnx_main.c`：调整 `cfg80211_*` 调用的参数数量与版本判断；补上 `start_radar_detection` 的 `link_id` 参数（新内核需要）
- `src/AIC8800/drivers/aic8800/aic8800_fdrv/rwnx_radar.c`：补 `cfg80211_cac_event` 新增参数（适配 6.12）
- `src/AIC8800/drivers/aic8800/aic8800_fdrv/rwnx_msg_tx.c`、`rwnx_tx.c`、`rwnx_txq.c`：补 `fallthrough;`（降低 `implicit-fallthrough` 风险）

重新打包并安装（在修改后的仓库根目录执行）：

```bash
dpkg-deb -b src aic8800fdrvpackage_linux_$(uname -r)_$(dpkg --print-architecture).deb
sudo dpkg -i aic8800fdrvpackage_linux_*.deb
```

# My record
[四 1月 29 23:55:22 2026] usb 1-3.2: new high-speed USB device number 11 using xhci_hcd
[四 1月 29 23:55:22 2026] usb 1-3.2: New USB device found, idVendor=2357, idProduct=0147, bcdDevice= 1.00
[四 1月 29 23:55:22 2026] usb 1-3.2: New USB device strings: Mfr=1, Product=2, SerialNumber=3
[四 1月 29 23:55:22 2026] usb 1-3.2: Product: AIC8800DC
[四 1月 29 23:55:22 2026] usb 1-3.2: Manufacturer: AICSemi
[四 1月 29 23:55:22 2026] usb 1-3.2: SerialNumber: 20220127
[四 1月 29 23:55:45 2026] audit: type=1400 audit(1769702145.316:172): apparmor="ALLOWED" operation="open" class="file" profile="Xorg" name="/tmp/server-1.xkm" pid=1873 comm="Xorg" requested_mask="r" denied_mask="r" fsuid=0 ouid=1000
[四 1月 29 23:55:45 2026] audit: type=1400 audit(1769702145.316:173): apparmor="ALLOWED" operation="unlink" class="file" profile="Xorg" name="/tmp/server-1.xkm" pid=1873 comm="Xorg" requested_mask="d" denied_mask="d" fsuid=0 ouid=1000
[四 1月 29 23:55:45 2026] audit: type=1400 audit(1769702145.584:174): apparmor="ALLOWED" operation="open" class="file" profile="Xorg" name="/tmp/server-1.xkm" pid=1873 comm="Xorg" requested_mask="r" denied_mask="r" fsuid=0 ouid=1000
[四 1月 29 23:55:45 2026] audit: type=1400 audit(1769702145.584:175): apparmor="ALLOWED" operation="unlink" class="file" profile="Xorg" name="/tmp/server-1.xkm" pid=1873 comm="Xorg" requested_mask="d" denied_mask="d" fsuid=0 ouid=1000
[四 1月 29 23:55:45 2026] audit: type=1400 audit(1769702145.624:176): apparmor="ALLOWED" operation="open" class="file" profile="Xorg" name="/tmp/server-1.xkm" pid=1873 comm="Xorg" requested_mask="r" denied_mask="r" fsuid=0 ouid=1000
[四 1月 29 23:55:45 2026] audit: type=1400 audit(1769702145.624:177): apparmor="ALLOWED" operation="unlink" class="file" profile="Xorg" name="/tmp/server-1.xkm" pid=1873 comm="Xorg" requested_mask="d" denied_mask="d" fsuid=0 ouid=1000
[四 1月 29 23:56:51 2026] audit: type=1400 audit(1769702211.172:178): apparmor="ALLOWED" operation="open" class="file" profile="Xorg" name="/tmp/server-1.xkm" pid=1873 comm="Xorg" requested_mask="r" denied_mask="r" fsuid=0 ouid=1000
[四 1月 29 23:56:51 2026] audit: type=1400 audit(1769702211.172:179): apparmor="ALLOWED" operation="unlink" class="file" profile="Xorg" name="/tmp/server-1.xkm" pid=1873 comm="Xorg" requested_mask="d" denied_mask="d" fsuid=0 ouid=1000
[四 1月 29 23:56:51 2026] audit: type=1400 audit(1769702211.408:180): apparmor="ALLOWED" operation="open" class="file" profile="Xorg" name="/tmp/server-1.xkm" pid=1873 comm="Xorg" requested_mask="r" denied_mask="r" fsuid=0 ouid=1000
[四 1月 29 23:56:51 2026] audit: type=1400 audit(1769702211.408:181): apparmor="ALLOWED" operation="unlink" class="file" profile="Xorg" name="/tmp/server-1.xkm" pid=1873 comm="Xorg" requested_mask="d" denied_mask="d" fsuid=0 ouid=1000
[四 1月 29 23:56:51 2026] audit: type=1400 audit(1769702211.456:182): apparmor="ALLOWED" operation="open" class="file" profile="Xorg" name="/tmp/server-1.xkm" pid=1873 comm="Xorg" requested_mask="r" denied_mask="r" fsuid=0 ouid=1000
[四 1月 29 23:56:51 2026] audit: type=1400 audit(1769702211.456:183): apparmor="ALLOWED" operation="unlink" class="file" profile="Xorg" name="/tmp/server-1.xkm" pid=1873 comm="Xorg" requested_mask="d" denied_mask="d" fsuid=0 ouid=1000





## 第一次修改

  cd /path/to/aic8800fdrvpackage
  dpkg-deb -b src aic8800fdrvpackage_linux_$(uname -r)_$(dpkg --print-architecture).deb
  sudo dpkg -i ./aic8800_neo_amd64.deb 2>&1 | tee /tmp/aic8800_install.log && cp /tmp/aic8800_install.log ./aic8800_install.log2 
  rg -n "error:" /tmp/aic8800_install.log | head -50


## 第二次修改

  已改动

  - tmp/aic8800fdrvpackage/src/AIC8800/drivers/aic8800/aic8800_fdrv/rwnx_radar.c：为 cfg80211_cac_event 增加 6.12 需要的新参数
  - tmp/aic8800fdrvpackage/src/DEBIAN/postinst：增加详细日志与可选调试开关（AIC_DEBUG=1），日志写入 /var/log/aic8800_install.log
  - tmp/aic8800fdrvpackage/UX3H_AIC8800_Debian_安装指南.md：补充新的错误类型与调试方法

  你下一步要做

  1. 用修改后的源码重新打包 deb（否则还是旧代码会继续失败）

     cd /path/to/aic8800fdrvpackage
     dpkg-deb -b src aic8800fdrvpackage_linux_$(uname -r)_$(dpkg --print-architecture).deb
  2. 用调试模式安装（会生成更详细日志）

     sudo AIC_DEBUG=1 dpkg -i aic8800fdrvpackage_linux_*.deb
  3. 看日志位置：/var/log/aic8800_install.log

  如果还有失败，把新的 /var/log/aic8800_install.log 里 error: 那几行贴出来即可，我继续补兼容点。
  
  
  ## install second time , seems success
  
    CC [M]  /AIC8800/drivers/aic8800/aic8800_fdrv/regdb.o
  CC [M]  /AIC8800/drivers/aic8800/aic8800_fdrv/md5.o
  CC [M]  /AIC8800/drivers/aic8800/aic8800_fdrv/aicwf_compat_8800dc.o
/AIC8800/drivers/aic8800/aic8800_fdrv/aicwf_compat_8800dc.c:1714:5: warning: no previous prototype for ‘aicwf_patch_table_load’ [-Wmissing-prototypes]
 1714 | int aicwf_patch_table_load(struct rwnx_hw *rwnx_hw, char *filename)
      |     ^~~~~~~~~~~~~~~~~~~~~~
/AIC8800/drivers/aic8800/aic8800_fdrv/aicwf_compat_8800dc.c:1774:6: warning: no previous prototype for ‘aicwf_patch_config_8800dc’ [-Wmissing-prototypes]
 1774 | void aicwf_patch_config_8800dc(struct rwnx_hw *rwnx_hw)
      |      ^~~~~~~~~~~~~~~~~~~~~~~~~
/AIC8800/drivers/aic8800/aic8800_fdrv/aicwf_compat_8800dc.c:1927:5: warning: no previous prototype for ‘aicwf_set_rf_config_8800dc’ [-Wmissing-prototypes]
 1927 | int aicwf_set_rf_config_8800dc(struct rwnx_hw *rwnx_hw, struct mm_set_rf_calib_cfm *cfm){
      |     ^~~~~~~~~~~~~~~~~~~~~~~~~~
/AIC8800/drivers/aic8800/aic8800_fdrv/aicwf_compat_8800dc.c:1961:5: warning: no previous prototype for ‘aicwf_plat_patch_load_8800dc’ [-Wmissing-prototypes]
 1961 | int aicwf_plat_patch_load_8800dc(struct rwnx_hw *rwnx_hw){
      |     ^~~~~~~~~~~~~~~~~~~~~~~~~~~~
/AIC8800/drivers/aic8800/aic8800_fdrv/aicwf_compat_8800dc.c:1989:9: warning: no previous prototype for ‘rwnx_plat_userconfig_load_8800dc’ [-Wmissing-prototypes]
 1989 | int     rwnx_plat_userconfig_load_8800dc(struct rwnx_hw *rwnx_hw){
      |         ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
/AIC8800/drivers/aic8800/aic8800_fdrv/aicwf_compat_8800dc.c:2017:6: warning: no previous prototype for ‘system_config_8800dc’ [-Wmissing-prototypes]
 2017 | void system_config_8800dc(struct rwnx_hw *rwnx_hw){
      |      ^~~~~~~~~~~~~~~~~~~~
/AIC8800/drivers/aic8800/aic8800_fdrv/aicwf_compat_8800dc.c: In function ‘aicwf_plat_patch_load_8800dc’:
/AIC8800/drivers/aic8800/aic8800_fdrv/aicwf_compat_8800dc.c:1965:9: warning: ‘sprintf’ argument 3 overlaps destination object ‘aic_fw_path’ [-Wrestrict]
 1965 |         sprintf(aic_fw_path, "%s/%s", aic_fw_path, "aic8800DC");
      |         ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
/AIC8800/drivers/aic8800/aic8800_fdrv/aicwf_compat_8800dc.c:1960:13: note: destination object referenced by ‘restrict’-qualified argument 1 was declared here
 1960 | extern char aic_fw_path[200];
      |             ^~~~~~~~~~~
  CC [M]  /AIC8800/drivers/aic8800/aic8800_fdrv/rwnx_radar.o
/AIC8800/drivers/aic8800/aic8800_fdrv/rwnx_radar.c:900:22: warning: no previous prototype for ‘pri_detector_init’ [-Wmissing-prototypes]
  900 | struct pri_detector *pri_detector_init(struct dfs_pattern_detector *dpd,
      |                      ^~~~~~~~~~~~~~~~~
  CC [M]  /AIC8800/drivers/aic8800/aic8800_fdrv/rwnx_debugfs.o
/AIC8800/drivers/aic8800/aic8800_fdrv/rwnx_debugfs.c: In function ‘rwnx_dbgfs_rc_fixed_rate_idx_write’:
/AIC8800/drivers/aic8800/aic8800_fdrv/rwnx_debugfs.c:1753:13: warning: the comparison will always evaluate as ‘false’ for the address of ‘mac’ will never be NULL [-Waddress]
 1753 |     if (mac == NULL)
      |             ^~
/AIC8800/drivers/aic8800/aic8800_fdrv/rwnx_debugfs.c:1740:8: note: ‘mac’ declared here
 1740 |     u8 mac[6];
      |        ^~~
/AIC8800/drivers/aic8800/aic8800_fdrv/rwnx_debugfs.c: At top level:
/AIC8800/drivers/aic8800/aic8800_fdrv/rwnx_debugfs.c:2095:6: warning: no previous prototype for ‘_rwnx_dbgfs_rc_stat_write’ [-Wmissing-prototypes]
 2095 | void _rwnx_dbgfs_rc_stat_write(struct rwnx_debugfs *rwnx_debugfs, uint8_t sta_idx)
      |      ^~~~~~~~~~~~~~~~~~~~~~~~~
  CC [M]  /AIC8800/drivers/aic8800/aic8800_fdrv/rwnx_fw_trace.o
  CC [M]  /AIC8800/drivers/aic8800/aic8800_fdrv/usb_host.o
/AIC8800/drivers/aic8800/aic8800_fdrv/usb_host.c:35:30: warning: no previous prototype for ‘aicwf_usb_host_txdesc_get’ [-Wmissing-prototypes]
   35 | volatile struct txdesc_host *aicwf_usb_host_txdesc_get(struct usb_host_env_tag *env, const int queue_idx)
      |                              ^~~~~~~~~~~~~~~~~~~~~~~~~
  CC [M]  /AIC8800/drivers/aic8800/aic8800_fdrv/aicwf_txrxif.o
  CC [M]  /AIC8800/drivers/aic8800/aic8800_fdrv/aicwf_usb.o
/AIC8800/drivers/aic8800/aic8800_fdrv/aicwf_usb.c:148:6: warning: no previous prototype for ‘rwnx_stop_sta_all_queues’ [-Wmissing-prototypes]
  148 | void rwnx_stop_sta_all_queues(struct rwnx_sta *sta, struct rwnx_hw *rwnx_hw)
      |      ^~~~~~~~~~~~~~~~~~~~~~~~
/AIC8800/drivers/aic8800/aic8800_fdrv/aicwf_usb.c:158:6: warning: no previous prototype for ‘rwnx_wake_sta_all_queues’ [-Wmissing-prototypes]
  158 | void rwnx_wake_sta_all_queues(struct rwnx_sta *sta, struct rwnx_hw *rwnx_hw)
      |      ^~~~~~~~~~~~~~~~~~~~~~~~
/AIC8800/drivers/aic8800/aic8800_fdrv/aicwf_usb.c:1616:6: warning: no previous prototype for ‘aicwf_usb_cancel_all_urbs’ [-Wmissing-prototypes]
 1616 | void aicwf_usb_cancel_all_urbs(struct aic_usb_dev *usb_dev){
      |      ^~~~~~~~~~~~~~~~~~~~~~~~~
  LD [M]  /AIC8800/drivers/aic8800/aic8800_fdrv/aic8800_fdrv.o
  MODPOST /AIC8800/drivers/aic8800/Module.symvers
  CC [M]  /AIC8800/drivers/aic8800/aic_load_fw/aic_load_fw.mod.o
  CC [M]  /AIC8800/drivers/aic8800/.module-common.o
  LD [M]  /AIC8800/drivers/aic8800/aic_load_fw/aic_load_fw.ko
  BTF [M] /AIC8800/drivers/aic8800/aic_load_fw/aic_load_fw.ko
  CC [M]  /AIC8800/drivers/aic8800/aic8800_fdrv/aic8800_fdrv.mod.o
  LD [M]  /AIC8800/drivers/aic8800/aic8800_fdrv/aic8800_fdrv.ko
  BTF [M] /AIC8800/drivers/aic8800/aic8800_fdrv/aic8800_fdrv.ko
make[1]: 离开目录“/usr/src/linux-headers-6.12.63+deb13-amd64”
[4/7] install kernel modules
mkdir -p /lib/modules/6.12.63+deb13-amd64/kernel/drivers/net/wireless/aic8800
install -p -m 644 aic_load_fw/aic_load_fw.ko  /lib/modules/6.12.63+deb13-amd64/kernel/drivers/net/wireless/aic8800/
install -p -m 644 aic8800_fdrv/aic8800_fdrv.ko  /lib/modules/6.12.63+deb13-amd64/kernel/drivers/net/wireless/aic8800/
/sbin/depmod -a 6.12.63+deb13-amd64
[5/7] load modules
insmod done
[6/7] build rftest tools
gcc -c wifi_test.c -o wifi_test.o
gcc wifi_test.o -o wifi_test
gcc -c bt_test.c -o bt_test.o
gcc bt_test.o -lpthread -o bt_test
[7/7] install rftest tools
install -m 0755 wifi_test /sbin/wifi_test
install -m 0755 bt_test /sbin/bt_test
Install aic8800 wifi driver successful!!!!!
===== [2026-01-30 01:03:33] aic8800 postinst end =====
