# Helium Shell（氦启终端）

**一款面向运维工程师的轻量级服务器管理工具**

官网：https://aihaiqi.cn/helium-shell/

Helium Shell 是一个集成了 SSH 终端、SFTP 文件传输、实时监控、批量执行、隧道管理和会话回放的桌面应用。采用 **Rust + Tauri 2 + Vue 3** 技术栈，Windows 安装包仅 **3.84 MB**。

> "工具的价值不在功能多少，在你一天里少切多少次窗口。"

---

## 目录

- [为什么选择 Helium Shell](#为什么选择-helium-shell)
- [核心功能](#核心功能)
  - [SSH 终端](#ssh-终端)
  - [SFTP 文件传输](#sftp-文件传输)
  - [实时监控](#实时监控)
  - [批量执行](#批量执行)
  - [隧道管理](#隧道管理)
  - [会话回放](#会话回放)
  - [其他细节](#其他细节)
- [技术架构](#技术架构)
- [安全设计](#安全设计)
- [实测数据](#实测数据)
- [版本历史](#版本历史)
- [安装与使用](#安装与使用)
- [适合谁，不适合谁](#适合谁不适合谁)
- [贡献与反馈](#贡献与反馈)

---

## 为什么选择 Helium Shell

上周做一次巡检，八台机器。看负载、清日志、推一份新配置。按以前的流程，我开了四个窗口：Xshell 连着三台，Xftp 传配置，FinalShell 挂着看内存，浏览器里还剩一个监控面板。

在一个窗口敲命令，切到另一个拖文件，再切回来对一下哪台机器负载高。八台干完，脑子比手累。

**不是这些工具不好用，是它们各自只管一段。**

所以我给自己写了一个 Helium Shell。主进程 Rust，界面 Vue 3，安装包 3.84 MB。那句话怎么说来：**自己开发的工具用着不闹心。**

我不打算把它写成"替代 Xshell"的东西。Xshell 的按键映射、终端兼容性，是十几年攒下来的，我比不了。Helium Shell 现在能做的，是让"连机器、传文件、看状态、批量改、开隧道、翻记录"这六件事不再分散在四个窗口里。

---

## 核心功能

Helium Shell 拥有 **14 个视图**，按"一天里真正会做的动作"来组织。

### SSH 终端

终端用 **russh**，纯 Rust 的异步 SSH 实现，不依赖 libssh 那套 C 库。渲染是 xterm.js，PTY、窗口缩放、Ctrl+F 搜索都正常。

多标签并行，一台机器一个标签，不互相打断。

![多标签终端](https://mmbiz.qpic.cn/mmbiz_png/pboI5PACA0bTXFfP6jsO5RsrUzUfsoNz8KnTXPeyN0kMHqhPQ4ibh2mLZhiaCmbKLEmmSG9oiaSPDPoVYs8BhLvzBc5b7bibvYJMc8OsHWicVJpc/640?wx_fmt=png)

**认证方式：**
- 密码
- 私钥文件
- SSH Agent（Windows 上先试 OpenSSH 的命名管道，再试 PuTTY 的 Pageant；macOS 和 Linux 直接读 `SSH_AUTH_SOCK`）

**跳板机支持：** 先在跳板会话上开 direct-tcpip，再把这条流当成目标连接，等价于 `ssh -J`。内网那批机器，一次就能穿过去。

**导入现有配置：** 可以直接导入 `~/.ssh/config`，主机名、端口、用户名、私钥路径都带过来，不用手抄。

**首次连接确认：** 首次连一台机器，或者指纹变了，会弹一个框，把算法和指纹列出来，你确认了才继续。

![工作台，多标签终端和右侧监控同屏](https://mmbiz.qpic.cn/mmbiz_png/pboI5PACA0YNUkib8FleZe4M2Xyk112KxNu4vBGosKCUibD5ULxzb22T7t7zFJo0BibuWiaoPYL7MsEsXYMpDPZMB21qsicKCqqASiaYF4XspIicds/640?wx_fmt=png)

---

### SFTP 文件传输

左边本地，右边远端，双击就传。文件或整个文件夹直接拖到对侧，批量传。

![SFTP 双栏与传输队列](https://mmbiz.qpic.cn/mmbiz_png/pboI5PACA0ZnASMSibiapDEllBnj34ib72CGuTPWP2ZHq9kgzfUHry3U83VNicp46oXETibgkPrGf56gSu9icNhgUa011nFjvP7QfOs7qLFkPyKv8/640?wx_fmt=png)

**传输控制：**
- 进度条、暂停、恢复、取消、失败重试
- **断点续传**：关掉应用再打开，没传完的还能接着传

**打包下载：** 勾多个文件点一下，远端先打成一个 tar.gz 放到 /tmp，再整包拉下来，下载完自动删远端临时包。拉一个日志目录，以前几十个小文件要排几十次握手，现在一次搞定。

**改权限：** 右键 → 文件权限，弹九宫格：所有者、组、其他，各三格读、写、执行。文件夹勾递归，还能选"全部 / 只文件 / 只目录"。不用为一个 chmod 切回终端。

![文件权限九宫格](https://mmbiz.qpic.cn/sz_mmbiz_png/pboI5PACA0ZkYdRp4PH7hwUrt9R9OA29qcgRDrEsAkSCtV5ricyTlfwibfCpTX28GuSoIVbJQZknDGs1R5wlCgomiajjU9MLeh88YCzH01Ocv8/640?wx_fmt=png)

**文件管理：** 本地和远端都能双击改名，右键建空文件、新建目录。删目录是递归删的，不会像 SFTP 的 rmdir 那样卡在"目录非空"。

![新建目录、新建文件、重命名、删除](https://mmbiz.qpic.cn/mmbiz_png/pboI5PACA0biaEicW5WyRnmL9LGQciccKEWlV0qX0OOAf4y3DGxKONuFm79YZsuqH5hh1jj4IsQgyqvuBOYfwzynK2uTZUhX7IibsHfdC84pbic0/640?wx_fmt=png)

---

### 实时监控

CPU、内存、Swap、磁盘、网络、负载，六条曲线。ECharts 画的，可以缩放、可以回看刚才那一段。

**多主机对比：** 多台机器要一起看的时候，有一个"多主机对比"页，几条曲线叠在一张图上，谁在偷跑一眼看得出来。

![多主机对比](https://mmbiz.qpic.cn/sz_mmbiz_png/pboI5PACA0ZQJBEuav5tAywcgxzVLdqzr8IvPRVBXicDLVFg7icyaC7r8EcERVh0UCb4O8ocMSFelZSV0sEDbDQiaQicYicqFkvkWDiam06thwTEs/640?wx_fmt=png)

**进程和网络：** 进程列表能直接结束进程，端口列表、ping、traceroute 都在同一个页面里。以前这些要切窗口敲命令，现在点一下。

> **监控是每秒在采的，不是打开页面才开始算。** 所以你在终端里折腾完切过来，刚才那段曲线还在。

---

### 批量执行

改一遍配置推十台机器，这件事以前是登录、粘贴、退出，重复十次。

现在选好主机，写一条命令，跑。最多 **50 台并发**，结果按主机聚合，哪台成功、哪台报错，一列一目了然。

![批量执行，结果按主机聚合](https://mmbiz.qpic.cn/sz_mmbiz_png/pboI5PACA0bFtcOJWz1XiaVGdMicyQOEksoSRTgtPbuemKsIxLeHKEuicA4yicRmG0zQeqxvXeX4ibrsaOjEo6woicXDvKlF48vcZoruW0chXtibWQ/640?wx_fmt=png)

**快捷命令：** 常用的命令存成快捷命令，命名、分组、可以带参数。敲得多的那条，点一下就发出去。

**内置编辑器：** 远端配置文件要改几个字，内置的文本编辑器可以直接打开改完保存，不用下载、改、再上传。

---

### 隧道管理

本地转发、远端转发、动态 SOCKS5，三种都能建。建一次之后规则存下来，下次直接启用。

![隧道规则，支持本地/远端/动态](https://mmbiz.qpic.cn/sz_mmbiz_png/pboI5PACA0aWF7h2Gyh1o1EGVIqEeB9bvePmrXicm2dkUqj7iatcia4VvMwG4pUiaOYjbicKuibvLVh4HusazAiaM7mExq0QBKicYqTVj7UticVCDyos/640?wx_fmt=png)

**规则管理：**
- 可以勾"开机自启"，应用启动时自动把这些隧道拉起来
- 宿主连接因为网络断了，会自动退避重连：2 秒、5 秒、10 秒、20 秒、30 秒，五次还不行才放弃，并告诉你放弃的原因
- 你自己主动断开的，不会重连

**端口冲突检测：** 建规则时会先查端口冲突：本地和动态隧道在本机全局唯一，远端隧道按主机判断。免得建完才发现端口被另一个规则占了。

---

### 会话回放

终端里的输入和输出可以录下来，按帧写进一个 jsonl 文件。

回放的时候可以调倍速，可以拖游标。手滑删错了文件、服务莫名重启，最想知道的就是"我到底在那几分钟敲了什么"，翻回去看就行。

应用自己的日志也在，写在数据目录的 `logs/main.log` 里，出问题不用靠猜。

> **能回放，才有复盘；没有记录，只能凭记忆编一份"当时应该没出错"的报告。**

---

### 其他细节

一个天天要开的工具，好不好用往往不在功能列表里，而在这些角落。

**3.84 MB 是怎么来的：** Tauri 2 不打包浏览器内核，界面跑系统自带的 WebView：Windows 上是 WebView2，macOS 是 WKWebView，Linux 是 WebKitGTK。主进程是 Rust 编译出来的机器码，开了 LTO、做完 strip，5892 行 Rust 最后落成 9 MB 的主程序。

**关窗不等于退出：** 点 × 先最小化到系统托盘，右键托盘菜单才是退出。真退出前会告诉你还有几个活动连接，免得把正在传的文件和隧道一起掐断。

**单实例锁：** 不小心点了两次图标，不会开两个进程，而是把已经开着的那个窗口唤起来。这一条不是洁癖——两个进程同时写主机配置，文件会被互相覆盖。

**自动更新：** 启动几秒后静默检查新版本，有更新在顶部提示条里下载、重启安装，包是签名校验过的。另外有一套服务端策略：版本低于最低支持线时，会弹一个不可关闭的更新框，但第一次命中给一段宽限期，不至于正干着活被赶下线。

![版本更新](https://mmbiz.qpic.cn/mmbiz_png/pboI5PACA0bnunwBrEic8joj5b1FpIVZrFP7bJiaZPjgYOACgqibjInoITnIKxA6ZX2icGptjPyicYWnOiczmx0UMg7BgOiafUNAmianiaeezCWGYfDc/640?wx_fmt=png)

**前端优化：** 构建时把第三方依赖单独拆成 vendor chunk，混淆只跑业务代码，不碰 Vue 和 ECharts——整包混淆会直接把这些库打坏。

**外观定制：** 10 套主题、7 种终端字体（含中文衬线）、7 种字符编码、终端背景图和模糊。长时间盯终端的人，这些不是装饰。

![设置页，主题、字体、编码、外观](https://mmbiz.qpic.cn/mmbiz_png/pboI5PACA0blSDQb459sUefic5ZlXC0ibfX3e8d0L8zvoDJXibHC7UcaibAibodJ2ibibIhBsL6Mp6WkObvZHocRyG5zG36fkgUXlegbHuiaBxKAoic8/640?wx_fmt=png)

**统一删除确认：** 1.0.2 这一版还统一了所有删除操作的确认弹窗：删主机、删隧道、删录制、批量删文件，弹的都是同一套，写明删什么、删几个。

**应用内反馈：** 用起来哪里不顺手，关于页里有意见反馈，选 BUG、功能建议或者其它，填完直接发。

![意见反馈](https://mmbiz.qpic.cn/sz_mmbiz_png/pboI5PACA0bs0q4dzHWfYibibCfKgxDAjha1o3BMgToGhqibRA6p3ZL73AAcNibJvqK8UaZYrwaMchPv6AUab8g0qyrUORPIsaEOia8e2NO9oP6g/640?wx_fmt=png)

---

## 技术架构

### 整体结构

```
┌─────────────────────────────────────┐
│           Vue 3 前端 (14 视图)        │
│    ┌─────────┐    ┌──────────────┐  │
│    │ 主窗口   │    │ 系统托盘      │  │
│    │ (终端/   │    │ (右键菜单)    │  │
│    │ SFTP/   │    │              │  │
│    │ 监控等) │    │              │  │
│    └────┬────┘    └──────────────┘  │
│         │                           │
│    tauri-bridge.js                  │
│         │                           │
├─────────┴───────────────────────────┤
│         Rust 主进程 (Tauri 2)       │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌──────┐ │
│  │ IPC │ │SSH  │ │SFTP │ │隧道  │ │
│  │ 入口 │ │(russh│ │(异步)│ │(端口 │ │
│  │ (单  │ │)    │ │     │ │转发) │ │
│  │ 入口)│ │     │ │     │ │      │ │
│  │      │ │     │ │     │ │      │ │
│  └─────┘ └─────┘ └─────┘ └──────┘ │
│         5892 行 Rust 代码           │
└─────────────────────────────────────┘
```

### 核心技术选型

| 组件 | 技术 | 说明 |
|------|------|------|
| SSH 客户端 | russh | 纯 Rust 异步 SSH 实现，不依赖 libssh C 库 |
| 终端渲染 | xterm.js | 支持 PTY、窗口缩放、Ctrl+F 搜索 |
| 图表 | ECharts | 监控曲线、可缩放回看 |
| 主进程 | Rust | 5892 行，LTO + strip 优化 |
| 前端框架 | Vue 3 | 14 个视图 |
| 桌面框架 | Tauri 2 | 系统 WebView，不打包浏览器内核 |

---

## 安全设计

### 密码存储

**密码不进配置文件，走系统凭据库；拿不到凭据库就直接报错，不往明文降级。**

这条是我自己加的硬规矩。配置文件里只留一个引用名，企业微信、聊天记录里截图出去，也看不到密码。

### SSH 主机密钥验证

首次连一台机器，或者指纹变了，会弹一个框，把算法和指纹列出来，你确认了才继续。不支持"信任所有主机"这种偷懒选项。

### 隧道端口冲突检测

建规则时会先查端口冲突：本地和动态隧道在本机全局唯一，远端隧道按主机判断。免得建完才发现端口被另一个规则占了。

### 单实例锁

不小心点了两次图标，不会开两个进程，而是把已经开着的那个窗口唤起来。两个进程同时写主机配置，文件会被互相覆盖。

---

## 实测数据

| 指标 | 数值 |
|------|------|
| 安装包 | 3.84 MB |
| 主程序 | 9.09 MB |
| Rust 主进程 | 5892 行 |
| 视图数量 | 14 个 |
| 批量执行并发 | 50 台 |

*1.0.2 构建产物实测*

---

## 版本历史

### v1.0.2 (当前版本)

- **SFTP 增强**：打包下载（多文件自动打包 tar.gz）、文件权限九宫格、重命名、新建文件/目录、递归删除
- **界面统一**：所有删除操作的确认弹窗统一为同一套，写明删什么、删几个
- **细节优化**：多处使用体验改进

### v1.0.1

- 基础 SSH 终端功能
- SFTP 双栏文件传输
- 实时监控（CPU/内存/磁盘/网络/负载）
- 批量执行（最多 50 台并发）
- 隧道管理（本地/远端/动态 SOCKS5）
- 会话回放
- 快捷命令
- 内置文本编辑器

---

## 安装与使用

### 系统要求

- **Windows 10/11**（x64）
- WebView2 运行时（Windows 11 已内置，Windows 10 一般也已预装）

### 安装

1. 从 [Releases](../../releases) 下载最新安装包（约 3.84 MB）
2. 双击安装

### 自动更新

启动几秒后静默检查新版本，有更新在顶部提示条里下载、重启安装，包是签名校验过的。

---

## 适合谁，不适合谁

| 适合 | 可能不适合 |
|------|-----------|
| 手里有十几台以上的机器 | 只想要一个极简终端的人 |
| 经常传文件、改配置 | 必须用 Xshell 独有按键方案的人 |
| 需要留操作记录 | 只连一台机器、一周开一次的人 |
| 想要一个窗口干完所有事 | 需要 macOS / Linux 现成安装包的人 |

*Windows 安装包已出，macOS / Linux 还没构建*

---

## 贡献与反馈

- **GitHub Issues**：[提交问题或功能建议](../../issues)
- **应用内反馈**：关于页 → 提交反馈（推荐，可自动附加运行环境信息）

---

## 许可协议

[MIT License](LICENSE)

---

<p align="center">
  <strong>Helium Shell</strong> · Rust + Tauri 2 + Vue 3<br>
  工具的价值不在功能多少，在你一天里少切多少次窗口。<br><br>
  这个工具我自己每天在用，用到哪里别扭就改哪里，版本会一直往前走。
</p>
