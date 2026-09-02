# Omawrite 定制指南（字号调节 + 自适应行宽）

## 背景

Omawrite 0.5.0（`/usr/bin/omawrite`）默认**没有独立的字号设置**——文字完全跟随桌面文本缩放（`omarchy display text size` / GNOME `text-scaling-factor`），基础字号硬编码 20px。

本次基于上游两个未合并 PR（#21、#41）的思路，本地编译了定制版本，新增：

1. **编辑器字号独立调节**（`Ctrl++`/`Ctrl+-`/`Ctrl+0`，持久化到配置）
2. **内容宽度自适应窗口**（替代原来的固定 65 字符行宽）

---

## 默认字号

- 默认：**14px**（已改，原上游为 20px）
- 调节：`Ctrl++` 或 `Ctrl+=` 增大 2px，`Ctrl+-` 减小 2px，`Ctrl+0` 恢复默认
- 范围：10–48px
- 桌面文本缩放（`textScale`）仍独立叠加在基础字号之上（`最终字号 = editorFontSize × textScale`），两者互不干扰
- 持久化位置：`~/.config/Omacom/omawrite.conf` 的 `[editor] fontSize=14`

## 内容宽度（行宽）逻辑

原逻辑（上游 master）：`editorWidth = min(平均字符宽×65, 窗口宽-20字符宽)`，编辑器居中。
窗口大时内容占比过低（2560px 窗口下文字只占 21%）。

定制后逻辑：**内容宽 = min(窗口宽 - 60, 窗口宽 × 0.75)**，编辑器居中。

| 窗口宽 | 内容宽 | 左右边距 | 占比 |
|--------|--------|----------|------|
| 1300px | 975px  | 162px    | 75%  |
| 2560px | 1920px | 320px    | 75%  |

- 最小边距 30px（即内容最宽 = 窗口 - 60）
- 内容上限 75% 窗口宽
- 两约束取 `min`（75% 通常占优，窗口 <240px 时才落到 30px 边距）

---

## 源码修改点

源码位于 `~/src/omawrite`（git clone 自 `https://github.com/omacom/omawrite`，基于 PR #41 分支 `pr41` + 本地定制）。

### 1. `src/backend.cpp` — 默认字号 + 持久化

```cpp
const QString editorFontSizeSetting = QStringLiteral("editor/fontSize");
constexpr int defaultEditorFontSize = 14;   // 默认字号（原 20）
constexpr int minimumEditorFontSize = 10;
constexpr int maximumEditorFontSize = 48;
```

构造函数读取配置：
```cpp
m_editorFontSize = qBound(minimumEditorFontSize,
                          QSettings().value(editorFontSizeSetting,
                                            defaultEditorFontSize).toInt(),
                          maximumEditorFontSize);
```

`setEditorFontSize` 持久化 + 发射信号，`resetEditorFontSize` 恢复默认。

### 2. `src/backend.h` — 暴露给 QML

```cpp
Q_PROPERTY(int editorFontSize READ editorFontSize WRITE setEditorFontSize
           NOTIFY editorFontSizeChanged)
Q_INVOKABLE void resetEditorFontSize();
```

### 3. `src/Main.qml` — 字号快捷键 + 行宽

字号接入：
```qml
readonly property int editorFontPixelSize: scaledSize(backend.editorFontSize)
```

快捷键：
```qml
Shortcut {
    sequences: ["Ctrl++", "Ctrl+="]
    context: Qt.ApplicationShortcut
    onActivated: backend.editorFontSize += 2
}
Shortcut {
    sequence: "Ctrl+-"
    context: Qt.ApplicationShortcut
    onActivated: backend.editorFontSize -= 2
}
Shortcut {
    sequence: "Ctrl+0"
    context: Qt.ApplicationShortcut
    onActivated: backend.resetEditorFontSize()
}
```

行宽自适应：
```qml
readonly property int editorWidth: Math.min(
    Math.round(width - scaledSize(30) * 2),
    Math.round(width * 0.75))
```

---

## 构建

依赖：`qt6-base`、`qt6-declarative`、`gcc`、`make`、`qmake6`（均已安装）。

```bash
cd ~/src/omawrite
./bin/build
# 产物：build/omawrite
```

> 注意：若之前改过 `src/Main.qml`（QML 资源），`./bin/build` 可能因 Makefile 增量规则不重新编译资源。
> 若构建时间未更新，进 `build/` 手动 `make -j$(nproc)`。

## 替换系统版本（一键脚本）

源码已固化在 `~/src/omawrite`，并配好一键脚本 `install-custom.sh`：

```bash
cd ~/src/omawrite && ./install-custom.sh
```

脚本自动完成：构建 → 备份原版（`/usr/bin/omawrite.bak.orig`）→ 替换 `/usr/bin/omawrite`。
（需 sudo，请在终端执行。）

### 升级后如何恢复定制版

`omawrite` 是 **omarchy 仓库**的 pacman 包，`pacman -Syu` 升级后会**覆盖** `/usr/bin/omawrite` 为官方版。

**升级后处理**：只需重跑一键脚本即可恢复定制版：

```bash
cd ~/src/omawrite && ./install-custom.sh
```

源码放在 `~/src/omawrite`（pacman 不管理该目录），升级不会碰它；脚本会重新构建并替换。

## 验证

```bash
# 1. 版本替换确认
ls -la /usr/bin/omawrite          # 时间戳应为构建时间

# 2. 启动并检查
omawrite %f &
# - 默认字号 14px，Ctrl+0 重置到 14
# - Ctrl++ / Ctrl+- 2px 步进，范围 10-48
# - 内容宽度随窗口缩放（min(窗口-60, 窗口×75%)）
# - 配置持久化到 ~/.config/Omacom/omawrite.conf
```

## 相关上游 PR（未合并，可追踪）

| PR | 内容 | 说明 |
|----|------|------|
| [omacom/omawrite#21](https://github.com/omacom/omawrite/pull/21) | 浏览器式缩放（50%-300% 步进，Ctrl+滚轮） | 整体 UI 缩放，非只编辑器 |
| [omacom/omawrite#41](https://github.com/omacom/omawrite/pull/41) | 独立编辑器字号（10-48px，2px 步进） | 本定制采用此思路 |

> 上游合并后可直接用官方包替代本地定制，届时删除本定制即可。