# st 中文输入（fcitx5 / XIM）

本文记录一次「st 内无法输入中文」的问题排查与修复。问题不在 st 源码，而在 **X11 输入法环境变量与 fcitx5 的 XIM 注册名不一致**。

## 现象

- 使用自编译的 st（`/usr/local/bin/st`，源码在本仓库）
- 系统输入法为 **fcitx5**（如拼音）
- 浏览器、GTK 等应用中文输入正常
- **仅在 st 中**无法调出输入法或无法提交中文

## 原因分析

### st 如何接收非 ASCII 输入

st 通过 **XIM（X Input Method）** 与输入法通信，相关逻辑在 `x.c`：

- 启动时：`ximopen()` → `XOpenIM()` / `XCreateIC()`
- 按键时：`XmbLookupString()` 从输入法获取组合后的字符串

```c
/* x.c — 简化说明 */
xw.ime.xim = XOpenIM(xw.dpy, NULL, NULL, NULL);
/* ... */
len = XmbLookupString(xw.ime.xic, e, buf, sizeof buf, &ksym, &status);
```

`XOpenIM()` 会读取环境变量 **`XMODIFIERS`**，按 `@im=<名称>` 查找对应的 XIM 服务。

### 根本原因：名称写错

当时环境配置为：

```bash
export XMODIFIERS="@im=fcitx5"
export GTK_IM_MODULE=fcitx5
export QT_IM_MODULE=fcitx5
```

而 **fcitx5 守护进程注册的 XIM 服务名是 `fcitx`，不是 `fcitx5`**。

可用以下命令自行确认：

```bash
# 应看到与 fcitx 相关的 XIM 服务
DISPLAY=:0 xprop -root XIM_SERVERS

# 诊断工具会提示 XMODIFIERS 与 XIM 服务名不匹配
fcitx5-diagnose
```

因此 st 的 `XOpenIM()` 找不到名为 `fcitx5` 的 XIM 服务，输入法无法连接，中文输入失效。

> **说明**：后台运行的仍是 **fcitx5** 程序；`fcitx` 是客户端模块 / XIM 注册使用的**兼容名称**。Arch Wiki 与 fcitx5 官方文档均推荐客户端使用 `@im=fcitx`，而非 `@im=fcitx5`。

### 为何其他程序正常、只有 st 出问题

| 应用类型 | 常用协议 | 是否受错误 `XMODIFIERS` 影响 |
|----------|----------|------------------------------|
| 浏览器、GTK/Qt（fcitx5-gtk/qt） | dbus / fcitx5 前端 | 通常仍可用 |
| **st** | **仅 XIM** | **直接受影响** |

st 没有实现 fcitx5 的 dbus 前端，只依赖 XIM，所以对 `XMODIFIERS` 最敏感。

## 修复方式

将三个环境变量改为 fcitx5 推荐的 **fcitx** 名称（无需改 st 源码、无需重新打补丁）：

```bash
export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
export XMODIFIERS="@im=fcitx"
```

### 已修改的配置文件（会话级）

以下文件在 2026-05 排查时已更新；若你重装系统或复制配置，请保持一致：

| 文件 | 说明 |
|------|------|
| `~/.xprofile` | 图形登录 / 部分显示管理器会加载 |
| `~/.xinitrc` | `startx` **不会**读 `.xprofile`，须在 `exec dwm` 等之前 export |
| `~/dwm/autostart.sh` | 在启动 `fcitx5 &` 之前 export |
| `~/.config/zsh/env.zsh` | 新开的 shell / st 继承环境 |
| `~/.config/tmux/tmux.conf` | `update-environment` 中包含 `GTK_IM_MODULE`、`XMODIFIERS`，避免 tmux 子进程丢失变量 |

### 使配置生效

1. **注销并重新登录**（或重启 X 会话）
2. **新开** st 窗口（旧窗口可能仍携带错误环境）
3. 在 st 内用 fcitx5 切换键（本机为 **Ctrl+Space**）切换到中文并输入

### 快速验证（无需完整重登）

在已有终端中：

```bash
export GTK_IM_MODULE=fcitx QT_IM_MODULE=fcitx XMODIFIERS='@im=fcitx'
st
```

若此窗口内中文正常，说明 st 本身与 XIM 代码无问题，只需保证会话环境正确。

## 与本仓库 st 源码的关系

- 本仓库 **已包含** 上游 suckless st 的 XIM 实现，**不需要** 为中文单独打 IME 补丁。
- 若中文仍不可用，优先检查：
  1. `fcitx5` 是否在运行：`pgrep -a fcitx5`
  2. 环境变量是否为 `@im=fcitx`
  3. `DISPLAY` 是否正确（远程 SSH 无 X 时无法使用 XIM）
  4. 是否在 **旧 st 进程** 中测试（应新开窗口）

## 参考

- [Arch Wiki — Fcitx5](https://wiki.archlinux.org/title/Fcitx5)
- [Fcitx5 — Setup XIM](https://fcitx-im.org/wiki/Setup_Fcitx5/en#XIM)
- st 源码：`x.c` 中 `ximopen`、`kpress`（`XmbLookupString`）

## 修订记录

| 日期 | 说明 |
|------|------|
| 2026-05-26 | 初稿：记录 fcitx5 / XMODIFIERS 名称不匹配导致 st 无法输入中文 |
