# st升级日志：0.8.3 → 0.9.3

## 升级概要
- **开始时间**: 2026-09-21
- **当前版本**: 0.8.3
- **目标版本**: 0.9.3
- **升级分支**: upgrade-0.9.3
- **备份分支**: backup-0.8.3

## 已完成工作

### 1. 备份阶段
- [x] 提交当前修改到master分支
- [x] 创建backup-0.8.3分支保存完整备份
- [x] 添加upstream远程仓库
- [x] 获取0.9.3标签

### 2. 版本差异分析
- [x] 比较config.def.h差异（58行变化）
- [x] 分析12个补丁的功能和依赖关系
- [x] 创建补丁分析文档 `patches-analysis.md`

### 3. 已应用补丁

#### 3.1 st-scrollback（手动集成）
**状态**: ✅ 完成
**修改文件**:
- `st.c`: 添加HISTSIZE定义、TLINE宏、修改Term结构、实现kscrollup/kscrolldown函数
- `st.h`: 添加滚动函数声明

**关键修改**:
```c
// 添加到st.c
#define HISTSIZE      2000
#define TLINE(y)      ((y) < term.scr ? term.hist[((y) + term.histi - \
                        term.scr + HISTSIZE + 1) % HISTSIZE] : \
                        term.line[(y) - term.scr])

// 修改Term结构
typedef struct {
    // ... 其他字段
    Line hist[HISTSIZE]; /* history buffer */
    int histi;    /* history index */
    int scr;      /* scroll back */
} Term;
```

**功能**: 提供2000行滚动历史，支持Alt+u/e滚动

#### 3.2 st-alpha（手动集成）
**状态**: ✅ 完成
**修改文件**:
- `config.def.h`: 添加alpha变量
- `config.mk`: 添加-lXrender链接库
- `st.h`: 声明alpha外部变量
- `x.c`: 实现32位视觉深度和透明度渲染

**功能**: 支持背景透明度，可通过-A参数或alpha变量配置

#### 3.3 st-ligatures（手动集成）
**状态**: ✅ 完成
**修改文件**:
- `hb.c`/`hb.h`: 新增HarfBuzz集成文件
- `Makefile`: 添加hb.c编译
- `config.mk`: 添加harfbuzz依赖
- `st.h`: 添加ATTR_LIGA属性，修改ATTRCMP宏
- `win.h`: 修改xdrawcursor函数签名
- `x.c`: 集成hb.h，调用hbtransform函数

**功能**: 支持字体连字显示

#### 3.4 st-anysize（直接应用）
**状态**: ✅ 完成
**应用方式**: git apply

**功能**: 支持窗口无边框调整大小

#### 3.5 st-hidecursor（手动集成）
**状态**: ✅ 完成
**修改文件**: `x.c`
- 添加vpointer、bpointer和pointerisvisible字段到XWindow结构
- 修改bmotion、xinit、xsetpointermotion、kpress函数

**功能**: 输入时隐藏鼠标光标

#### 3.6 st-blinking_cursor（手动集成）
**状态**: ✅ 完成
**修改文件**:
- `config.def.h`: 将cursorshape重命名为cursorstyle
- `x.c`: 修改xdrawcursor函数添加闪烁支持，修改run函数添加闪烁逻辑

**功能**: 支持光标闪烁

#### 3.7 st-lukesmith-externalpipe（手动集成）
**状态**: ✅ 完成
**修改文件**:
- `config.def.h`: 添加openurlcmd、copyurlcmd、copyoutput命令和快捷键
- `st.c`: 添加TLINE_HIST宏、tlinehistlen和externalpipe函数
- `st.h`: 添加externalpipe函数声明
- `Makefile`: 添加st-copyout安装
- `st-copyout`: 复制脚本文件

**功能**: 外部管道支持（Alt+l打开URL、Alt+y复制URL、Alt+o复制输出）

#### 3.8 st-copyurl（已集成）
**状态**: ✅ 完成
**功能**: 循环复制屏幕上的URL（Alt+l）

#### 3.9 st-fontfix（已集成）
**状态**: ✅ 完成
**功能**: 字体渲染修复

#### 3.10 st-desktopentry（已集成）
**状态**: ✅ 完成
**功能**: XDG桌面条目

#### 3.2 st-alpha（手动集成）
**状态**: ✅ 完成
**修改文件**:
- `config.def.h`: 添加alpha变量
- `config.mk`: 添加-lXrender链接库
- `st.h`: 声明alpha外部变量
- `x.c`: 实现32位视觉深度和透明度渲染

**关键修改**:
```c
// config.def.h
float alpha = 0.8;

// x.c - xinit函数修改
XMatchVisualInfo(xw.dpy, xw.scr, xw.depth, TrueColor, &vis);
xw.vis = vis.visual;
xw.cmap = XCreateColormap(xw.dpy, parent, xw.vis, None);
```

**功能**: 支持背景透明度，可通过-A参数或alpha变量配置

#### 3.3 st-ligatures（手动集成）
**状态**: ✅ 完成
**修改文件**:
- `hb.c`/`hb.h`: 新增HarfBuzz集成文件（从备份复制）
- `Makefile`: 添加hb.c编译
- `config.mk`: 添加harfbuzz依赖
- `st.h`: 添加ATTR_LIGA属性，修改ATTRCMP宏
- `win.h`: 修改xdrawcursor函数签名
- `x.c`: 集成hb.h，调用hbtransform函数

**关键修改**:
```c
// st.h
#define ATTR_LIGA       1 << 11
#define ATTRCMP(a, b)   (((a).mode & (~ATTR_WRAP) & (~ATTR_LIGA)) != \
                         ((b).mode & (~ATTR_WRAP) & (~ATTR_LIGA)) || \
                         (a).fg != (b).fg || (a).bg != (b).bg)

// x.c - xdrawcursor修改
void xdrawcursor(int cx, int cy, Glyph g, int ox, int oy, 
                 Glyph og, Line line, int len)
```

**功能**: 支持字体连字显示（需要harfbuzz库）

#### 3.4 st-anysize
**状态**: ✅ 完成
**应用方式**: git apply（直接应用成功）
**功能**: 支持窗口无边框调整大小

#### 3.5 st-hidecursor
**状态**: ✅ 完成
**应用方式**: 手动集成到 x.c
**功能**: 输入时隐藏鼠标指针（vpointer/bpointer + pointerisvisible）

#### 3.6 st-blinking_cursor
**状态**: ✅ 完成
**应用方式**: 手动集成 config.def.h + x.c
**功能**: 光标闪烁支持，cursorshape 重命名为 cursorstyle（0-7 种样式）

#### 3.7 st-lukesmith-externalpipe
**状态**: ✅ 完成
**应用方式**: 手动集成 config.def.h + st.c + st.h + Makefile + st-copyout
**功能**: 外部管道（Ctrl+Alt+l 打开 URL、Alt+y 复制 URL、Alt+o 复制命令输出）
**依赖**: st-scrollback ✅

#### 3.8 st-copyurl
**状态**: ✅ 完成
**应用方式**: git apply（直接应用成功）
**功能**: 循环复制屏幕上的 URL（Alt+l）

#### 3.9 st-fontfix
**状态**: ✅ 完成
**应用方式**: git apply（直接应用成功）
**功能**: 字体渲染修复（检查 FC_COLOR 属性）

#### 3.10 st-desktopentry
**状态**: ✅ 完成
**应用方式**: 手动集成 Makefile + 新增 st.desktop
**功能**: XDG 桌面条目（安装到 /usr/share/applications）

## 待完成工作

### 4. 待应用补丁
**全部 6 个补丁已应用完成 ✅**（st-hidecursor / st-blinking_cursor / st-lukesmith-externalpipe / st-copyurl / st-fontfix / st-desktopentry）

### 5. 配置文件更新
- [x] 更新config.h，整合新功能配置（颜色/透明度/快捷键已生成）
- [x] 添加Catppuccin Mocha主题颜色（采用官方 catppuccin/st 仓库 themes/mocha.h）
- [x] 静音 erresc 调试告警（st.c 中 erresc 系列 fprintf 已注释）
- [x] 配置快捷键（默认已含 externalpipe/copyurl 等）

### 6. 测试验证
- [x] 编译测试（make clean && make 通过，无警告无错误）
- [ ] 滚动功能测试
- [ ] 透明度测试
- [ ] 连字显示测试
- [ ] 光标功能测试
- [ ] 外部管道测试

## 补丁依赖关系图

```
全部已应用 ✅:
├── st-scrollback ✅
├── st-alpha ✅
├── st-ligatures ✅
├── st-anysize ✅
├── st-hidecursor ✅
├── st-blinking_cursor ✅
├── st-lukesmith-externalpipe ✅
│   └── 依赖 st-scrollback ✅
├── st-copyurl ✅
├── st-fontfix ✅
└── st-desktopentry ✅
```

## 已知问题

1. 全部补丁已应用并编译通过（make clean && make 无错误无警告）
2. 后续可选的用户配置项：主题颜色、快捷键、alpha 值等（见第 5 节）

## 下一步行动

1. ✅ 更新 config.h：添加 Catppuccin Mocha 主题颜色、调整快捷键
2. ✅ 编译测试：make clean && make 编译成功
3. ⬜ 功能验证（滚动/透明度/连字/光标/外部管道）
4. ⬜ `make install` 重建安装
5. ⬜ 提交升级代码到 upgrade-0.9.3 分支

## 参考文档

- [补丁功能分析](patches-analysis.md)
- [中文输入配置](chinese-input.md)
- [官方st文档](https://st.suckless.org)
