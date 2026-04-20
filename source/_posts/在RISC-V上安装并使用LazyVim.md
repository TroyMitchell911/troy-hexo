---
title: 在RISC-V上安装并使用LazyVim
date: 2026-04-20 17:09:46
tags:
  - RISC-V
  - Neovim
  - LazyVim
  - LuaJIT
---


## 背景

LazyVim 是基于 lazy.nvim 的 Neovim 预配置发行版，提供了开箱即用的现代编辑器体验。然而，lazy.nvim 强制要求 Neovim 必须使用 LuaJIT 构建，而官方 LuaJIT 并不支持 RISC-V 架构。

本文记录了在 RISC-V Linux 环境（Ubuntu/Debian）上，从零开始编译安装 Neovim + LazyVim 的完整流程，包括所有踩坑和解决方案。

测试环境：
- 架构：riscv64gc
- 系统：Ubuntu (bianbu/resolute)
- 板卡：K3 开发板

---

## 1. 编译安装 RISC-V LuaJIT

官方 LuaJIT 不支持 RISC-V，但社区有一个活跃的移植分支：[IgnotaYun/LuaJIT](https://github.com/IgnotaYun/LuaJIT)（`riscv` 分支）。该项目持续维护，CI 全绿，可用于生产。

### 1.1 安装编译依赖

```bash
sudo apt install -y \
  build-essential \
  git
```

### 1.2 编译 LuaJIT

```bash
git clone -b riscv https://github.com/IgnotaYun/LuaJIT.git ~/luajit-riscv
cd ~/luajit-riscv
make -j$(nproc)
sudo make install
```

### 1.3 配置动态链接

如果 LuaJIT 安装到了 `/usr/local/lib`，需要让系统链接器能找到它：

```bash
echo "/usr/local/lib" | sudo tee /etc/ld.so.conf.d/luajit.conf
sudo ldconfig
```

### 1.4 验证

```bash
luajit -v
# 应输出类似：LuaJIT 2.1.xxxxxxxxxx -- Copyright (C) 2005-2025 Mike Pall. https://luajit.org/
```

---

## 2. 编译安装 Neovim

### 2.1 安装编译依赖

```bash
sudo apt install -y \
  ninja-build \
  gettext \
  libtool \
  libtool-bin \
  autoconf \
  automake \
  cmake \
  g++ \
  pkg-config \
  unzip \
  curl \
  git
```

### 2.2 获取源码

```bash
git clone https://github.com/neovim/neovim.git ~/neovim
cd ~/neovim
git checkout stable  # 或指定 tag，如 v0.12.1
```

### 2.3 编译第三方依赖

关键：使用 `-DUSE_BUNDLED_LUAJIT=OFF` 跳过 Neovim 自带的 LuaJIT 下载，改用系统安装的 RISC-V LuaJIT。

```bash
cmake -S cmake.deps -B .deps \
  -DUSE_BUNDLED_LUAJIT=OFF

cmake --build .deps -j$(nproc)
```

### 2.4 编译 Neovim 本体

```bash
cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=Release

cmake --build build -j$(nproc)
```

如果 cmake 找不到系统 LuaJIT，手动指定路径：

```bash
cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -DLUAJIT_INCLUDE_DIR=/usr/local/include/luajit-2.1 \
  -DLUAJIT_LIBRARY=/usr/local/lib/libluajit-5.1.so
```

可以用以下命令确认路径：

```bash
find /usr -name "luajit.h" 2>/dev/null
find /usr -name "libluajit*" 2>/dev/null
```

### 2.5 安装

```bash
sudo cmake --install build
```

### 2.6 验证

```bash
nvim --version
```

输出中应包含 `LuaJIT 2.1.xxx` 字样，说明 Neovim 已正确链接 RISC-V LuaJIT。

---

## 3. 安装 LazyVim

参考官方文档：https://www.lazyvim.org/installation

### 3.1 备份已有配置（如有）

```bash
mv ~/.config/nvim{,.bak}
mv ~/.local/share/nvim{,.bak}
mv ~/.local/state/nvim{,.bak}
mv ~/.cache/nvim{,.bak}
```

### 3.2 克隆 LazyVim Starter

```bash
git clone https://github.com/LazyVim/starter ~/.config/nvim
rm -rf ~/.config/nvim/.git
```

### 3.3 启动 Neovim

```bash
nvim
```

首次启动时 lazy.nvim 会自动下载并安装所有插件。

---

## 4. 解决 tree-sitter-cli 缺失问题

### 4.1 问题现象

启动 nvim 后，nvim-treesitter 报错：

```
❌ tree-sitter (CLI)
Failed to install tree-sitter-cli with mason
```

### 4.2 原因

tree-sitter-cli 通过 Mason 安装时会下载预编译二进制，但官方没有提供 RISC-V 架构的预编译包。

### 4.3 解决方案：通过 Rust/cargo 从源码编译

首先安装 Rust（如果还没有）：

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source ~/.cargo/env
```

在 RISC-V 上，Rust 的 bindgen 会遇到 clang target triple 不匹配的问题，需要安装依赖并设置环境变量：

```bash
sudo apt install -y libclang-dev clang llvm
```

```bash
# 加入 ~/.zshrc 或 ~/.bashrc，以后所有 cargo build 都需要
export BINDGEN_EXTRA_CLANG_ARGS="--target=riscv64-linux-gnu -I/usr/include -I/usr/include/riscv64-linux-gnu"
```

然后编译安装：

```bash
cargo install tree-sitter-cli
```

验证：

```bash
tree-sitter --version
```

---

## 5. 解决 lua_ls (Lua Language Server) 安装失败

### 5.1 问题现象

Mason 报错无法安装 lua_ls。

### 5.2 原因

与 tree-sitter-cli 相同，Mason 下载预编译二进制，没有 RISC-V 版本。

### 5.3 解决方案：从源码编译 lua_ls

```bash
cd ~
git clone --recurse-submodules https://github.com/LuaLS/lua-language-server.git
cd lua-language-server
./make.sh
```

将编译产物加入 PATH：

```bash
echo 'export PATH="$HOME/lua-language-server/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

然后在 LazyVim 配置中告诉 nvim-lspconfig 不要通过 Mason 管理 lua_ls：

```lua
-- ~/.config/nvim/lua/plugins/lsp.lua
return {
  {
    "neovim/nvim-lspconfig",
    opts = {
      servers = {
        lua_ls = {
          mason = false,
        },
      },
    },
  },
}
```

---

## 6. 解决 blink.cmp 报错

### 6.1 问题现象

启动 nvim 时报错：

```
blink.cmp  ENOENT: no such file or directory: ...
blink.cmp  Falling back to Lua implementation due to error while downloading pre-built binary
```

### 6.2 原因

blink.cmp 是 LazyVim 默认的补全插件，它包含一个用 Rust 编写的高性能 fuzzy matching 库。插件启动时会尝试下载预编译的 `.so` 文件，但没有 RISC-V 版本，因此报错并 fallback 到纯 Lua 实现。

### 6.3 解决方案：在本机编译 blink.cmp 的 Rust 原生库

进入插件目录：

```bash
cd ~/.local/share/nvim/lazy/blink.cmp
```

编译：

```bash
cargo build --release
```

编译完成后，产物位于 `target/release/libblink_cmp_fuzzy.so`。blink.cmp 的 Lua 加载代码会自动在 `<插件根目录>/target/release/` 下搜索该文件，无需手动移动。

验证产物存在：

```bash
ls target/release/libblink_cmp_fuzzy.so
```

然后在 LazyVim 配置中禁用预编译下载(可省略, 如果还报错就执行该步骤)：

```lua
-- ~/.config/nvim/lua/plugins/blink.lua
return {
  {
    "saghen/blink.cmp",
    opts = {
      fuzzy = {
        prebuilt_binaries = {
          download = false,
        },
      },
    },
  },
}
```

重启 nvim，blink.cmp 将使用本地编译的原生 Rust 库。

> **注意：** 每次 blink.cmp 插件更新后，可能需要重新执行 `cargo build --release`，因为 Rust 源码可能有变更。

---

## 7. 解决 snacks.nvim indent/scope 报错

### 7.1 问题现象

用 nvim 打开 Lua 文件（或其他文件）时报错：

```
Error in BufReadPost Autocommands for "*":
Lua callback: ...snacks.nvim/lua/snacks/scope.lua:136: attempt to index local 'self' (a nil value)
```

### 7.2 原因

尚不确定是 snacks.nvim 本身的 bug 还是 RISC-V LuaJIT 实现的兼容性问题。

推测是后者：`snacks/scope.lua` 在第 136 行使用了 `__eq` metamethod 比较两个 scope 对象。在标准 LuaJIT 中，当比较对象之一为 `nil` 时，`__eq` 不应被触发（Lua 规范规定只有两个操作数类型相同且都有 `__eq` 时才调用元方法）。但 IgnotaYun 的 RISC-V LuaJIT 分支可能在 `__eq` 的 JIT 编译路径上存在细微行为差异，导致 `nil` 值意外进入了元方法调用，从而触发 `attempt to index local 'self' (a nil value)` 错误。

### 7.3 解决方案：禁用 snacks.nvim 的 indent 功能

```lua
-- ~/.config/nvim/lua/plugins/snacks.lua
return {
  {
    "folke/snacks.nvim",
    opts = {
      indent = { enabled = false },
    },
  },
}
```

影响：失去缩进参考线和当前代码块 scope 高亮。这是纯视觉功能，不影响编辑、LSP、补全、treesitter 等核心功能。

---

## 8. 总结


| 组件 | 问题 | 解决方式 |
|------|------|----------|
| LuaJIT | 官方不支持 RISC-V | 使用 IgnotaYun/LuaJIT riscv 分支 |
| Neovim | 默认拉取不支持 RV 的 LuaJIT | `-DUSE_BUNDLED_LUAJIT=OFF` 使用系统 LuaJIT |
| tree-sitter-cli | 无预编译包 | `cargo install tree-sitter-cli` |
| lua_ls | Mason 无预编译包 | 源码编译 + `mason = false` |
| blink.cmp | 无预编译 .so | 插件目录内 `cargo build --release` |
| snacks.nvim | indent/scope 触发 LuaJIT __eq 异常 | 配置 `indent = { enabled = false }` |

### 建议写入 shell 配置的环境变量

```bash
# ~/.zshrc 或 ~/.bashrc
export BINDGEN_EXTRA_CLANG_ARGS="--target=riscv64-linux-gnu -I/usr/include -I/usr/include/riscv64-linux-gnu"
export PATH="$HOME/lua-language-server/bin:$PATH"
```

完成以上步骤后，你将在 RISC-V 上拥有一个功能完整的 LazyVim 开发环境。