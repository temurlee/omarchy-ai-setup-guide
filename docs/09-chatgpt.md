# 优化 ChatGPT 桌面应用的中文输入体验

## 配置收益

让 Omarchy 中的 ChatGPT 桌面应用保持原生 Wayland 模式，并修复使用 fcitx5/Rime 输入中文时候选框位置不跟随光标、候选字过小等问题。

本章不负责安装 ChatGPT。如果系统中没有 `chatgpt` 命令或桌面入口，智能体应停止并询问用户，不得猜测安装来源。

## 执行前检查

```bash
command -v chatgpt
test -f /usr/share/applications/chatgpt.desktop && echo "desktop entry found"
test -f ~/.config/codex-flags.conf && cat ~/.config/codex-flags.conf
```

确认中文输入法已按[中文输入与应用兼容](04-fcitx5-rime.md)配置并正常运行。

## 1. 保持原生 Wayland 模式

ChatGPT 启动脚本 `/usr/bin/chatgpt` 会在 Wayland 会话中自动添加 `--ozone-platform=wayland`，但 `~/.config/codex-flags.conf` 中的旧参数可能强制应用使用 X11。

如果存在以下参数，先备份文件，再删除或注释该行：

```text
--ozone-platform=x11
```

不要直接修改 `/usr/bin/chatgpt`。

## 2. 创建用户级桌面入口

不要修改 `/usr/share/applications/chatgpt.desktop`。将其复制到用户目录进行覆盖：

```bash
mkdir -p ~/.local/share/applications
cp --backup=numbered /usr/share/applications/chatgpt.desktop \
  ~/.local/share/applications/chatgpt.desktop
```

在用户级文件中找到启动应用的 `Exec=` 行，在命令前加入 `env XDG_CURRENT_DESKTOP=KDE`：

```ini
Exec=env XDG_CURRENT_DESKTOP=KDE chatgpt %U
```

该环境变量只应用于 ChatGPT，避免全局修改 `XDG_CURRENT_DESKTOP` 影响文件选择器、portal 或其他程序。

## 3. 验证

完全退出并重新启动 ChatGPT，然后检查：

1. `hyprctl clients` 中 ChatGPT 显示 `xwayland: False`。
2. 在输入框中使用 Rime 输入中文时，候选框跟随光标。
3. 候选字大小与其他原生 Wayland 应用一致。
4. 诊断信息中的 ChatGPT 输入前端为 `wayland_v2`，而不是 `dbus`：

```bash
fcitx5-diagnose | grep -E "program:|frontend:|cap:"
```

## 回滚

删除用户级桌面入口即可恢复系统提供的入口：

```bash
rm ~/.local/share/applications/chatgpt.desktop
```

如果修改过 `~/.config/codex-flags.conf`，使用执行前创建的备份恢复。

## 相关文件

| 项目 | 路径 |
|---|---|
| ChatGPT 启动脚本 | `/usr/bin/chatgpt`（只读） |
| ChatGPT 启动参数 | `~/.config/codex-flags.conf` |
| 系统桌面入口 | `/usr/share/applications/chatgpt.desktop`（只读） |
| 用户桌面入口 | `~/.local/share/applications/chatgpt.desktop` |

