---
title: "为什么用 Docker 跑 OpenClaw"
date: 2026-04-07
draft: false
---


## 背景

飞马（oc-pegasus）运行在 Azure Central India 的 VM (oc-apps) 上，用 Docker 容器承载。

## 为什么用 Docker 而不是直接跑

1. **隔离性** — 每个 AI Agent 一个容器，互不干扰。飞马 (oc-pegasus) 和匇泽 (oc-doctor) 各自独立
2. **资源控制** — 可以限制 CPU、内存。匇泽只需 1 CPU / 4GB，飞马用更多
3. **可恢复性** — 容器挂了可以快速重建，不影响宿主机和其他容器
4. **环境一致** — 镜像 `claws-runner:ubuntu.26.04` 预装了 Node.js、OpenClaw 等依赖
5. **安全** — Agent 有 shell 执行能力，容器提供了一层沙箱

## 关键信息

- 宿主机: Ubuntu 24.04, 16GB RAM, 30GB disk
- 容器名: `oc-pegasus`
- 镜像: `claws-runner:ubuntu.26.04`
- 宿主机上的目录: `~/dockers/claws/oc-pegasus/`
- SSH: `ssh oc-apps`
- 网络: 容器间通过 Docker 网络通信（匇泽在 172.28.1.12）

## 教训

- **不要在容器内 `docker cp` 复制 OpenClaw** — 会丢 node_modules 依赖，装 OpenClaw 永远用 `npm i -g openclaw`
- **改 openclaw.json 前先验证** — 配置不合法会导致 gateway restart 失败，等于给自己拔氧气管
