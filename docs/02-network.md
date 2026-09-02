# 网络准备与下载排错

## 8. 通用网络问题与解决

**问题根源**：国内网络环境下，很多国外源（Google、GitHub 直连）不可达或极慢。

### 8.1 Go 模块下载超时

**症状**：`go build` 卡在 `go: downloading ...` 然后报错：
```
Get "https://proxy.golang.org/github.com/xxx/yyy.zip": dial tcp ...: i/o timeout
```

**原因**：Go 默认模块代理 `proxy.golang.org`（Google 服务器）在国内被墙。

**解决**：使用国内可达的 Go 代理 `goproxy.cn`，并**持久化**到 shell 配置：
```bash
echo 'export GOPROXY=https://goproxy.cn,direct' >> ~/.bashrc
export GOPROXY=https://goproxy.cn,direct
```
验证可达性：`curl -s -o /dev/null -w "%{http_code}" https://goproxy.cn`（应返回 200）

### 8.2 GitHub 文件下载慢/中断

**症状**：`curl` 下载 GitHub release 文件时速度极慢或中断，`-w` 显示 `size` 不完整、`http:000`。

**解决**：使用 `curl -C -` **断点续传**，循环执行直到下载完成：
```bash
for i in 1 2 3 4 5 6 7 8 9 10; do
  curl -sL -C - -o FILE URL -w "iter$i http:%{http_code} size:%{size_download}\n"
  sleep 1
done
```
- 下载完成后再次续传会返回 `http:416`（Range 越界，表示文件已完整）
- **验证文件完整性**：用 `od -c FILE | head -1` 检查文件头（如 mmdb 应以 `\0\0 001` 开头，ttf/ttc 应以 `ttcf` 或 `\0 001` 开头）
- 常见 ghproxy 镜像（`mirror.ghproxy.com`、`ghfast.top`、`gh-proxy.com`）大多已失效，优先用直连 + 断点续传

---
