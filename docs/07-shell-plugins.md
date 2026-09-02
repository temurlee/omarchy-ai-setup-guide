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
