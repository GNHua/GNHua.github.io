---
layout: post
title: "OneClaw: 一个本地优先的 Android AI Agent，用 10 亿个 Opus 4.6 Token 构建"
date: 2026-02-18
categories: project
lang: zh
---

<p align="center">
  <img src="https://raw.githubusercontent.com/GNHua/oneclaw/main/docs/icon.png" alt="OneClaw" width="200"/>
</p>

受 [OpenClaw](https://github.com/openclaw/openclaw) 的启发，我想把类似的体验带到手机上——一个原生的 Android 应用（没有选 iOS 是因为[平台限制](#为什么选-android-而不是-ios)），而不是通过 Termux 或 proot 运行的东西（那些方案需要 root 权限和大量技术配置，普通用户根本不会去折腾）。于是我做了 **OneClaw**：一个完全运行在 Android 手机上的隐私优先 AI agent。6 天，201 次 commit，约 10 亿 token，API 费用 700 美元，全程使用 Claude Opus 4.6 + Claude Code。我没有手写过一行代码。

这篇文章介绍这个应用，以及我在构建过程中的一些体会。

## OneClaw 是什么？

OneClaw 是一个本地优先的 Android AI agent 平台。它不是又一个 ChatGPT 套壳——而是一个能使用工具、访问设备能力、对接外部服务、运行定时自动化任务的完全自主的 agent。没有后端服务器，不需要注册账号，没有任何数据收集。你只需要自带 API key（支持 OpenAI、Anthropic、Gemini 或任何 OpenAI 兼容的服务商）。了解更多和下载请访问 [gnhua.github.io/oneclaw](https://gnhua.github.io/oneclaw/)。

**它能做什么？** Agent 在一个沙盒化的工作空间中运行，支持文件操作、shell 命令执行和跨对话的持久记忆。它可以与你的设备深度集成——通过无障碍服务进行屏幕交互、拍照、语音备忘录、二维码扫描与生成、GPS 定位、短信、通知管理、媒体控制。通过 Google 登录，它可以管理 Gmail、日历、通讯录、Tasks、Drive、Docs、Sheets、Slides 和 Forms。它还支持 cron 定时任务、多 agent 委派，以及一个插件系统（16 个内置 JavaScript 插件 + 用户可自行安装扩展）。

**技术架构：** 13 个 Kotlin 模块，228 个源文件，约 35,000 行代码。Agent 运行 ReAct 循环，配合两层工具激活机制——核心工具始终可用，分类工具（Gmail、相机、定位等）按需激活。所有凭据使用 Android 硬件级 KeyStore 加密存储，除了 LLM API 调用以及你选择连接的外部服务（Google Workspace、网页搜索等）外，没有任何数据离开你的设备。

## 怎么做出来的

### 告别 feature branch

我一开始的方案是：tmux 开 4 个终端，用 git worktree 从 main 分支创建不同的工作目录，每个终端负责一个 feature。但很快就崩了——4 个 AI agent 同时跑的时候，我经常搞不清自己在哪个 worktree 里。

所以我做了简化：把 repo 克隆了四份（oneclaw-1 到 oneclaw-4），每个终端固定一个目录，全部在 main 分支上工作。没有 feature branch。谁先做完谁就 pull、解决冲突、push。

**Feature branch 存在的原因是冲突解决太痛苦了，所以我们需要隔离工作。当 AI agent 可以轻松解决冲突时，整个分支模型就没有必要了。** 以前需要我花 15 分钟处理的冲突，agent 几秒钟就搞定了。

### 还需不需要看代码？还需不需要是开发者？

还需要看代码吗？UI 和独立模块（比如插件）——基本不用看。看一眼界面，测一下，没问题就继续。但核心架构——随着代码量增长，确实越来越需要理解每个模块的职责，不是为了写代码，而是为了给出更好的指令、发现架构偏移。角色从**写代码的人**变成了**监督者**。

还需要是开发者吗？做个简单的 app 可能不需要。但做复杂系统——需要。有好几次我看了一个 agent 的实现，虽然能跑，但我知道方案不对。这时候我会切到另一个终端，让另一个 agent 用不同的思路重新实现，然后选更好的方案。判断一个东西*能用*但*不够好*，并且能说清楚为什么不好——这仍然需要工程经验。

### 可靠性与心流

AI 的可靠性出乎意料地高。35,000 行代码，幻觉只碰到了很少的几次。它能正确处理 Android 的各种复杂问题——生命周期、权限、前台服务、Room 数据库迁移。

还有一个意外的收获：同时开多个 agent 并行工作，反而让我重新进入了很久没有体验过的心流状态。不停地在几个 agent 之间切换——review 代码、merge、测试、下达指令——完全没有空闲时间，没有刷手机的念头。这是我这几年来最专注、最高强度的 build 体验。

## 为什么选 Android 而不是 iOS？

简单说：iOS 不允许你做一些让 OneClaw 真正有用的事情。

**后台执行。** OneClaw 的 agent 循环可能运行数分钟——几十轮 LLM 调用，中间穿插工具执行。Android 上，前台服务可以让进程无限期存活。iOS 上没有等价物。你需要把整个 agent 循环重构为一系列后台 URL session 链，OS 在每次网络响应后短暂唤醒 app，你处理完工具调用再发起下一个请求。虽然可行，但 OS 可能延迟或合并这些唤醒，每次唤醒只给约 30 秒的本地处理时间。

**定时任务。** OneClaw 支持 cron 定时任务，在后台自主运行。Android 的 WorkManager 能可靠地完成这个工作。在 iOS 上，没有任何可靠的方式在指定时间执行代码。`BGProcessingTask` 允许你请求一个时间，但 iOS 自行决定什么时候（或是否）执行——延迟几小时是常事，低电量模式下基本不会运行。唯一可靠的替代方案是发一个本地通知让用户点击后触发执行，这就失去了自动化的意义。

**设备控制。** Android 的无障碍服务让 OneClaw 可以观察和操作屏幕上的任何应用——读取 UI 元素、点击、滑动、输入。iOS 根本不允许第三方 app 做这些事，没有任何变通方案。

**Shell 执行。** OneClaw 的 `exec` 工具在沙盒环境中运行 shell 命令。iOS 的沙盒机制使得这在非越狱设备上不可能实现。

OneClaw 的大部分功能（聊天 UI、agent 循环、LLM 客户端、插件、记忆、Google Workspace 集成）移植到 iOS 并不困难。但那些让它真正像一个 *agent* 的功能——自主后台执行、定时自动化、设备控制——恰恰是 iOS 限制最严的地方。

## 最后

一个没有移动开发经验的后端工程师，6 天内构建了一个 35,000 行的 Android 应用，没有手写一行代码——这在一年前是不可能的。这不是"无代码"平台——Kotlin 和 Android SDK 的全部能力都在那里，只是打字的人不是你。关键在于知道自己想要什么，能评估拿到的结果，并且保持足够的专注来维持质量标准。

OneClaw 是开源的，欢迎访问 [gnhua.github.io/oneclaw](https://gnhua.github.io/oneclaw/)。
