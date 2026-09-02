# 桌面行为与显示

## 4. 触控板设置

`~/.config/hypr/input.lua`：

```lua
hl.config({
  input = {
    touchpad = {
      -- macOS 式自然（反向）滚动
      natural_scroll = true,

      -- 三指拖拽（左键按住并拖动）
      drag_3fg = 1,

      -- 关闭「轻点后拖拽」：点一下不松开再滑动不再触发选择/移动
      tap_and_drag = false,
    },
  },
})
```

### 关键坑：Hyprland Lua 配置键名

Hyprland 的 `luaConfigValueName()` 会把配置键做转换：

```
':' → '.'      '-' → '_'
```

所以原生配置里的 `tap-and-drag`，在 Lua 里必须写成 **`tap_and_drag`**（下划线）。
直接写 `tap-and-drag`（或 `["tap-and-drag"]`）会报 `unknown config key`。

另外 `hyprctl keyword` 在新解析器下不可用（报 `Use eval`），`hyprctl eval` 实际是 Lua 执行环境。

### 验证

```bash
hyprctl reload
hyprctl configerrors          # 必须无输出
hyprctl getoption input:touchpad:tap-and-drag
```

---

## 11. Chrome 中文界面

### 11.1 背景

现象：网页内容显示中文，但 Chrome 界面（菜单、右键、设置页）仍是英文。

原因：界面语言由**系统 locale 或 `--lang` 启动参数**决定，与网页语言无关；系统 locale 为 `en_US.UTF-8`，
所以界面走英文。网页语言由网页自身内容决定，因此不受影响。

### 11.2 Preferences 设置界面语言（须先完全退出 Chrome）

UI 语言键是 **`intl.app_locale`**（不是 `ui_locale`）。改动前必须**完全退出 Chrome**（含后台进程），
否则 Chrome 退出时会用内存中的配置覆盖磁盘文件：

```bash
pgrep -x chrome    # 必须无输出；有则先彻底退出
python3 << 'EOF'
import json
import os
p = os.path.expanduser('~/.config/google-chrome/Default/Preferences')
d = json.load(open(p))
i = d.setdefault('intl', {})
i['app_locale'] = 'zh-CN'          # 界面语言（关键键）
i['accept_languages'] = 'zh-CN,zh'
i['selected_languages'] = 'zh-CN,en-US,en'
i.pop('ui_locale', None)           # 无效键，顺手清掉
json.dump(d, open(p, 'w'), ensure_ascii=False)
EOF
```

### 11.3 桌面入口加 `--lang=zh-CN` + `LANGUAGE` 环境变量（持久化，推荐）

用户目录的 `.desktop` 优先于系统目录，在 `~/.local/share/applications/` 创建覆盖版即可：

```bash
mkdir -p ~/.local/share/applications
sed -e 's|Exec=/usr/bin/google-chrome-stable %U|Exec=env LANGUAGE=zh_CN:zh /usr/bin/google-chrome-stable --lang=zh-CN %U|' \
    -e 's|Exec=/usr/bin/google-chrome-stable$|Exec=env LANGUAGE=zh_CN:zh /usr/bin/google-chrome-stable --lang=zh-CN|' \
    -e 's|Exec=/usr/bin/google-chrome-stable --incognito|Exec=env LANGUAGE=zh_CN:zh /usr/bin/google-chrome-stable --incognito --lang=zh-CN|' \
    /usr/share/applications/google-chrome.desktop > ~/.local/share/applications/google-chrome.desktop
```

覆盖三个入口：普通启动（`%U`）、新窗口、无痕窗口。

> **关键坑：`--lang` 单独不够，必须配 `LANGUAGE` 环境变量。**
> 桌面启动器（systemd app scope）不传 locale 环境变量时，即使命令行带 `--lang=zh-CN`，
> Chrome 仍解析为英文（主进程 cmdline 有 `--lang=zh-CN`，但子进程 renderer 的 `--lang` 是 `en-US`）。
> 加上 `LANGUAGE=zh_CN:zh` 后子进程才变成 `zh-CN`。系统无需安装 `zh_CN.UTF-8` glibc locale，
> 只设 `LANGUAGE` 即可（不要设 `LANG`，本机未生成该 locale）。

### 11.4 关键坑：必须彻底杀干净旧进程

`--lang` 只在**没有残留进程**时生效。若 Chrome 已在运行，新参数会被忽略——新实例只是把请求
转交给旧进程后退出。改完配置后：

```bash
pkill -9 -x chrome; sleep 2; pgrep -f chrome    # 确认无输出
nohup env LANGUAGE=zh_CN:zh google-chrome-stable >/dev/null 2>&1 &   # 从菜单启动亦可
ps -eo args | grep "type=renderer" | grep -v grep | head -1 | grep -o "lang=[^ ]*"   # 应显示 zh-CN
```

### 11.5 验证

- 启动后菜单、右键均为中文即成功。
- 中文语言包已随 Chrome 内置（`/opt/google/chrome/locales/zh-CN.pak`），无需联网下载。
- Figma（`figma.sh`）走独立启动命令，如需界面也中文，可在其 `/opt/google/chrome/chrome` 后追加 `--lang=zh-CN`。

### 11.6 关闭 Chrome QUIC（配合 TUN/HTTP 代理）

Chrome 的 QUIC 使用 UDP/443，不走普通 HTTP/HTTPS 代理。在 Mihomo 同时启用系统代理与 TUN 时，可能导致
同一网站的 TCP 与 UDP 请求采用不同路径。用 Chrome 管理策略持久关闭 QUIC：

```bash
cat > /tmp/mihomo-chrome-policy.json <<'EOF'
{
  "QuicAllowed": false
}
EOF
pkexec install -o root -g root -m 0644 \
  /tmp/mihomo-chrome-policy.json /etc/opt/chrome/policies/managed/mihomo-proxy.json
rm -f /tmp/mihomo-chrome-policy.json
```

完全退出并重新启动 Chrome 后生效。打开 `chrome://policy`，应看到 `QuicAllowed = false` 且状态正常。
该设置只关闭 Chrome 的 QUIC 传输，不改变 Mihomo 的规则或策略组选择。

---
