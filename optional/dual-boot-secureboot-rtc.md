# 双系统修复笔记：Secure Boot 与 Windows 时间

系统：Omarchy (Arch) + Windows 双系统，Limine 12.6.0 引导器，UKI 内核。

---

## 一、Secure Boot（已解决）

### 问题
BIOS 开启 Secure Boot 后无法进入 Linux，只能进 Windows。
原因：引导文件未签名（Limine、UKI 等），固件拒绝加载。

### 已完成的设置
1. 安装 `sbctl`，生成密钥（PK/KEK/db）
2. 在 BIOS 清空原有密钥（进入 Setup Mode）后注册：本机密钥 + Microsoft 密钥
   （保留 Microsoft 密钥，Windows 才能继续启动）
3. 签名以下文件：
   - `/boot/EFI/limine/limine_x64.efi`（引导器）
   - `/boot/EFI/Linux/omarchy_linux.efi`（内核 UKI）
   - `/boot/EFI/arch/fwupdx64.efi`（fwupd 固件更新）
   - `/boot/878583882c694d3e9e8ea359ec27c14e/limine_history/*`（回滚快照 UKI）
4. 全部登记到 sbctl 数据库（`/var/lib/sbctl/files.json`），
   更新后 `zz-sbctl` pacman hook 会自动重新签名
5. 启用 Limine 配置校验（`/etc/default/limine` 中
   `ENABLE_ENROLL_LIMINE_CONFIG=yes`），limine.conf 的 BLAKE2B 已嵌入
   limine 二进制并签名
6. 刷新 limine.conf / snapshots.json 中所有文件哈希，与签名后文件一致

### 验证命令
```bash
sudo sbctl status        # 查看密钥状态
sudo sbctl verify        # 检查所有引导文件是否已签名
```

### 重要注意事项
- **手动编辑 `/boot/limine.conf` 后必须重新嵌入配置哈希**，
  否则 Secure Boot 开启时 Limine 会 panic：
  ```bash
  sudo limine-enroll-config
  ```
- 内核 / 引导器更新会自动签名（已实测验证），无需手动处理
- 若开机失败：回 BIOS 临时关闭 Secure Boot 排查

---

## 二、Windows 时间错误（待处理）

### 原因
Linux 把硬件时钟（RTC）当 UTC，Windows 默认当本地时间（CST +8），
两个系统互相覆盖 RTC 导致时间错 8 小时。

### 修复方法
在 **Windows** 管理员 CMD 中运行（推荐，一次永久生效）：

```cmd
reg add "HKLM\SYSTEM\CurrentControlSet\Control\TimeZoneInformation" /v RealTimeIsUniversal /t REG_DWORD /d 1 /f
```

重启进 Windows 确认时间正确即可。

（备选方案：在 Linux 执行 `sudo timedatectl set-local-rtc 1 --adjust-system-clock`，
但会影响 NTP 同步和 RTC 唤醒闹钟，不推荐。）