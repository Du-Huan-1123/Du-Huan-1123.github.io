---
title: "Hermes Agent 介绍与安装部署指南"
date: 2026-07-09
draft: false
tags: ["AI Agent", "Hermes", "开源", "教程"]
categories: ["技术分享"]
summary: "Hermes Agent 是由 Nous Research 开发的开源 AI Agent 框架，支持跨平台、可执行、可记忆、可扩展。本文介绍其核心特性和 Windows 安装部署流程。"
---

# Hermes Agent 介绍

**Hermes Agent** 是由 Nous Research 开发的一个开源跨平台、可执行、可记忆、可扩展的 AI Agent 框架。

**特点：** 在持续使用中，不断积累用户偏好、任务经验和可复用技能，让每一次交互都成为下一次更优服务的基石。

**最终目标：** 成为一个真正理解你、适应你工作方式、并能随时间推移而变得更聪明、更懂你的个人 AI 助手。

## 核心特征

- Token 消耗少
- 长期记忆更长
- 可以自我进化

## 学习循环（The Learning Loop）

把完成的工作流转化为可复用、可检索、可持续更新的长期能力。

## 记忆系统（Multi-Level Memory System）

按"是否常驻、是否检索、是否程序化、是否个性化"分为四层管理记忆。

---

# 安装部署（Windows）

## 1. 安装 Git

下载地址：https://git-scm.com/?hl=zh-cn

## 2. 安装 WSL2

官方文档：https://learn.microsoft.com/zh-cn/windows/wsl/install

在管理员模式下打开 PowerShell，输入 wsl --install 命令，然后重新启动计算机。

查看已安装的 WSL 发行版：

```bash
wsl --list --verbose
```

设置默认发行版并进入：

```bash
wsl --set-default Ubuntu
wsl
```

## 3. 安装 Hermes Agent

官网：https://hermes-agent.nousresearch.com/

```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
```

## 4. 选择模型和配置 API

```bash
hermes setup model
```

## 5. 接入应用

```bash
hermes gateway setup
```

---

# 启动运行

## 启动方式

1. 打开 PowerShell
2. 进入 WSL
3. 运行 hermes

## 退出方式

- Ctrl + C 中断
- 输入 exit 退出

---

# 个性化定制

在 SOUL.md 文件中可以定义 Agent 的身份，默认是 Hermes。

## SOUL.md 中可以放：

- 语调
- 沟通风格
- 直接程度
- 默认交互风格
- 避免什么
- 如何处理不确定性、分歧或模糊性

## 不应该放：

- 一次性项目指令
- 文件路径
- 仓库规范
- 临时工作细节

## 一个好的 SOUL 文件：

- 在不同上下文中保持稳定
- 足够宽泛，适用于许多对话
- 足够具体，能实质性地塑造声音
- 专注于沟通和身份，而不是特定任务的指令

---

# Agent Skills

技能是按需加载的知识文档，Agent 在需要时将其载入，可以最小化 token 使用量。

所有技能在 ~/.hermes/skills/ 目录下。

查看技能列表：

```bash
hermes skills list
```