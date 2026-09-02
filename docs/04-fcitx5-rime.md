# 中文输入与应用兼容

## 2. 中文输入法（fcitx5 + Rime）

### 2.1 安装软件包

```bash
yay -S --noconfirm fcitx5-rime fcitx5-configtool rime-emoji
# 中文字体优化（修复「复」「门」等字渲染过窄/变形的问题）
yay -S --noconfirm adobe-source-han-sans-cn-fonts
```

> `rime-emoji` 官方包把 emoji 数据装到 `/usr/share/rime-data/`（opencc + emoji_suggestion.yaml），
> librime 全局目录会自动搜索，无需手动复制到用户目录。

### 2.2 创建 Rime 配置

用户目录：`~/.local/share/fcitx5/rime/`

```bash
mkdir -p ~/.local/share/fcitx5/rime
```

**`default.custom.yaml`** — 默认方案为朙月拼音简体：

```yaml
patch:
  schema_list:
    - schema: luna_pinyin_simp
```

**`luna_pinyin_simp.custom.yaml`** — Emoji 支持：

```yaml
patch:
  switches/@next:
    name: emoji_suggestion
    reset: 1
    states: [ "🈚︎", "🈶️" ]
  'engine/filters/@before 0':
    simplifier@emoji_suggestion
  emoji_suggestion:
    opencc_config: emoji.json
    option_name: emoji_suggestion
    tips: none
    inherit_comment: false
```

### 2.3 注册 Rime 到 fcitx5

`~/.config/fcitx5/profile`：

```ini
[Groups/0]
Name=Default
Default Layout=us
DefaultIM=rime

[Groups/0/Items/0]
Name=keyboard-us
Layout=

[Groups/0/Items/1]
Name=rime
Layout=

[GroupOrder]
0=Default
```

### 2.4 重启 fcitx5 生效

```bash
systemctl --user restart omarchy-fcitx5.service
```

### 排错经验

- **不要用 `pkill fcitx5` 手动重启**。`omarchy-fcitx5.service` 带 `Restart=always`，
  手动 kill 会与 systemd 抢进程，导致新实例抢不到 dbus 名、配置被旧实例覆盖（Rime 从 profile 里被清掉）。
  一律用 `systemctl --user restart omarchy-fcitx5.service`。
- 若 Rime 在 profile 中被清除，重写 profile 后用 systemctl 干净重启即可。
- 若怀疑 fcitx5 未读取配置，可用 dbus 查内存中的实际值：
  `dbus-send --session --print-reply --dest=org.fcitx.Fcitx5 /controller org.fcitx.Fcitx.Controller1.GetConfig string:'fcitx://config/addon/classicui'`

---

## 5. CapsLock 切换输入法（macOS 风格）

需求：CapsLock 切换中/英文输入，且不再切换大小写；大小写改用 Shift 触发。

### 两个前置问题

1. Omarchy 默认 `compose:caps` 把 CapsLock 映射为 Compose 键 → 需移除。
2. Hyprland 绑定 `CAPS_LOCK`（keysym）触发命令时，XKB 仍会把该键当成大写锁定键翻转状态 → 需用 `caps:none` 禁用。

### 配置

`~/.config/hypr/input.lua`（键盘部分）：

```lua
hl.config({
  input = {
    -- 禁用 CapsLock 的大小写功能（改作输入法切换键）；
    -- 同时保留「双 Shift 切换大写锁定」。
    kb_options = "caps:none,shift:both_capslock_cancel",
  },
})
```

`~/.config/hypr/bindings.lua`：

```lua
-- CapsLock 切换输入法。
-- 用 keycode 绑定：caps:none 移除了 Caps_Lock keysym，keysym 绑定会失效。
-- keycode 66 = CapsLock（见 /usr/share/X11/xkb/keycodes/evdev 中 <CAPS> = 66）
o.bind("code:66", "Toggle input method", "fcitx5-remote -t")
```

### 验证

```bash
hyprctl getoption input:kb_options     # 应含 caps:none,shift:both_capslock_cancel
hyprctl binds | grep 'Toggle input'    # 应显示 key: code:66
fcitx5-remote -t                       # 应在 rime 与 keyboard-us 间切换
```

---

## 13. Chrome 系应用输入法选词条定位修复（XDG_CURRENT_DESKTOP=KDE）

### 13.1 问题现象

在 **Hyprland (Wayland) 原生模式**下，Chrome 系应用的网页输入框（飞书 web app、ChatGPT app 等）输入中文时：

- 选词条**不跟随光标**（第一次出现在窗口左上角，之后不再出现），或完全看不到
- 其他程序（GTK/Qt 原生应用）正常

### 13.2 根因

- Chrome/Chromium 在 Wayland 原生模式下，网页输入框强制走 **text-input-v3 协议**（不受 `GTK_IM_MODULE` 控制）
- Hyprland 的 text-input-v3 实现**不发送 `done` 事件**，而 Chromium 严格按协议依赖 `done` 事件才会上报光标位置（`set_cursor_rectangle`）
- fcitx5 拿不到光标坐标 → 选词条无法定位
- 上游 issue：Chromium `#384531043`（Chromium 官方确认修复需 compositor 侧配合）

> 注意：Chrome 的**原生 UI**（地址栏等）走 GTK IM module，不受此问题影响；
> 只有**网页内容**走 text-input 协议才有问题。

### 13.3 解决方案：`XDG_CURRENT_DESKTOP=KDE`

让 Chrome 系应用以为自己在 KDE Plasma 下运行，从而走不同的 text-input 路径：

```bash
XDG_CURRENT_DESKTOP=KDE <chrome-ish-command> ...
```

**不生效的尝试（避免重复踩坑）**：

| 尝试 | 结果 |
|------|------|
| `--wayland-text-input-version=3` | 无效（Chrome 依赖的是 Hyprland 的 `done` 事件） |
| `--gtk-version=4` | 只影响 Chrome 原生 UI，网页输入仍走 text-input；**已从 `~/.config/chrome-flags.conf` 移除** |
| `GTK_IM_MODULE=fcitx` 全局 | 只修好 Chrome 地址栏，网页输入仍异常 |
| 飞书/ChatGPT 强制走 X11（`--ozone-platform=x11`） | 选词条跟随正常，但候选字用 Xft.dpi=96 渲染，字号偏小；且改 Xft.dpi 会**全局影响所有 XWayland 应用**（界面放大），不可行 |

### 13.4 飞书（Feishu web app）

飞书桌面入口 `~/.local/share/applications/Feishu.desktop`：

```ini
[Desktop Entry]
Version=1.0
Name=Feishu
Comment=Feishu
Exec=env XDG_CURRENT_DESKTOP=KDE omarchy-launch-webapp https://www.feishu.cn/messenger/
Terminal=false
Type=Application
Icon=feishu
StartupNotify=true
```

> 关键：`XDG_CURRENT_DESKTOP=KDE` 要放在 `Exec` 行的命令前面（`env` 前缀）。
> `omarchy-launch-webapp` 会透传 URL，KDE 环境变量经 `env` 传给子进程。

**固化到环境（可选，全局生效）**：把 `XDG_CURRENT_DESKTOP=KDE` 写入
`~/.config/environment.d/99-kde-desktop.conf`，则所有 Chrome 系应用自动生效。
但注意这会**影响其他依赖 `XDG_CURRENT_DESKTOP` 的程序**（文件选择器、portal 等），
因此推荐只对飞书/ChatGPT 单独配置，不全局设置。

### 13.5 ChatGPT app

ChatGPT 的完整检查、用户级桌面入口、验证和回滚步骤已经独立整理到
[优化 ChatGPT 桌面应用的中文输入体验](09-chatgpt.md)。不要修改系统提供的桌面入口或 `/usr/bin/chatgpt`。

### 13.6 验证

1. 启动飞书 / ChatGPT，确认窗口是 **Wayland 原生**（`hyprctl clients` 显示 `xwayland: False`）。
2. 在网页输入框输入中文：
   - 选词条**跟随光标**
   - 候选字大小与原生 Wayland 应用一致
3. 诊断 IC：

```bash
fcitx5-diagnose | grep -E "program:|frontend:|cap:"
# ChatGPT 正常时 frontend 应为 wayland_v2 且 cap=100000072 级别（而非 dbus）
```

### 13.7 相关文件

| 项目 | 路径 |
|------|------|
| 飞书桌面入口 | `~/.local/share/applications/Feishu.desktop` |
| ChatGPT 启动脚本 | `/usr/bin/chatgpt` |
| ChatGPT flags | `~/.config/codex-flags.conf` |
| ChatGPT 桌面入口 | `/usr/share/applications/chatgpt.desktop`（或 `~/.local/share/applications/` 覆盖） |
| fcitx5 全局环境 | `/usr/share/omarchy/default/environment.d/10-omarchy-fcitx.conf`（缺 `GTK_IM_MODULE=fcitx`，已补到用户目录） |

---
