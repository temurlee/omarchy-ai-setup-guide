# Omarchy Shell 插件

## 6. 通知中心插件（shavanced.notification-center）

### 6.1 作用

第三方**通知中心**插件，配合 Omarchy 自带 `omarchy.notifications` 通知服务使用：
- 在 bar 右侧显示通知铃铛图标 + 未读数量；
- 点击打开通知中心面板：查看实时通知、持久化历史、搜索、逐条关闭、清空；
- 自带勿扰（Do Not Disturb）开关；
- **不替换** Omarchy 原有通知弹窗，也不额外占用通知守护进程。

> 项目主页：https://github.com/Shavanced/omarchy-notification-center-plugin

### 6.2 安装

```bash
omarchy plugin add https://github.com/Shavanced/omarchy-notification-center-plugin.git --enable
```

若已安装只是没启用：
```bash
omarchy plugin enable shavanced.notification-center
```

验证启用状态：
```bash
omarchy plugin list | grep notification
# 应看到：shavanced.notification-center  enabled
```

### 6.3 放到 bar 右侧

```bash
omarchy bar move shavanced.notification-center --section right
```

### 6.4 测试

```bash
notify-send -a "Test" "Notification Center" "Hello from Omarchy!"
# 点击 bar 铃铛图标应能看到这条通知
```

> 依赖 Omarchy 自带的 `omarchy.notifications` 服务（first-party，默认启用），无需额外安装。

---

## 7. wl-clip-persist 安装配置

### 7.1 作用

解决 Wayland 剪贴板"复制它的应用关闭后内容消失"的问题：
后台守护进程在应用退出时代为持有剪贴板，之后 Ctrl+V 依然有效。

### 7.2 安装

```bash
pkexec pacman -S wl-clip-persist    # 0.5.0-2，官方 extra 仓库
```

- 项目主页: https://github.com/Linus789/wl-clip-persist
- 原理: 复制时把剪贴板数据全部读入内存，再以自己为持有者重新写入，从而顶替退出的应用。
- 参数: 建议只接管 `--clipboard regular`。**不要开 primary**（中键选区），
  会导致 GTK 应用无法选中文本（见其 README Troubleshooting）。

### 7.3 开机自启

`~/.config/hypr/autostart.lua` 添加：

```lua
-- Keep Wayland clipboard alive after the copying app closes.
o.launch_on_start("wl-clip-persist --clipboard regular")
```

> `o.launch_on_start` 会套 `uwsm-app --`，确保拿到 Wayland 会话环境（`~/.config/hypr/autostart.lua` 引用自 `hyprland.lua`）。

### 7.4 立即启动 + 验证

```bash
nohup uwsm-app -- wl-clip-persist --clipboard regular > /tmp/wl-clip-persist.log 2>&1 &
ps aux | grep wl-clip-persist     # 进程在跑即可

# 模拟"复制→来源退出→粘贴"（CLI 验证）
printf 'test-%s' "$(date +%s)" | wl-copy &
sleep 1; kill -9 $!; wait $! 2>/dev/null
wl-paste                          # 仍能读到内容 = 生效
```

### 7.5 与 omarchy 剪贴板插件的关系

两者可共存、不冲突：
- omarchy 插件：记录历史，随时翻找重新复制；
- wl-clip-persist：保住"当前"这份剪贴板，来源应用退出也不丢。

---

## 8. Exposé 窗口概览插件（expose.window-overview）

### 8.1 作用

第三方 **overlay** 插件，macOS 风格的 Exposé（窗口概览）：
- 打开后把当前聚焦显示器的所有窗口以实时预览平铺展示；
- 输入文字可搜索窗口，`Space` 快速预览（Quick Look），`Enter` 激活窗口；
- 点击卡片激活窗口、中键点击关闭窗口；支持拖拽移动窗口；
- 支持多显示器：只在聚焦显示器上打开，并显示该显示器的窗口。

> 项目主页：https://github.com/kristofferR/omarchy-expose
> 插件 id：`expose.window-overview`

### 8.2 安装

```bash
omarchy plugin add https://github.com/kristofferR/omarchy-expose.git --enable
```

验证启用状态：
```bash
omarchy plugin list | grep expose
# 应看到：expose.window-overview  enabled  third-party overlay
```

### 8.3 触发方式：四指上滑打开、四指下滑关闭

作者默认的热角（左上角）已**关闭**，改用四指触控板手势控制。

在 `~/.config/hypr/input.lua` 末尾添加：

```lua
-- Exposé (expose.window-overview): four-finger swipe up to open, down to close.
hl.gesture({
  fingers = 4,
  direction = "up",
  action = function()
    hl.dispatch(hl.dsp.exec_cmd("omarchy-shell expose open"))
  end,
})

hl.gesture({
  fingers = 4,
  direction = "down",
  action = function()
    hl.dispatch(hl.dsp.exec_cmd("omarchy-shell expose close"))
  end,
})
```

关闭热角：
```bash
omarchy-shell expose hotCorner off
```

### 8.4 常用 IPC 命令

```bash
omarchy-shell expose toggle        # 打开/关闭（也支持 open / close）
omarchy-shell expose settings toggle   # 打开设置面板
omarchy-shell expose hotCorner on|off  # 热角开关
```

### 8.5 验证

```bash
hyprctl reload
hyprctl configerrors    # 无报错
omarchy-shell expose toggle   # 四指上滑或此命令应打开概览
```

> 插件依赖 Quickshell 原生的 Hyprland 模型（窗口/工作区/显示器状态），不依赖 `hyprexpo`、`hyprpm` 或客户端轮询。

---
