# 执行前检查

开始配置前，智能体必须先阅读仓库根目录的 [`AGENTS.md`](../AGENTS.md)，然后完成本页检查。这里不修改系统。

## 目标环境

| 项目 | 目标状态 |
|---|---|
| 系统 | Omarchy（Arch Linux + Hyprland + Wayland） |
| Hyprland | 文档最初基于 0.56.x Lua 配置；执行时必须核对当前版本 |
| 输入法 | fcitx5 用户服务 + Rime |
| AUR 助手 | `yay` 或当前 Omarchy 推荐方式 |
| 需要用户提供 | Mihomo Clash 订阅链接，仅在执行时使用 |

## 只读检查

```bash
omarchy version
omarchy commands
hyprctl version
systemctl --user status omarchy-fcitx5.service --no-pager
command -v yay
```

如果当前命令、配置格式或服务名称与文档不一致，暂停并向用户说明差异，不要强行套用旧步骤。

## 执行原则

- 一次只执行 README 所列的一篇文档。
- 先读取目标文件和现有设置，避免重复安装或覆盖用户修改。
- 在交互式终端里需要提权时使用 `sudo`；无法交互输入密码的图形执行环境才使用 `pkexec`。
- 标记为“需用户提供”或“必须确认”的步骤必须暂停询问。
- 每篇文档验证成功后才继续下一篇。
- `optional/` 不属于默认安装流程。

## 执行顺序

返回 [`README.md`](../README.md#配置流程) 按顺序执行。网络准备应早于 Mihomo；代理可用后再处理依赖 GitHub 下载的应用。
