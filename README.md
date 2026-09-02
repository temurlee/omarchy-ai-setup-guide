# Omarchy AI Setup Guide

这是一份面向人类和 AI 智能体的 Omarchy 系统恢复手册。目标是在全新安装的 Omarchy 系统上，按照可检查、可验证、可回滚的步骤恢复个人工作环境。

## 使用方式

1. 让智能体先完整阅读 [`AGENTS.md`](AGENTS.md)。
2. 从 [`docs/00-prerequisites.md`](docs/00-prerequisites.md) 开始检查系统和用户输入。
3. 按下列顺序逐篇执行；每一步验证通过后才进入下一步。
4. `optional/` 中的内容不属于默认流程，只有用户明确要求时才能执行。

## 默认执行顺序

1. [执行前检查](docs/00-prerequisites.md)
2. [Web Apps](docs/01-web-apps.md)
3. [网络准备与下载排错](docs/02-network.md)
4. [Mihomo 代理](docs/03-mihomo.md)
5. [中文输入与应用兼容](docs/04-fcitx5-rime.md)
6. [输入法主题](docs/05-input-method-theme.md)
7. [桌面行为与显示](docs/06-desktop-behavior.md)
8. [Omarchy Shell 插件](docs/07-shell-plugins.md)
9. [应用配置](docs/08-applications.md)
10. [验证清单与关键路径](docs/09-verification.md)

## 可选内容

- [Omawrite 定制](optional/omawrite-customization.md)：需要本地编译并覆盖系统程序。
- [双系统、Secure Boot 与 RTC](optional/dual-boot-secureboot-rtc.md)：涉及固件、启动链和 Windows 设置，风险较高。

## 重要说明

- 文档中的版本和路径基于作者当前环境，执行前必须核对。
- Mihomo 订阅链接、密码、令牌和私钥不得写入本仓库。
- 本仓库不会自动执行命令；智能体必须逐步说明、检查和验证。
- Omarchy 更新后，应重新核对命令、配置格式和可选定制。
