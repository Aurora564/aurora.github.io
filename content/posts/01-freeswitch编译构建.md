---
title: "01 Freeswitch编译构建"
slug: "freeswitch-build"
date: 2026-03-13T00:08:35+08:00
draft: false
tags:
  - C 
  - sip
  - freeswitch 
categories:
  - 技术
series:
  - Freeswitch 从零到实战
description: ""
showToc: true
TocOpen: false
---


背景：学习和调试freeswitch
环境信息：fedora 43
编译要求：
- 模块自选安装（sofia协议栈必须，需要可以实现基础的sip通信）
- 生成nvim-clangd lsp需要的文件，我是用nvim阅读freeswitch源码
编译安装目录：项目自用，避免污染整个系统




---

# FreeSWITCH 编译安装指南（学习/调试）

**环境**：Fedora 43 | **目标**：`/home/aurora/project/freeswitch`

---

## 一、目录规划

```bash
mkdir -p ~/project/freeswitch/{src,build,install}
```

```
~/project/freeswitch/
 ├─ src/      # 源码（git clone 到此）
 ├─ build/    # 构建中间产物 + compile_commands.json
 └─ install/  # 最终安装目录（bin/lib/conf）
```

---

## 二、安装依赖

```bash
sudo dnf group list
sudo dnf group install "Development Tools"

sudo dnf install -y \
  git autoconf automake libtool cmake \
  clang clang-tools-extra \
  bear \
  pkgconf pkgconf-pkg-config \
  openssl-devel pcre2-devel zlib-devel \
  sqlite-devel libsndfile-devel curl-devel \
  libuuid-devel speex-devel speexdsp-devel \
  opus-devel libedit-devel \
  libldns-devel unbound-devel
```

`bear` 用于生成 `compile_commands.json`，一起装好省后续麻烦。

---

## 三、克隆源码

```bash
cd ~/project/freeswitch/src
git clone https://github.com/signalwire/freeswitch.git
cd freeswitch
git checkout v1.10   # 固定稳定版本，方便长期阅读
```

---

## 四、配置模块（仅保留学习必需）

```bash
cp modules.conf.in modules.conf
nvim modules.conf
```

注释掉所有模块，**只保留以下内容**：

```
endpoints/mod_sofia
dialplans/mod_dialplan_xml
codecs/mod_opus
codecs/mod_pcm
formats/mod_sndfile
applications/mod_echo
applications/mod_commands
applications/mod_dptools
loggers/mod_console
```

> `mod_sofia` 是 SIP 协议栈核心，必须保留。其余按需增减。

```bash
cd ~/project/freeswitch/build

# 先注释掉所有模块
sed -i 's|^[^#]|#&|' modules.conf

# 只启用必要模块
sed -i \
  -e 's|^#endpoints/mod_sofia$|endpoints/mod_sofia|' \
  -e 's|^#endpoints/mod_loopback$|endpoints/mod_loopback|' \
  -e 's|^#dialplans/mod_dialplan_xml$|dialplans/mod_dialplan_xml|' \
  -e 's|^#codecs/mod_opus$|codecs/mod_opus|' \
  -e 's|^#codecs/mod_g723_1$|codecs/mod_g723_1|' \
  -e 's|^#codecs/mod_g729$|codecs/mod_g729|' \
  -e 's|^#applications/mod_commands$|applications/mod_commands|' \
  -e 's|^#applications/mod_dptools$|applications/mod_dptools|' \
  -e 's|^#applications/mod_echo$|applications/mod_echo|' \
  -e 's|^#formats/mod_sndfile$|formats/mod_sndfile|' \
  -e 's|^#formats/mod_native_file$|formats/mod_native_file|' \
  -e 's|^#formats/mod_tone_stream$|formats/mod_tone_stream|' \
  -e 's|^#loggers/mod_console$|loggers/mod_console|' \
  -e 's|^#loggers/mod_logfile$|loggers/mod_logfile|' \
  -e 's|^#event_handlers/mod_event_socket$|event_handlers/mod_event_socket|' \
  -e 's|^#say/mod_say_en$|say/mod_say_en|' \
  modules.conf
```


---

## 五、Bootstrap

```bash
cd ~/project/freeswitch/src/freeswitch
./bootstrap.sh -j
```

---

## 六、Configure（指向自定义路径）

```bash
cd ~/project/freeswitch/build

../src/freeswitch/configure \
  --prefix=/home/aurora/project/freeswitch/install \
  CFLAGS="-O0 -g3" \
  CXXFLAGS="-O0 -g3"
```

| 参数 | 说明 |
|------|------|
| `--prefix` | 安装到项目目录，不污染系统 |
| `-O0 -g3` | 关闭优化、最大调试符号，方便 gdb/nvim 跳转 |

---

## 七、编译并生成 compile_commands.json

```bash
cd ~/project/freeswitch/build
bear -- make -j$(nproc)
```

编译完成后 `compile_commands.json` 就在 `build/` 目录下。

---

## 八、安装

```bash
make install
```

安装结果：

```
install/
 ├─ bin/freeswitch
 ├─ lib/freeswitch/mod/   # 各模块 .so
 ├─ conf/                 # 默认配置文件
 └─ log/
```

---

## 九、配置 clangd（Neovim）

把 `compile_commands.json` 软链到源码根目录，clangd 会自动识别：

```bash
cd ~/project/freeswitch/src/freeswitch
ln -s ../../build/compile_commands.json .
```

`nvim LSP` 配置（`init.lua`）：

```lua
require('lspconfig').clangd.setup {
  cmd = {
    "clangd",
    "--background-index",
    "--compile-commands-dir=" .. vim.fn.expand("~/project/freeswitch/build"),
  }
}
```

---

## 十、运行与调试

**启动**：
```bash
~/project/freeswitch/install/bin/freeswitch -nonat -nc
```

**CLI 连接**：
```bash
~/project/freeswitch/install/bin/fs_cli
```

**gdb 调试**：
```bash
gdb ~/project/freeswitch/install/bin/freeswitch
(gdb) b switch_core_session_request
(gdb) run -nonat -nc
```

---

## 十一、源码阅读路线（推荐顺序）

```
src/switch.c                        # 程序入口
src/switch_core.c                   # 核心初始化
src/switch_loadable_module.c        # 模块加载机制
src/switch_channel.c / session.c    # 通话对象模型
src/mod/endpoints/mod_sofia/        # SIP 协议栈实现
```

---

## 快速参考

| 路径 | 用途 |
|------|------|
| `~/project/freeswitch/src/freeswitch` | 源码（nvim 在此阅读） |
| `~/project/freeswitch/build` | 构建产物 + `compile_commands.json` |
| `~/project/freeswitch/install/bin` | 可执行文件 |
| `~/project/freeswitch/install/conf` | 配置文件 |

