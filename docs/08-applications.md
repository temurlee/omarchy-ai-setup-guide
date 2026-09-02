# 应用配置

## 10. Figma 本地字体（微软雅黑）

### 10.1 背景

Figma 官方**没有 Linux 版字体助手**，且网页版检测到 **Linux User-Agent 会禁用字体助手**，导致无法使用本机字体。解决方案：

1. 装本地字体服务 `figma-agent`（监听 `127.0.0.1:44950`）；
2. 用 **Windows User-Agent** 启动 Figma（`figma.sh`），让网页版连接该服务。

### 10.2 安装 figma-agent（AUR）

```bash
# AUR 二进制包，免编译；在 ~/.cache/aur-build 构建避免 sudo 卡住
mkdir -p "$HOME/.cache/aur-build"
git clone https://aur.archlinux.org/figma-agent-linux-bin.git "$HOME/.cache/aur-build/figma-agent-linux-bin"
cd "$HOME/.cache/aur-build/figma-agent-linux-bin"
makepkg -sf --noconfirm
cp figma-agent-linux-bin-*.pkg.tar.zst "$HOME/figma-agent.pkg.tar.zst"
pkexec pacman -U --noconfirm "$HOME/figma-agent.pkg.tar.zst"   # 弹图形认证框
rm "$HOME/figma-agent.pkg.tar.zst"
```

### 10.3 启用服务（socket 激活，按需启动不常驻）

```bash
systemctl --user enable --now figma-agent.socket
systemctl --user is-active figma-agent.socket   # active
ss -tln | grep 44950                            # 127.0.0.1:44950 监听
```

### 10.4 创建启动脚本 `~/.local/bin/figma.sh`

```bash
cat > "$HOME/.local/bin/figma.sh" << 'EOF'
#!/bin/bash
# Windows UA 让 Figma 启用字体助手，连接本地 figma-agent（127.0.0.1:44950）
exec /opt/google/chrome/chrome \
  --user-agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/151.0.0.0 Safari/537.36" \
  --app=https://www.figma.com/
EOF
chmod +x "$HOME/.local/bin/figma.sh"
```

### 10.5 更新桌面入口 `~/.local/share/applications/Figma.desktop`

```bash
# Exec 必须是绝对路径，~ 在 desktop 文件里不生效，用 $HOME 拼接生成
cat > "$HOME/.local/share/applications/Figma.desktop" << EOF
[Desktop Entry]
Version=1.0
Name=Figma
Comment=Figma Design Tool (local fonts)
Exec=$HOME/.local/bin/figma.sh
Terminal=false
Type=Application
Icon=figma
StartupNotify=true
EOF
```

### 10.6 Chrome 本地网络权限

Chrome 限制网页连接本机应用（Local Network Access），需给 figma.com 授权。**先完全关闭 Chrome**，再改 `~/.config/google-chrome/Default/Preferences`：

```bash
python3 << 'EOF'
import json, time, os
p = os.path.expanduser('~/.config/google-chrome/Default/Preferences')
d = json.load(open(p))
exceptions = d['profile']['content_settings'].setdefault('exceptions', {})
figma = 'https://www.figma.com:443,*'
now = int(time.time()*1000000)
for cat in ['local_network','local_network_access']:
    c = exceptions.setdefault(cat, {})
    if figma not in c:
        c[figma] = {'last_modified': str(now), 'setting': 1}
    else:
        c[figma]['setting'] = 1
json.dump(d, open(p,'w'), indent=2)
EOF
```

> `setting: 1` = 允许。`local_fonts`、`loopback_network` 通常已允许，无需再改。

### 10.7 安装微软雅黑字体（Regular + Bold 都要）

```bash
mkdir -p ~/.local/share/fonts
# msyh.ttc（Regular）、msyhbd.ttc（Bold）← Bold 易遗漏
curl -sL -C - -o ~/.local/share/fonts/msyh.ttc   "https://github.com/fernvenue/microsoft-yahei/raw/refs/heads/master/msyh.ttc"
curl -sL -C - -o ~/.local/share/fonts/msyhbd.ttc "https://github.com/fernvenue/microsoft-yahei/raw/refs/heads/master/msyhbd.ttc"
fc-cache -f ~/.local/share/fonts
fc-list | grep -i yahei    # 应看到 Regular 和 Bold 两条
```

> 国内下载慢可用 `curl -C -` 断点续传循环（见[网络准备](02-network.md#82-github-文件下载慢中断)）。

### 10.8 启动 + 使用

```bash
setsid nohup ~/.local/bin/figma.sh >/tmp/figma-launch.log 2>&1 &
pgrep -af 'figma.com' | grep 'Windows NT'    # 确认 UA 生效

# 验证服务
curl -s http://127.0.0.1:44950/figma/version              # => {"package":"...","version":...}
curl -s http://127.0.0.1:44950/figma/font-files | grep -o 'Microsoft YaHei' | head -1
```

浏览器内：文件 → 选中文字图层 → 字体下拉 → 过滤 **"Installed by you"** → 搜 **Microsoft YaHei**。首次连接若 Chrome 弹 **"Apps on device"** 提示，点允许。

### 10.9 注意事项

- **必须用 `Figma.desktop`（figma.sh）启动**，不要用 `omarchy-launch-webapp https://www.figma.com/`，否则丢 Windows UA、本地字体失效。
- 若 figma-agent 曾卸载重装，Chrome 的 figma.com 权限**不会丢**（存在 Preferences 里），无需重设。
- 停用：`systemctl --user disable --now figma-agent.socket`。

---

## 12. 微信（wechat-universal-bwrap + HiDPI 缩放）

### 12.1 安装

微信 Universal（bwrap 沙箱版，来源 AUR，`wechat-universal-bwrap`，维护者 7Ji）：

```bash
omarchy pkg aur add wechat-universal-bwrap
```

- 构建时下载官方 `.deb`（约 202.5 MB，来源 `linux.weixin.qq.com`），sha256 校验通过后打包进 bwrap 沙箱。
- **必须在有 sudo 交互的终端跑**（本机还绑定了指纹认证），非交互 shell 会在提权一步失败。
- 数据目录默认 `~/Documents/WeChat_Data`（聊天文件、图片都在这）。

启动 / 退出：

```bash
wechat-universal                       # 或从启动器打开
/usr/lib/wechat-universal/stop.sh      # 退出（也用于重启用，先停再启）
```

### 12.2 关键坑：HiDPI 下窗口太小

现象：2880x1920 @ 2x 缩放下窗口按 1x 渲染，整体过小。

原因：微信 Universal 只支持 xcb（走 XWayland），启动脚本 `wechat-universal.sh` 写死了
`QT_AUTO_SCREEN_SCALE_FACTOR=1`，在 XWayland 下探测不到 HiDPI 缩放。

### 12.3 修复：用户级 desktop 覆盖，写死 QT_SCALE_FACTOR

创建 `~/.local/share/applications/wechat-universal.desktop`（用户目录优先于系统目录，
**升级 `wechat-universal-bwrap` 不会覆盖**），`Exec` 用 `env QT_SCALE_FACTOR=1.75` 启动：

```bash
mkdir -p ~/.local/share/applications
cat > ~/.local/share/applications/wechat-universal.desktop <<'EOF'
[Desktop Entry]
Type=Application
Name=WeChat (Universal)
Comment=WeChat Universal (HiDPI fix)
TryExec=/usr/lib/wechat-universal/start.sh
Exec=env QT_SCALE_FACTOR=1.75 /usr/lib/wechat-universal/start.sh %u
Icon=wechat-universal
Categories=Network;InstantMessaging;Chat;
Terminal=false
StartupWMClass=wechat
X-GNOME-SingleWindow=true
SingleMainWindow=true
Actions=quit;

[Desktop Action quit]
Name=Quit WeChat
Exec=/usr/lib/wechat-universal/stop.sh
Icon=application-exit
EOF
update-desktop-database ~/.local/share/applications
```

调整后生效方式：`/usr/lib/wechat-universal/stop.sh` 先停 → 改 `QT_SCALE_FACTOR` → 再从启动器打开。
本机最终值：**1.75x**（2x 偏大、1.5x 偏小，取 1.75）。

### 12.4 注意事项

- 只有从**启动器 / 桌面入口**打开才带缩放；手动敲 `wechat-universal` 走原脚本、无缩放（日常用不到）。
- 沙箱内只读绑定 `/usr`、字体目录；输入法走 fcitx 自动 workaround（`QT_IM_MODULE=fcitx`）。
- 升级包后 desktop 修复仍生效（用户覆盖优先），无需重做。

---
