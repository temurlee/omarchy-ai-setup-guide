# 键盘媒体键定制指南（PrtSc / F7–F11 / Insert）

系统：Omarchy (Hyprland) @ Lenovo IdeaPad，内置键盘为 `at-translated-set-2-keyboard`。

---

## 背景

这台笔记本的 **PrtSc 和 F7–F11 等键在硬件/固件层被重映射**，按下的不是标准的 `Print` / `F7`…`F11` 键，而是：

- **媒体键值**（如 `XF86SelectiveScreenshot`、`XF86RFKill`、`XF86Favorites`）
- **`Super + 字母` 宏**（如 F7 = `Super+p`、F10 = `Super+i`、F11 = `Super+l`）
- 个别键发的键码**没有有效键名**（NoSymbol），只有键码没有 keysym

因此 Omarchy 默认给这些键绑定（`PRINT` 截图、`F9` 听写等）**全部对不上**，按了没反应或触发奇怪行为（例如 PrtSc 一按就弹出 Google Maps——因为 PrtSc 宏里的 `Super+Shift+S` 恰好是 Omarchy 的 Google Maps 快捷键）。

本指南记录逐键核实实际键值、改绑 `~/.config/hypr/bindings.lua` 的过程与最终结果。

---

## 这台键盘的实际键值

以下为实测值（`wev` + Hyprland `input.keyboard.key` 事件 + `xkbcli` 交叉确认）：

| 物理键 | 图标 | 实际发出的内容 | 类型 |
|--------|------|----------------|------|
| PrtSc | 截图 | `XF86SelectiveScreenshot` + 额外宏 `Super+Shift+S` | 媒体键 + 宏 |
| F7 | 投屏 | `Super+p`（keycode 33） | Super 宏 |
| F8 | 飞行 | `XF86RFKill` | 媒体键 |
| F9 | 听写 | keycode 146（`Help` 键，无有效 keysym，按下/松开几乎同时发出） | 无键名键码 |
| F10 | 设置 | `Super+i` | Super 宏 |
| F11 | 锁屏 | `Super+l` | Super 宏 |
| Insert | 智能体 | `XF86Favorites` | 媒体键 |

> keycode 与 keysym 的对应：`xkbcli compile-keymap --layout us --model pc105` 可查。例如 keycode 33 = `<AD10>` = `p`，keycode 146 = `<HELP>`。

---

## 诊断方法（本次排查用到的）

```bash
# 1. 抓实际键值（按住要测的键，看 sym 行）
wev

# 2. 用 Hyprland 内置事件记录按键序列（能捕获 wev 看不到的宏按键）
hyprctl eval 'hl.on("input.keyboard.key", function(...) local p={}; for i=1, select("#",...) do p[i]=tostring(select(i,...)) end; os.execute("echo \"[$(date +%H:%M:%S)] " .. table.concat(p," | ") .. "\" >> /tmp/keys.log") end)'

# 3. 查键位表确认 keycode -> keysym
xkbcli compile-keymap --layout us --model pc105 | grep -E "key <HELP>|key <AC04>"
```

---

## 最终按键绑定（`~/.config/hypr/bindings.lua` 的改动部分）

| 按键 | 最终功能 | 实现方式 |
|------|----------|----------|
| PrtSc | 截图（框选） | 绑定 `XF86SelectiveScreenshot` |
| Alt+PrtSc | 全屏录屏开关 | 绑定 `ALT + XF86SelectiveScreenshot` |
| Super+PrtSc | 取色器 | 绑定 `SUPER + XF86SelectiveScreenshot` |
| Super+Ctrl+PrtSc | 截屏取字(OCR) | 绑定 `SUPER + CTRL + XF86SelectiveScreenshot` |
| Super+Shift+S | 解除（原 Google Maps） | `hl.unbind`，防止 PrtSc 宏误触发 |
| F7 | 显示器面板（投屏） | 把 `SUPER + P`（F7 宏）改绑 |
| F8 | 飞行模式（Wi-Fi/蓝牙） | 绑定 `XF86RFKill` → `rfkill toggle` |
| F9 | 语音听写（开关） | 绑定 `code:146` → `voxtype record toggle` |
| F10 | Atmos 偏好设置 | 把 `SUPER + I`（F10 宏）改绑 |
| F11 | 锁屏 | 把 `SUPER + L`（F11 宏）改绑 |
| Insert | 启动智能体 | 绑定 `XF86Favorites` → `omarchy agent` |

对应配置片段：

```lua
-- PrtSc：这台键盘发 XF86SelectiveScreenshot 而非 PRINT
o.bind("XF86SelectiveScreenshot", "Screenshot", "omarchy-capture-screenshot")
o.bind("ALT + XF86SelectiveScreenshot", "Screenrecording", "omarchy-capture-screenrecording --stop-recording || omarchy-capture-screenrecording --fullscreen")
o.bind("SUPER + XF86SelectiveScreenshot", "Color picker", "pkill hyprpicker || hyprpicker -a")
o.bind("SUPER + CTRL + XF86SelectiveScreenshot", "Extract text (OCR) from screenshot", "omarchy-capture-text")

-- PrtSc 还会发 Super+Shift+S 宏，与 Google Maps 冲突，解除
hl.unbind("SUPER + SHIFT + S")

-- F8 飞行模式
o.bind("XF86RFKill", "Toggle airplane mode", "rfkill toggle")

-- F10 设置 -> Atmos 偏好设置
o.bind("SUPER + I", "Atmos preferences", o.launch("atmos"))

-- Insert 智能体
o.bind("XF86Favorites", "Agent", "omarchy agent")

-- F11 锁屏（F11 = Super+l 宏，重定向 Super+L）
hl.unbind("SUPER + L")
o.bind("SUPER + L", "Lock screen", "omarchy-system-lock")

-- F9 听写（F9 = keycode 146，无 keysym，且无法按住 -> 用开关模式）
hl.unbind("F9")
o.bind("code:146", "Toggle dictation", "voxtype record toggle")

-- F7 投屏（F7 = Super+p 宏，重定向 Super+P）
hl.unbind("SUPER + P")
o.bind("SUPER + P", "Display panel", "omarchy-shell shell toggle omarchy.monitor")
```

> **为什么"改绑 Super+字母"**：F7/F10/F11 这类键在硬件层就是 `Super+字母` 宏，Hyprland 无法区分"物理 F7"和"手动 Super+p"，所以只能把那个 `Super+字母` 组合重定向成想要的功能（会占用该组合的默认功能，如 `Super+P` 伪平铺、`Super+L` 布局切换）。

---

## F9 语音听写（voxtype）额外依赖

F9 的听写走 voxtype，需要守护进程和语音模型，否则按了没反应：

```bash
# 下载 Whisper 模型（base.en，约 141MB）
voxtype setup --model base.en --download

# 启用并启动守护进程（开机自启）
systemctl --user enable --now voxtype.service

# 检查
voxtype setup check
voxtype status        # 期望：idle
```

> `voxtype setup check` 中的 `input` 组警告只影响 voxtype 内置 evdev 热键；用 Hyprland 键位绑定则无需加入 `input` 组。

---

## 验证

```bash
# 1. 配置无错误
hyprctl reload && hyprctl configerrors

# 2. 确认绑定已注册
hyprctl binds -j | jq -r '.[] | select(.description|test("Screenshot|Dictation|Display panel|Lock screen|Atmos|airplane|Agent";"i")) | "\(.description)"'

# 3. 逐键手测
#    PrtSc        -> 弹截图选区
#    Alt+PrtSc    -> 开始/停止全屏录屏
#    F7           -> 弹显示器面板（同 Super+Ctrl+D）
#    F8           -> 切换飞行模式（Wi-Fi/蓝牙）
#    F9           -> 按一下开始听写、再按停止
#    F10          -> 打开 Atmos 设置
#    F11          -> 锁屏
#    Insert       -> 启动编程智能体
```

---

## 回滚

所有改动都在用户级配置文件 `~/.config/hypr/bindings.lua`，删除其中新增的绑定（或 `omarchy refresh hyprland` 重置整套 Hyprland 配置，会自动备份）即可恢复默认。

```bash
# 重置 Hyprland 配置为默认（会先备份当前配置）
omarchy refresh hyprland
```

voxtype 相关（服务 + 模型）：

```bash
systemctl --user disable --now voxtype.service
rm -rf ~/.local/share/voxtype/models   # 删除模型（如需彻底移除）
```

---

## 已知问题与注意事项

1. **`Super+字母` 宏无法独立于手动组合键**：F7/F10/F11 被硬件固定为 `Super+p/i/l`，与手动按 `Super+P/I/L` 等价。改绑会占用对应 `Super+字母` 的默认功能。
2. **F9 无法"按住说话"**：它的按下/松开几乎同时到达，只能做成"按一次开始、再按一次停止"的开关模式。
3. **PrtSc 宏 = `Super+Shift+S`**：若未来重新启用 Google Maps 绑定，必须避开 `Super+Shift+S`，否则按 PrtSc 会再次弹地图。
4. **Hyprland 0.56 的 modifier-only release 回归（GitHub #15837）**：把单独的 `Super` 键绑成"松开时触发"会在使用其他 Super 组合键后误触发，0.56.2 未修复。F11 通过重定向 `Super+L` 实现锁屏，绕开了该问题。
5. **模型/服务**：F9 听写依赖 voxtype 守护进程与 Whisper 模型；`pacman -Syu` 后模型/服务不受影响，但若 voxtype 包升级导致配置变化，需重新 `voxtype setup check`。