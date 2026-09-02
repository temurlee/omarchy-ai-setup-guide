# Omarchy AI Setup Guide

这是一份面向人类和 AI 智能体的 Omarchy 系统恢复手册。目标是在全新安装的 Omarchy 系统上，按照可检查、可验证、可回滚的步骤恢复个人工作环境。

## 使用方式

1. 让智能体先完整阅读 [`AGENTS.md`](AGENTS.md)。
2. 从 [`docs/00-prerequisites.md`](docs/00-prerequisites.md) 开始检查系统和用户输入。
3. 按下列顺序逐篇执行；每一步验证通过后才进入下一步。
4. `optional/` 中的内容不属于默认流程，只有用户明确要求时才能执行。

## 配置流程

下面的步骤会把一台新安装的 Omarchy 系统逐步配置成作者日常使用的工作环境。你可以完成整个流程，也可以只选择自己需要的功能。

1. **[确认系统是否适合使用本指南](docs/00-prerequisites.md)**

   检查 Omarchy、Hyprland、输入法服务和 AUR 工具的当前状态，确认文档与系统版本是否匹配。本步骤只读取信息，不修改系统。

2. **[整理常用网站应用和桌面图标](docs/01-web-apps.md)**

   检查 Omarchy 预装的 Web Apps，保留 X 和 YouTube，移除不需要的预装入口，并添加飞书、飞书文档、Teams 和 Figma 的桌面入口与图标。删除前必须由用户确认。

3. **[解决软件安装和 GitHub 下载问题](docs/02-network.md)**

   配置 Go 模块镜像，并提供 GitHub 大文件断点续传和下载校验方法，减少安装 AUR 软件及下载依赖时的超时和文件损坏。

4. **[配置系统代理与网络访问](docs/03-mihomo.md)**

   安装 Mihomo 核心和 Omarchy 控制面板，使用用户提供的 Clash 订阅，配置 TUN、DNS、Geo 数据和服务启动，并记录常见断网与 DNS 污染问题的排查方法。订阅地址和 secret 不会写入仓库。

5. **[安装中文输入法并修复应用兼容问题](docs/04-fcitx5-rime.md)**

   安装 fcitx5、Rime、简体拼音、Emoji 和中文字体；设置 CapsLock 切换输入法，并修复 Chrome、飞书和 ChatGPT 等 Wayland 应用中的候选框位置与大小问题。

6. **[美化中文输入法界面](docs/05-input-method-theme.md)**

   安装并启用 `mellow-graphite-dark` 输入法主题，同时说明 fcitx5 主题配置的正确写法和常见配置陷阱。

7. **[调整触控板、显示和 Chrome 使用体验](docs/06-desktop-behavior.md)**

   调整触控板滚动方向与手势行为，将 Chrome 界面设置为中文，并根据代理环境关闭可能影响连接稳定性的 QUIC。

8. **[添加通知中心和剪贴板功能](docs/07-shell-plugins.md)**

   安装 Omarchy 通知中心插件并放到状态栏右侧，同时启用 `wl-clip-persist`，让复制内容在来源应用关闭后仍可粘贴。

9. **[配置 Figma、微信等常用应用](docs/08-applications.md)**

   配置 Figma 本地字体服务和微软雅黑字体，创建专用启动入口；安装微信并针对 HiDPI 屏幕修正界面缩放。

10. **[优化 ChatGPT 桌面应用的中文输入体验](docs/09-chatgpt.md)**

    让 ChatGPT 保持原生 Wayland 模式，修复使用 fcitx5/Rime 输入中文时候选框位置不跟随光标、候选字过小等问题，并通过用户级桌面入口保存设置。

11. **[检查所有配置并准备回滚](docs/10-verification.md)**

    汇总检查 Hyprland、输入法、代理、通知中心、剪贴板和应用入口是否正常，同时记录备份位置及恢复方法。

## 可选内容

以下项目不属于默认配置流程。只有确实需要对应功能，并且用户明确同意相关风险后，智能体才可以执行。

1. **[让 Omawrite 支持独立字号和自适应行宽](optional/omawrite-customization.md)**

   修改并编译 Omawrite 源码，为编辑器增加字号快捷键、字号持久化和更宽的自适应内容区域。安装定制版本会备份并覆盖 `/usr/bin/omawrite`，Omarchy 更新也可能恢复官方版本，因此执行前必须取得用户确认。

2. **[修复双系统的 Secure Boot 和时间问题](optional/dual-boot-secureboot-rtc.md)**

   为 Limine、Omarchy UKI 内核和固件更新程序配置 Secure Boot 签名，并说明如何让 Windows 与 Linux 统一使用 UTC 硬件时钟。该流程涉及 BIOS、启动密钥、EFI 文件和 Windows 注册表，操作失误可能导致系统无法启动，必须逐步确认和验证。

## 重要说明

- 文档中的版本和路径基于作者当前环境，执行前必须核对。
- Mihomo 订阅链接、密码、令牌和私钥不得写入本仓库。
- 本仓库不会自动执行命令；智能体必须逐步说明、检查和验证。
- Omarchy 更新后，应重新核对命令、配置格式和可选定制。
