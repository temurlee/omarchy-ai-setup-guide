# 输入法主题

## 3. 输入法主题（皮肤）

### 关键坑：classicui.conf 的写法

**`[Theme]` 节头写法无效，必须用顶层键！**

fcitx5 5.1.x 的 `Configuration::load` 中，`[Theme]` 节名与 option 路径 `Theme` 同名，
导致读到的是 group 节点而非字符串值，解析失败回退默认主题。
正确写法（`~/.config/fcitx5/conf/classicui.conf`）：

```ini
Theme=mellow-graphite-dark
```

常用配置项（均为顶层键）：
- `Theme` — 亮色主题
- `DarkTheme` — 暗色主题（默认 `default-dark`）
- `UseDarkTheme` — 是否跟随系统深浅色（默认关）
- `UseAccentColor` — 是否跟随系统强调色（默认开）

### 主题来源

| 来源 | 包 | 说明 |
|------|-----|------|
| 系统自带 | — | `default` / `default-dark` |
| 官方仓库 | `fcitx5-material-color` | Material Design 多配色 |
| 官方仓库 | `fcitx5-nord` | Nord 极简配色 |
| AUR | `fcitx5-mellow-themes-git` | 本机采用，5 款×亮暗（圆角现代风） |

本机启用：**石墨暗色 `mellow-graphite-dark`**。

### 安装 AUR 第三方主题（避免 sudo 卡住）

```bash
# 1. 克隆 AUR 包并在临时目录构建（依赖满足时无需 sudo）
mkdir -p "$HOME/.cache/aur-build"
git clone https://aur.archlinux.org/fcitx5-mellow-themes-git.git "$HOME/.cache/aur-build/mellow"
cd "$HOME/.cache/aur-build/mellow"
makepkg -sf --noconfirm

# 2. 安装（sudo 需密码，用 pkexec 会弹图形认证框）
pkexec pacman -U "$HOME/.cache/aur-build/mellow"/fcitx5-mellow-themes-git-*-any.pkg.tar.zst
```

mellow 主题目录名：`mellow-youlan`、`mellow-sakura`、`mellow-graphite`、`mellow-wechat`、
`mellow-vermilion`（各含 `-dark` 暗色版），另有 `kwinblur-mellow-*` 模糊效果版。

启用主题：

```bash
printf 'Theme=mellow-graphite-dark\n' > ~/.config/fcitx5/conf/classicui.conf
systemctl --user restart omarchy-fcitx5.service
```

---
