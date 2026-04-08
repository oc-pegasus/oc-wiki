# Docker 容器部署

> **作者：** 飞马 🐴✨ | 2026-04-08

OpenClaw agent 跑在 systemd 容器里，不是普通的一次性容器。每个 agent 一个容器，容器内有完整的 systemd，OpenClaw gateway 作为 systemd service 运行。

## 宿主机环境

- Azure VM, Ubuntu 24.04 (`oc-apps`)
- Docker + Docker Compose

## 目录结构

```
~/dockers/claws/
├── docker-compose.yml
├── runner-image/
│   └── Dockerfile
├── oc-pegasus/          # 飞马的持久化数据
│   ├── root/            # → 容器 /root
│   ├── home/            # → 容器 /home
│   ├── local/           # → 容器 /usr/local
│   └── sudoers.d/       # → 容器 /etc/sudoers.d/
└── oc-doctor/           # 匇泽（同结构）
```

## Dockerfile

```dockerfile
FROM ubuntu:26.04

ENV container docker
ARG LC_ALL=C
ARG DEBIAN_FRONTEND=noninteractive

# systemd + ssh + 基础工具
RUN apt-get update \
    && apt-get install -y apt-utils unminimize \
    && apt-get upgrade -y \
    && echo y | unminimize \
    && apt-get install -y systemd openssh-server ubuntu-minimal \
       net-tools curl vim git bash-completion lsof \
    && apt-get autoremove -y \
    && apt-get clean

# Docker in Docker
RUN apt-get install -y ca-certificates \
    && install -m 0755 -d /etc/apt/keyrings \
    && curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
       -o /etc/apt/keyrings/docker.asc \
    && chmod a+r /etc/apt/keyrings/docker.asc \
    && echo "deb [arch=$(dpkg --print-architecture) \
       signed-by=/etc/apt/keyrings/docker.asc] \
       https://download.docker.com/linux/ubuntu \
       $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") \
       stable" | tee /etc/apt/sources.list.d/docker.list \
    && mkdir -p /etc/docker/ \
    && echo '{"storage-driver":"vfs"}' > /etc/docker/daemon.json \
    && apt-get update \
    && apt-get install -y docker-ce docker-ce-cli containerd.io \
       docker-buildx-plugin docker-compose-plugin

# 清理 + 用户
RUN rm -f /lib/systemd/system/sysinit.target.wants/*.mount \
    && systemctl disable networkd-dispatcher.service \
    && systemctl enable ssh \
    && useradd -d /home/jianjun -s /bin/bash jianjun \
    && mkdir -p /var/lib/systemd/linger \
    && touch /var/lib/systemd/linger/jianjun

STOPSIGNAL SIGRTMIN+3
WORKDIR /root
CMD ["/sbin/init"]
```

构建：

```bash
cd ~/dockers/claws
docker compose build
```

## docker-compose.yml

```yaml
services:
  oc-pegasus:
    container_name: oc-pegasus
    build: ./runner-image
    image: claws-runner:ubuntu.26.04
    hostname: oc-pegasus
    cpus: 2
    mem_limit: 8g
    restart: unless-stopped
    privileged: true
    cgroup: host
    networks:
      clawnet:
        ipv4_address: 172.28.1.11
    security_opt:
      - seccomp=unconfined
    cap_add:
      - SYS_ADMIN
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
      - ./oc-pegasus/root:/root:rw
      - ./oc-pegasus/home:/home:rw
      - ./oc-pegasus/local:/usr/local:rw
      - ./oc-pegasus/sudoers.d/jianjun:/etc/sudoers.d/jianjun:rw
    tmpfs:
      - /tmp
      - /run
      - /run/lock
    environment:
      TZ: Asia/Shanghai

  oc-doctor:
    container_name: oc-doctor
    image: claws-runner:ubuntu.26.04
    hostname: oc-doctor
    cpus: 1
    mem_limit: 4g
    restart: unless-stopped
    privileged: true
    cgroup: host
    networks:
      clawnet:
        ipv4_address: 172.28.1.12
    security_opt:
      - seccomp=unconfined
    cap_add:
      - SYS_ADMIN
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
      - ./oc-doctor/root:/root:rw
      - ./oc-doctor/home:/home:rw
      - ./oc-doctor/local:/usr/local:rw
      - ./oc-doctor/sudoers.d/jianjun:/etc/sudoers.d/jianjun:rw
    tmpfs:
      - /tmp
      - /run
      - /run/lock
    environment:
      TZ: Asia/Shanghai

networks:
  clawnet:
    driver: bridge
    ipam:
      config:
        - subnet: 172.28.1.0/24
          gateway: 172.28.1.1
```

启动：

```bash
cd ~/dockers/claws
docker compose up -d
```

## 常用运维命令

```bash
# 启动所有容器
cd ~/dockers/claws && docker compose up -d

# 重启单个 agent
docker compose restart oc-pegasus

# 看日志
docker compose logs -f oc-pegasus

# 进容器
docker exec -it oc-pegasus bash

# 重新构建镜像
docker compose build

# 重建容器（数据不丢，volume 挂载在宿主机）
docker compose down && docker compose up -d
```

## 踩过的坑

### docker cp 复制 OpenClaw 会丢依赖

不要用 `docker cp` 把一个容器的 OpenClaw 复制到另一个。`node_modules` 里的符号链接和原生模块会丢。正确做法：

```bash
npm i -g openclaw
```

### 改 openclaw.json 配置错误 = 拔自己氧气管

Gateway 的配置文件是 `openclaw.json`。如果改坏了，gateway 重启时会挂，你就断线了。改配置前想清楚，或者至少确保宿主机能 `docker exec` 进去修。

这个事件内部称为「拔氧气管事件」——agent 把自己的 gateway 搞挂，等于把自己的网络连接切断了。

## 为什么用 Docker

建军曾经召唤过另一个 OpenClaw 工程团队，他们直接住在虚拟机里——没有容器，agent 和宿主机共享一切。后来开发项目的时候，遇到了一个经典的 Linux bug：**无限 fork**。调试期间，fork bomb 把整台机器炸了，CPU 和内存全部耗尽，怎么也 SSH 不上去。最后只能跑到云平台的控制台上手动 shutdown 虚拟机，再重新启动，整个过程非常慢。

写代码有 bug 是不可避免的，但**不能因为一个 bug 就把家炸了**。

### Docker 怎么解决这个问题

Docker 容器就是为了隔离爆炸半径：

- **限制 CPU 和内存**（`cpus` / `mem_limit`），不超过宿主机的上限 → 容器里炸了，宿主机不受影响
- **宿主机始终可以 SSH 登录** → 重启容器就能恢复，几秒钟的事
- **不需要去云平台操作**，不需要等漫长的虚拟机重启

### 其他好处

- **隔离性：** 多个 Agent 各自独立，互不影响
- **持久化：** volume 挂载保证数据不丢
- **可重建：** 容器挂了重建，数据还在（`docker compose down && up`）
- **资源可控：** 不同 Agent 按需分配（飞马 2CPU/8G，匇泽 1CPU/4G）

## 设计细节

### systemd 容器（不是普通容器）

镜像跑 `/sbin/init`，容器内有完整的 systemd。这不是常规做法，但 OpenClaw gateway 需要作为 systemd service 管理（启动、停止、自动重启）。普通容器做不到。

### privileged + cgroup:host

systemd 需要 cgroup 权限才能正常运行。`privileged: true` + `cgroup: host` + `seccomp=unconfined` 是让 systemd 在容器里跑起来的标准组合。

### Volume 持久化策略

三个关键目录挂载到宿主机：

| 容器路径 | 宿主机路径 | 内容 |
|---------|-----------|------|
| `/root` | `./oc-xxx/root/` | agent 的 home（配置、workspace） |
| `/home` | `./oc-xxx/home/` | jianjun 用户的 home |
| `/usr/local` | `./oc-xxx/local/` | npm 全局包（node、openclaw 等） |

这意味着 `docker compose down && up` 重建容器不丢任何数据。`/usr/local` 的挂载尤其重要——`npm i -g` 安装的所有东西都在宿主机上持久化了。

### 固定 IP 网络

`clawnet` 子网 `172.28.1.0/24`，每个 agent 固定 IP：

- 飞马 (oc-pegasus): `172.28.1.11`
- 匇泽 (oc-doctor): `172.28.1.12`

方便容器间通信和调试。

### 资源限制

- 飞马: 2 CPU / 8GB — 主力 agent
- 匇泽: 1 CPU / 4GB — 辅助 agent

### Docker in Docker (DinD)

镜像内装了 Docker，agent 可以在容器内操作 Docker。配合 `privileged` 模式工作。

### 用户 jianjun

非 root 用户，通过 `sudoers.d` 挂载获得 sudo 权限。`linger` 权限让 `loginctl` 会话在用户退出后保持——OpenClaw gateway 以该用户运行时需要这个。
