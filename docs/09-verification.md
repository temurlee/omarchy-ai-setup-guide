# 验证清单与关键路径

完成默认流程后，智能体应逐项报告结果，不要只报告命令已执行。

## 最终检查

- `hyprctl configerrors` 没有新增错误。
- fcitx5 用户服务正常，Rime 可在目标应用中输入中文。
- Mihomo 配置校验通过、服务正常，且订阅地址和 secret 未写入仓库。
- 通知中心与剪贴板服务正常。
- Web App 图标和启动命令有效。
- Chrome、Figma 和微信仅检查用户选择安装的项目。
- 修改过的原文件均有备份，临时下载和测试文件已说明或清理。
- `optional/` 中未获用户明确同意的项目没有执行。

## 回滚记录

智能体应在最终报告中列出每个备份文件的路径，以及恢复它所需的命令。若某项没有自动回滚方式，应明确说明。

## 14. 附录：本机关键路径

| 项目 | 路径 |
|------|------|
| Mihomo 二进制 | `/usr/bin/mihomo` |
| Mihomo 配置 | `/etc/mihomo/config.yaml` |
| Mihomo 配置备份 | `/etc/mihomo/config.yaml.bak-*` |
| Mihomo Geo 数据 | `/etc/mihomo/Country.mmdb`、`/etc/mihomo/ASN.mmdb` |
| Mihomo 服务 | `systemctl status mihomo` |
| Mihomo 控制器 | `127.0.0.1:9090` |
| Chrome/Mihomo 策略 | `/etc/opt/chrome/policies/managed/mihomo-proxy.json` |
| 面板插件 | `~/.config/omarchy/plugins/io.github.lijiawei0305-pixel.mihomo/` |
| 面板核心探测 | `~/.config/omarchy/plugins/.../bin/mihomo-ctl endpoint` |
| 通知中心插件 | `~/.config/omarchy/plugins/shavanced.notification-center/` |
| figma-agent 服务 | `systemctl --user status figma-agent.{socket,service}`（监听 `127.0.0.1:44950`） |
| Figma 启动脚本 | `~/.local/bin/figma.sh` |
| 微软雅黑字体 | `~/.local/share/fonts/msyh.ttc`、`~/.local/share/fonts/msyhbd.ttc` |
| 微信桌面入口（HiDPI 修复） | `~/.local/share/applications/wechat-universal.desktop`（`QT_SCALE_FACTOR=1.75`） |
| 微信数据目录 | `~/Documents/WeChat_Data` |
| 微信启动/停止 | `/usr/lib/wechat-universal/start.sh`、`/usr/lib/wechat-universal/stop.sh` |
| 飞书桌面入口（IME 修复） | `~/.local/share/applications/Feishu.desktop`（`XDG_CURRENT_DESKTOP=KDE`） |
| ChatGPT 启动脚本 | `/usr/bin/chatgpt`（自动加 `--ozone-platform=wayland`） |
| ChatGPT flags | `~/.config/codex-flags.conf`（勿用 `--ozone-platform=x11`） |
| ChatGPT 桌面入口 | `/usr/share/applications/chatgpt.desktop`（IME 修复用 `~/.local/share/applications/` 覆盖） |
| fcitx5 用户环境 | `~/.config/environment.d/10-omarchy-fcitx.conf`（补了 `GTK_IM_MODULE=fcitx`） |
