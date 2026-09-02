# Mihomo 代理

## 9. Mihomo 代理核心 + 控制面板插件

### 9.1 安装 Omarchy Mihomo 控制面板插件

```bash
omarchy plugin add https://github.com/lijiawei0305-pixel/omarchy-mihomo-plugin.git --enable --yes
omarchy bar move io.github.lijiawei0305-pixel.mihomo --section right
```

### 9.2 安装 Mihomo 核心（AUR）

- 用 `omarchy pkg aur add mihomo` 安装，会自动构建 `mihomo`（可能顺带 `clash-geoip`，但**不要依赖**它的 `/etc/clash/Country.mmdb`，详见[配置步骤](#93-配置-mihomo-订阅并启动服务)和[排错](#94-常见故障与排错mihomo-内核起不来--断网)）。
- **注意**：该包依赖 `go` 编译（耗时较长，需耐心等待）。
- **可能失败点**：Go 依赖下载超时（见[网络准备](02-network.md#81-go-模块下载超时)）。若失败，需在构建目录手动设置 `GOPROXY` 后重新构建。

**手动构建（网络问题时的备用方案）**：
```bash
cd ~/.cache/yay/mihomo/src/mihomo-<版本>/
export GOPROXY=https://goproxy.cn,direct
GOOS=linux go build -trimpath -buildmode=pie -tags with_gvisor -o /tmp/mihomo-test
```

**手动安装（绕过 yay 交互式 sudo 的问题）**：
> 注意：非交互环境下 `sudo`/`pkexec` 弹窗可能无法确认。`pkexec` 是图形化认证，在用户桌面有会话时可正常弹窗。
```bash
# 安装脚本示例：安装二进制、systemd 服务、sysusers、tmpfiles、config、符号链接
install -Dm755 /tmp/mihomo-test /usr/bin/mihomo
ln -sf /usr/bin/mihomo /usr/bin/clash-meta
install -Dm644 mihomo.service /usr/lib/systemd/system/
install -Dm644 mihomo.sysusers /usr/lib/sysusers.d/mihomo.conf
install -Dm644 mihomo.tmpfiles /usr/lib/tmpfiles.d/mihomo.conf
systemd-sysusers /usr/lib/sysusers.d/mihomo.conf
systemd-tmpfiles --create /usr/lib/tmpfiles.d/mihomo.conf
# 注意：不要用 ln -sf 链接 /etc/clash/Country.mmdb —— clash-geoip 包可能未安装、
# /etc/clash 目录可能不存在，会留下死符号链接导致 mihomo 启动即崩（见 10.4 排错）。
# Geo 数据统一按 10.3 第 2 步下载为实体文件放到 /etc/mihomo/。
```

### 9.3 配置 Mihomo 订阅并启动服务

1. **下载订阅配置**：**需用户提供** clash 订阅链接。先下载到临时文件，**不要直接覆盖**正在使用的配置：
   ```bash
   pkexec install -d -o root -g root -m 755 /etc/mihomo
   curl -fsSL -o /tmp/mihomo-subscription.yaml "<订阅链接>"
   ```

   **必须保留/补入域名嗅探配置**。Chrome 安全 DNS 可能让 TUN 收到的连接只剩真实 IP；没有 `sniffer` 时，
   `DOMAIN-SUFFIX` 规则会失效并落入“漏网之鱼”，同一网站的页面、API、图片可能走不同出口。把下面配置放在顶层
   （建议位于 `tun:` 后、`proxies:` 前），不要改写原订阅的策略组和规则：

   ```yaml
   sniffer:
     enable: true
     parse-pure-ip: true
     override-destination: true
     sniff:
       HTTP:
         ports: [80, 8080-8880]
       TLS:
         ports: [443, 8443]
       QUIC:
         ports: [443, 8443]
   ```

   > **订阅更新原则**：新增或更换订阅时，上游配置可能覆盖本地顶层配置。每次更新都必须先写临时文件，
   > 合并上述 `sniffer` 后再校验和安装。不要为了修复某个网站把它改到另一策略组；嗅探恢复域名后，
   > 应继续命中订阅原有规则（例如 `x.com → 🚀 节点选择`），保持原来的分流设计。
2. **Geo 数据**：配置里用了 `GEOIP,CN` 和 `IP-ASN` 规则，需要 `Country.mmdb` 和 `ASN.mmdb`。**两者都从 MetaCubeX release 直接下载为实体文件**放到 `/etc/mihomo/`：
   - 不要依赖 `clash-geoip` 包的 `/etc/clash/Country.mmdb`（可能未安装），也不要用符号链接（会留下死链接，见[排错](#94-常见故障与排错mihomo-内核起不来--断网)）。
   - 下载可用 `curl -C -` 断点续传（见[网络准备](02-network.md#82-github-文件下载慢中断)），下载后**必须校验文件头**（mmdb 以 `\0\0 001` 开头）：
     ```bash
     # Country.mmdb（约 7.5MB，meta-rules-dat 定制版）
     curl -L -o "$HOME/Country.mmdb" "https://github.com/MetaCubeX/meta-rules-dat/releases/download/latest/country.mmdb"
     # ASN.mmdb（约 12MB，GeoLite2）
     curl -L -o "$HOME/ASN.mmdb" "https://github.com/MetaCubeX/meta-rules-dat/releases/download/latest/GeoLite2-ASN.mmdb"
     od -A x -t x1 -N 4 "$HOME/Country.mmdb"   # 应输出 "00 00 01 xx"
     pkexec install -Dm644 "$HOME/Country.mmdb" /etc/mihomo/Country.mmdb
     pkexec install -Dm644 "$HOME/ASN.mmdb" /etc/mihomo/ASN.mmdb
     rm -f "$HOME/Country.mmdb" "$HOME/ASN.mmdb"
     ```
3. **验证配置并安全替换**：
   ```bash
   mihomo -f /tmp/mihomo-subscription.yaml -t
   # 输出 "configuration file ... test is successful" 表示成功

   # 已有配置则先备份，再安装通过校验的新配置
   pkexec cp --preserve=mode,ownership /etc/mihomo/config.yaml \
     "/etc/mihomo/config.yaml.bak-$(date +%Y%m%d-%H%M%S)"
   pkexec install -o root -g root -m 0644 \
     /tmp/mihomo-subscription.yaml /etc/mihomo/config.yaml
   ```
4. **启动服务**：
   ```bash
   systemctl daemon-reload
   systemctl enable --now mihomo
   ```
5. **验证**：
   ```bash
   systemctl status mihomo
   curl http://127.0.0.1:9090/version   # 返回 {"meta":true,...} 表示控制器正常
   curl -fsS http://127.0.0.1:9090/configs | jq '.sniffing'  # 必须为 true
   ```

**关键配置点**：
- 面板通过 `external-controller`（默认 `127.0.0.1:9090`）连接核心。若订阅配置里已有该行则无需额外配置。
- 面板探测核心的顺序：`~/.config/omarchy-mihomo/config` → 核心启动参数 → yaml → 默认 `127.0.0.1:9090`。
- 若端口/secret 非默认，创建 `~/.config/omarchy-mihomo/config`：
  ```
  endpoint = 127.0.0.1:9090
  secret = your-secret
  ```
- 面板默认**不**自动开启系统代理/TUN，需在面板 Home 页手动开启。
- **强烈建议在配置里加 `dns:` 块**（fake-ip + 普通 UDP DNS，见[故障四](#94-常见故障与排错mihomo-内核起不来--断网)），否则走系统 DNS 时 YouTube/Twitter 会被污染解析导致时通时不通。
- **必须启用 `sniffer`**：TUN 有效不等于域名规则一定能识别流量。运行时用
  `curl -fsS http://127.0.0.1:9090/configs | jq '.sniffing'` 检查，必须返回 `true`。

### 9.4 常见故障与排错（mihomo 内核起不来 / 断网）

> 本节来自真实故障：`Country.mmdb` 死符号链接导致内核启动即崩，叠加系统代理残留导致整机断网。
> 现象与排错步骤完整记录如下，重装/故障时照此处理。

**故障一：面板提示"未连接内核"，且 `systemctl status mihomo` 反复重启（activating/失败）**

- **现象**：Omarchy 面板顶部显示未连接内核；`9090` 端口无监听；`ss -tlnp | grep 9090` 无输出。
- **根因**：`/etc/mihomo/Country.mmdb` 是**死符号链接**（指向 `/etc/clash/Country.mmdb`，但该文件不存在）。配置里含 `GEOIP,CN` 规则需要 GeoIP 库，加载时 mihomo 尝试下载失败 → `fatal: Parse config error` → 进程退出码 1 → systemd `Restart=always` 每 5 秒重启，反复失败。
- **诊断**：
  ```bash
  journalctl -u mihomo -n 20 --no-pager
  # 关键报错：can't initial GeoIP: can't download MMDB: open /etc/mihomo/Country.mmdb: no such file
  ls -la /etc/mihomo/Country.mmdb
  # 看到 -> /etc/clash/Country.mmdb 且目标不存在 = 死链接
  readlink /etc/mihomo/Country.mmdb; ls -la /etc/clash/Country.mmdb
  ```
- **修复**：下载 GeoIP 库覆盖为**实体文件**（不是链接）：
  ```bash
  sudo systemctl stop mihomo
  sudo rm -f /etc/mihomo/Country.mmdb        # 删除死链接
  curl -L -o /tmp/country.mmdb \
    "https://github.com/MetaCubeX/meta-rules-dat/releases/download/latest/country.mmdb"
  # 校验文件头：od -A x -t x1 -N 4 /tmp/country.mmdb 应输出 "00 00 01 xx"（断点续传见 9.2）
  sudo install -o mihomo -g mihomo -m 644 /tmp/country.mmdb /etc/mihomo/Country.mmdb
  rm -f /tmp/country.mmdb
  sudo systemctl reset-failed mihomo
  sudo systemctl enable --now mihomo
  ```
- **验证**：
  ```bash
  systemctl is-active mihomo                  # active
  ss -tlnp | grep -E ':(7890|7891|9090)'      # 三端口监听
  curl -fsS http://127.0.0.1:9090/version     # {"meta":true,...}
  curl -x http://127.0.0.1:7890 -fsSI https://www.baidu.com  # 走代理连通
  ```

**故障二：整机无法上网（网页全打不开），但 `curl --noproxy '*'` 直连正常**

- **现象**：所有浏览器/应用无法访问网页；`curl -fsSI https://www.baidu.com` 卡住或超时，加 `--noproxy '*'` 后正常。
- **根因**：系统代理残留。Omarchy 面板开启"系统代理"后，会在 `~/.config/environment.d/99-mihomo-proxy.conf` 写入 `http_proxy/https_proxy/all_proxy` 指向 `127.0.0.1:7890`。当 mihomo 内核未运行（故障一的状态）时，这些变量仍把**所有**流量导向不存在的 7890 端口 → 全系统断网。
- **诊断**：
  ```bash
  env | grep -i proxy                      # 会话里有残留代理变量
  cat ~/.config/environment.d/99-mihomo-proxy.conf   # 含 http_proxy=... 即残留
  ss -tlnp | grep 7890                     # 无输出 = 内核没监听，流量走死端口
  ```
- **修复**：关闭所有系统级代理（备份原文件），并防止插件重启时重新写回：
  ```bash
  # 1) 清环境变量 + gsettings
  systemctl --user unset-environment http_proxy https_proxy all_proxy no_proxy HTTP_PROXY HTTPS_PROXY ALL_PROXY NO_PROXY
  gsettings set org.gnome.system.proxy mode 'none'
  # 2) 注释掉 drop-in 里的代理行（或删除该文件）
  sed -i -E '/^(http_proxy|https_proxy|all_proxy|no_proxy|HTTP_PROXY|HTTPS_PROXY|ALL_PROXY|NO_PROXY)=/ s/^/# disabled: /' \
    ~/.config/environment.d/99-mihomo-proxy.conf
  # 3) 插件状态改为 off，避免下次启动重新写回代理
  sed -i 's/^sysproxy = on$/sysproxy = off/' ~/.config/omarchy-mihomo/ui
  # 4) 修复内核（故障一）后再恢复代理
  ```
- **注意**：已在运行的应用仍持有旧环境变量，彻底生效需重启桌面会话或重启系统；`systemctl --user unset-environment` 只影响之后启动的应用。

**故障三：配置明明存在，面板却显示"没有配置"**

- **原因**：面板通过 `9090` 控制器 API 读配置，内核起不来（故障一）时读取不到 → 显示空白/无配置。
- **处理**：先修内核（故障一），配置路径 `/etc/mihomo/config.yaml` 本身无需改动。

**故障四：部分网站（YouTube/Twitter）时通时不通，其他国外站正常**

- **现象**：Google/Instagram 能翻，但 YouTube/Twitter 反复超时或偶尔能开；`curl -x http://127.0.0.1:7890 https://www.youtube.com` 超时。
- **根因**：订阅配置**缺少 `dns:` 配置块**时，mihomo 走系统 DNS（运营商 DNS），对敏感域名返回**污染假地址**（如 `www.youtube.com` → `2001::1`、`69.171.235.22`、`0.0.0.x`），导致连接假 IP 超时。不敏感的域名（Google 主域、Instagram 等）恰好解析正常，所以表现为"部分能翻"。
- **诊断**：
  ```bash
  # 1) 系统 DNS 解析是否被污染（对比真实 IP 段）
  getent ahostsv4 www.youtube.com     # 出现 2001::1 / 69.171.x.x(Facebook) / 0.0.0.x = 污染
  # 2) 配置里有没有 dns 块
  grep -c "^dns:" /etc/mihomo/config.yaml    # 0 = 缺 DNS 配置
  # 3) 节点本身是否正常（delay 测试能通就排除节点问题）
  curl -fsS "http://127.0.0.1:9090/proxies/%E9%A6%99%E6%B8%AF%2001/delay?url=https%3A%2F%2Fwww.gstatic.com%2Fgenerate_204&timeout=5000"
  ```
- **修复**：给配置加上 `dns:` 块（fake-ip 模式 + 普通 UDP DNS），让国外域名解析交给代理节点，绕开被污染的系统 DNS：
  ```yaml
  dns:
    enable: true
    ipv6: false
    enhanced-mode: fake-ip
    fake-ip-range: 198.18.0.1/16
    fake-ip-filter:
      - '*.lan'
      - '*.local'
      - '+.msftconnecttest.com'
      - '+.msftncsi.com'
      - '+.pool.ntp.org'
    default-nameserver:
      - 223.5.5.5
      - 119.29.29.29
    nameserver:
      - 223.5.5.5
      - 119.29.29.29
    proxy-server-nameserver:
      - 223.5.5.5
    nameserver-policy:
      "geosite:cn":
        - 223.5.5.5
        - 119.29.29.29
  ```
  - **注意**：不要用 `dns.google` / `cloudflare-dns.com` 这类 DoH 做 nameserver/fallback —— 它们在国内被墙（连接被 reset），会导致 mihomo 连节点服务器域名都解析不了，**所有**走代理流量全挂（表现为 `delay 测试 503`）。用普通 UDP DNS（223.5.5.5 等）即可，fake-ip 会把国外域名解析交给节点完成。
  - `nameserver-policy` 里的 `geosite:cn` 需要 `GeoSite.dat`（约 4.2MB），mihomo 会自动下载，但若网络不稳会下载到损坏文件（见下）。
- **验证**：`systemctl restart mihomo` 后走代理测 YouTube 应稳定 200。

**故障五：`GeoSite.dat` 反复下载到损坏文件，内核起不来**

- **现象**：加了 `geosite:cn` 的 DNS 配置后，日志出现 `failed to decode geosite file: GeoSite.dat`、`cannot parse invalid wire-format data`，端口一直没监听，服务反复重启。
- **根因**：mihomo 自动下载 GeoSite.dat 时网络中断，得到几十 KB 的损坏文件；每次启动检测到损坏又重下，恶性循环。
- **修复**：手动放一个完整文件（或先去掉 `geosite:cn` 那行配置）：
  ```bash
  sudo systemctl stop mihomo
  sudo install -o mihomo -g mihomo -m 644 /path/to/valid/GeoSite.dat /etc/mihomo/GeoSite.dat
  sudo systemctl reset-failed mihomo
  sudo systemctl start mihomo
  ```
  完整 GeoSite.dat 约 4.2MB，来源：`https://github.com/MetaCubeX/meta-rules-dat/releases/download/latest/geosite.dat`（断点续传见 9.2）。

**排查陷阱：`curl --noproxy '*'` 会忽略 `-x` 代理**

- **现象**：用 `curl --noproxy '*' -x http://127.0.0.1:7890 https://www.google.com` 测试一直超时，误判为"代理失效/节点挂/出口是中国 IP"，实际是 `--noproxy '*'` 让 curl 对所有域名绕过代理直连，直连被墙域名自然超时。
- **正确做法**：测代理连通性时**不要加 `--noproxy '*'`**，直接用 `curl -x http://127.0.0.1:7890 https://www.google.com`；要绕过代理用 `--noproxy '*'`，两者不可同时用于同一目标。
- **出口 IP 验证**：`curl -x http://127.0.0.1:7890 -fsS http://ip-api.com/json/` 返回的 `query` 字段应是非中国 IP（如台湾/香港/新加坡）才算真正走代理。

**故障六：Chrome 打开 X 没样式，登录时报 “Something went wrong”**

- **现象**：显式代理测试 X 返回 200，但 Chrome 页面只有 HTML 骨架、一直 `Loading…`，点击登录后报错；
  X 可能附带“隐私扩展”提示，即使没有第三方扩展也会出现。
- **根因**：Chrome 安全 DNS 得到真实 IP 后，部分流量经 TUN 进入 Mihomo 时只有 IP。若未开启 `sniffer`，
  `x.com`、`api.x.com`、`twimg.com` 的域名规则无法命中，请求落入“漏网之鱼”，造成同一会话出口不一致。
  Chrome 的 QUIC（UDP/443）还可能绕过普通 HTTP 代理，加重问题。
- **诊断**：
  ```bash
  curl -fsS http://127.0.0.1:9090/configs | jq '.sniffing'  # false = 未启用嗅探
  journalctl -u mihomo --since '-5 min' --no-pager | \
    grep -E 'x\.com|twitter|twimg|104\.244|199\.59'
  # 故障特征：X 的真实 IP 显示 match Match using 🐟 漏网之鱼
  ```
- **修复**：按[配置步骤](#93-配置-mihomo-订阅并启动服务)加入 `sniffer`，校验后重启 Mihomo；不要把 X
  写死到美国或其他节点，保留订阅原有的 `🚀 节点选择` 语义。
- **验证**：刷新 X 后日志应显示类似：
  ```text
  api.x.com:443 match DomainSuffix(x.com) using 🚀 节点选择[当前节点]
  abs.twimg.com:443 match DomainSuffix(twimg.com) using 🚀 节点选择[当前节点]
  ```
- **Chrome QUIC 持久化措施**：见[桌面行为与显示](06-desktop-behavior.md#116-关闭-chrome-quic配合-tunhttp-代理)。

---
