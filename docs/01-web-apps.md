# Web Apps

## 1. 清理预装 Web App (只保留 X 和 YouTube)

系统预装了多个以 `omarchy-launch-webapp` 启动的 Web App(实际是 `.desktop` 快捷方式, 指向对应 URL, 用 Chrome 打开)。

### 1.1 识别 Web App

Web App 的 `.desktop` 文件在 `~/.local/share/applications/`, 特征是 `Exec=omarchy-launch-webapp <url>`:

```bash
for f in ~/.local/share/applications/*.desktop; do
  echo "=== $f ==="
  grep -E "^(Name|Exec)" "$f"
done
```

### 1.2 本次保留/删除清单

保留:**X** (`X.desktop`, x.com)、**YouTube** (`YouTube.desktop`)。

> **后加的 Web App（非预装，保留，需创建）：**
> - **Feishu** (`Feishu.desktop`) → `https://www.feishu.cn/messenger/`
> - **Feishu Docs** (`Feishu Docs.desktop`) → `https://www.feishu.cn/drive/home/`
> - **Teams** (`Teams.desktop`) → `https://teams.microsoft.com/v2/`
> - **Figma** (`Figma.desktop`) → `~/.local/bin/figma.sh`（Windows UA，配 `figma-agent` 用本地字体，详见[应用配置](08-applications.md#10-figma-本地字体微软雅黑)）

创建上述 4 个 desktop 入口（Figma 用 figma.sh，其余用 `omarchy-launch-webapp`）：

```bash
mkdir -p ~/.local/share/applications
# Feishu 需加 XDG_CURRENT_DESKTOP=KDE 修复输入法选词条定位（见第 13 章）；
# 其余用 omarchy-launch-webapp 即可。
for name in "Feishu:https://www.feishu.cn/messenger/:feishu" "Feishu Docs:https://www.feishu.cn/drive/home/:feishu-docs" "Teams:https://teams.microsoft.com/v2/:teams"; do
  n="${name%%:*}"; rest="${name#*:}"; url="${rest%%:*}"; icon="${rest#*:}"
  printf '[Desktop Entry]\nVersion=1.0\nName=%s\nComment=%s\nExec=omarchy-launch-webapp %s\nTerminal=false\nType=Application\nIcon=%s\nStartupNotify=true\n' \
    "$n" "$n" "$url" "$icon" > "$HOME/.local/share/applications/$n.desktop"
done
# Figma 入口由第 10.5 节创建（Icon=figma）

# 单独修正 Feishu 入口，加上输入法修复所需的 KDE 环境（详见第 13.4 节）
sed -i 's|^Exec=omarchy-launch-webapp https://www.feishu.cn/messenger/$|Exec=env XDG_CURRENT_DESKTOP=KDE omarchy-launch-webapp https://www.feishu.cn/messenger/|' \
  "$HOME/.local/share/applications/Feishu.desktop"
```

> **Web App 图标（ico）**：desktop 入口需配 `Icon=` 才有专属图标。
> - **X / YouTube**：图标由 `omarchy-settings` 包提供（`/usr/share/icons/hicolor/`），无需额外处理。
> - **Feishu / Feishu Docs / Teams / Figma**：图标不属任何包，需手动放置（备份在 `assets/icons/` 下的 feishu.png、feishu-docs.png、teams.png、figma.png）：
>   ```bash
>   mkdir -p ~/.local/share/icons/hicolor/256x256/apps
>   install -m644 assets/icons/{feishu.png,feishu-docs.png,teams.png,figma.png} \
>     ~/.local/share/icons/hicolor/256x256/apps/
>   ```
>   `Figma.desktop` 的 `Icon=figma` 依赖同一份 figma.png。

删除以下 9 个(命令用 `rm -vf`，文件不存在则跳过不报错):

```bash
rm -vf \
  ~/.local/share/applications/Basecamp.desktop \
  ~/.local/share/applications/Discord.desktop \
  ~/.local/share/applications/Google\ Contacts.desktop \
  ~/.local/share/applications/Google\ Maps.desktop \
  ~/.local/share/applications/Google\ Messages.desktop \
  ~/.local/share/applications/Google\ Photos.desktop \
  ~/.local/share/applications/HEY.desktop \
  ~/.local/share/applications/WhatsApp.desktop \
  ~/.local/share/applications/Zoom.desktop
```

> 注意: `~/.local/share/applications/` 下还有**非 Web App**(Disk Usage、Docker、foot、imv、mpv), 不要误删。删除时务必先逐个核对 `Exec=` 是否为 `omarchy-launch-webapp`。

---
