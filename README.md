<div align="center"><img src="docs/public/logo.svg" width="88" height="88" alt="Cloud CLI Proxy" />
Cloud CLI Proxy
企业级安全远程开发环境。为团队提供隔离、可审计、策略驱动的 Claude Code 容器化运行平台。

https://github.com/ZaneL1u/cloud-cli-proxy/actions/workflows/ci.yml/badge.svg
https://github.com/ZaneL1u/cloud-cli-proxy/actions/workflows/build-images.yml/badge.svg
https://img.shields.io/github/v/release/ZaneL1u/cloud-cli-proxy
https://img.shields.io/badge/License-MIT-blue.svg

English | 文档

</div>
解决什么问题？
随着 AI 编码工具在企业中的普及，安全与合规团队面临新的挑战：开发环境不一致、AI 代理权限过高、敏感代码库缺乏隔离、网络出口难以管控、操作行为无法审计。直接在生产环境或个人设备上运行 Claude Code，往往无法满足企业的安全基线与合规要求。

Cloud CLI Proxy 提供了一套自托管、容器化的远程开发基础设施，将 Claude Code 封装在受控的隔离环境中。每个开发者获得独立、标准化的容器工作区，所有网络出口、系统配置和操作行为均由平台统一策略管理，确保 AI 编码活动在安全、合规、可审计的框架内进行。

部署在你自己的服务器上，开发者通过 SSH 接入即可开始编码。本地项目目录自动挂载到容器，路径保持一致，开发体验与本地运行无异。

它怎么做到的？
环境隔离与安全加固
平台从多个维度对开发环境进行标准化和隔离，确保容器工作区与宿主系统解耦，并满足企业安全基线：

系统环境标准化 — 统一 CPU 型号、MAC 地址、machine-id 等系统标识，确保所有开发者容器环境一致，消除因环境差异导致的安全盲区。ioreg、system_profiler、sysctl 等命令的输出经过策略化处理，避免泄露底层基础设施信息。

标准化主机命名 — 自动生成符合企业命名规范的主机名，便于资产管理与审计追踪。

最小化攻击面 — 移除容器内不必要的系统信息暴露（如 /.dockerenv、cgroup 中的容器运行时标识），遵循安全加固最佳实践，减少潜在攻击面。

区域与语言合规 — 默认配置太平洋时区与 en_US.UTF-8，创建容器时可按需调整，满足不同地区的合规与本地化要求。

出站流量合规 — 出口流量采用 uTLS，指纹与标准 Chrome 浏览器一致，确保网络通信符合企业出站策略，避免异常流量特征触发安全告警。

数据防泄露（DLP）与网络出站控制 — 在 DNS 级别实施网络策略，屏蔽 statsig.anthropic.com、sentry.io、cdn.growthbook.io 等遥测与追踪域名，防止敏感数据通过 AI 代理意外外泄，满足企业 DLP 要求。

网络策略执行与出口隔离
每个容器绑定独立的出口 IP，所有流量——HTTP、DNS、WebRTC——全部通过 sing-box TUN 隧道从指定出口发出。nftables 默认拒绝直连，确保不存在策略绕过路径。

支持 SOCKS5、VMess、VLESS、Shadowsocks、Trojan、HTTP 六种协议，方便接入企业现有的网络出口或合规代理基础设施。

企业级网络出口建议：请使用符合企业合规要求的网络出口，例如企业自有的住宅宽带、商业 VPN 或云网络出口。避免使用已被标记的数据中心 IP 段，以确保网络策略的稳定性和合规性。

代码映射——与本地开发体验一致
这是与普通 VPS 最大的区别。容器通过 sshfs 将本地项目目录挂载进去，路径完全一致。

什么意思？你在本地 ~/my-project 里写代码，容器里也是 ~/my-project。Claude Code 读到的路径与本地没有丝毫差别，就像在本机运行一样。挂载支持三种模式：全量同步、智能热同步、纯挂载。断线自动重连，30 秒内恢复，输入不丢失。tmux 多端会话，断开也不丢失工作区。

开箱即用
管理员在后台创建容器后，用户收到 curl 接入命令，终端运行、输入密码、等待容器启动，自动 SSH 进入。容器内 Claude Code 已预装，直接运行 claude 即可使用。

企业级管理与合规能力
平台提供完整的管理与审计能力，帮助企业满足安全合规要求：

用户生命周期管理 — 用户创建、暂停、过期自动停机、密码轮换，支持基于角色的访问控制（RBAC）。

主机管理 — 主机创建、启动、停止、重建（保留或清空 /workspace）、删除，所有操作可追溯。

网络出口管理 — 出口 IP 增删改查、连通性测试，确保网络策略始终有效。

Bypass 防火墙 — 按域名、CIDR、端口白名单直连，支持快照版本管理，预览→应用→回滚，满足精细化网络访问控制需求。

审计日志 — 所有操作写入审计事件，可追溯，满足 SOX、GDPR 等合规审计要求。

实时监控 — SSE 实时推送任务进度和主机状态。

远程桌面 — 容器内置 KasmVNC + Chromium，管理后台一键打开远程桌面，便于运维与支持。

快速开始
bash
git clone https://github.com/ZaneL1u/cloud-cli-proxy.git
cd cloud-cli-proxy

bash deploy/scripts/setup-env.sh

docker compose pull
docker compose up -d

curl http://127.0.0.1:8080/healthz
# {"status":"ok"}
启动后：

管理后台：http://YOUR_HOST:8080

API：http://YOUR_HOST:8080

SSH 代理：YOUR_HOST:2222

首次使用：登录管理后台 → 添加出口 IP → 创建用户 → 创建主机 → 把接入命令发给用户。

安装
环境要求
Docker Engine 28.x+

Docker Compose v2

Docker Compose（推荐）
bash
bash deploy/scripts/setup-env.sh  # 交互式生成密码和密钥
docker compose pull               # 拉取预构建镜像
docker compose up -d              # 启动
宿主机直接部署
bash
sudo bash deploy/scripts/deploy.sh
创建 cloudproxy 系统用户，构建二进制和镜像，安装 systemd 单元并启动。

环境变量
变量	说明	默认值
DATABASE_URL	SQLite 数据库路径	file:/data/cloud-cli-proxy.db
ADMIN_USERNAME	管理员用户名	admin
ADMIN_PASSWORD	管理员密码（bcrypt）	必填
ADMIN_JWT_SECRET	JWT 签名密钥	必填
SSH_PROXY_PORT	SSH 代理端口	2222
LOG_FORMAT	日志格式 json / text	json
LOG_LEVEL	日志级别	info
管理后台
除了环境隔离和网络策略执行，平台还提供完整的管理能力：

用户创建、暂停、过期自动停机、密码轮换

主机创建、启动、停止、重建（保留或清空 /workspace）、删除

出口 IP 增删改查、连通性测试

Bypass 防火墙——按域名、CIDR、端口白名单直连，快照版本管理，预览→应用→回滚

所有操作写入审计事件，可追溯

SSE 实时推送任务进度和主机状态

容器内置 KasmVNC + Chromium，管理后台一键打开远程桌面

架构
text
                                                    ┌───────────────────────────────────┐
用户 ──curl──> Control Plane (:8080) ──Docker──>    │ 用户容器                          │
                    │                                │  SSH + Claude Code + VNC          │
                   SQLite                             │  sshfs ← 本地 CWD 同名路径映射    │
                    │                                │  sing-box tun 隧道                │
              Admin UI (embed)                   │       ↓                           │
                    │                                │  指定出口 IP                      │
              SSH Proxy (:2222)                      └───────────────────────────────────┘
组件	说明
Control Plane	Go API，认证、用户管理、任务编排、SSH 代理
Host Agent	特权代理，管理 Docker 容器、网络命名空间和隧道
用户容器	Ubuntu 24.04，预装 OpenSSH + Claude Code + sshfs + KasmVNC + Chromium
SQLite	持久化用户、主机、出口 IP、任务、事件、审计日志（WAL 模式）
Admin SPA	React 19 + TypeScript + Vite + Tailwind CSS
合规与法律声明
本项目设计为企业内部自托管基础设施，用于满足安全、审计和合规需求。

使用本项目时，每个最终用户应使用自己的凭据进行身份验证，并遵守 Anthropic 的商业服务条款。

本项目不提供、不鼓励任何规避服务条款或绕过安全机制的行为。用户应确保其使用方式符合所在地区的法律法规及服务提供商的政策。

项目采用 MIT 许可证，作者不对因使用本项目而产生的任何法律或合规问题承担责任。

参与贡献
Bug 报告和功能建议请提交 Issue。

Pull Request 流程：

Fork 仓库，从 main 分支创建 feature 分支

修改代码，确保 make test 通过

提交 PR，描述改了什么、为什么改

本地开发环境搭建：

bash
make setup    # 安装依赖
make dev      # 后端 + 前端热重载（API :8080，前端 localhost:2568）
make test     # 运行测试
更多命令见 make help。

文档
完整文档见 GitHub Pages：快速开始、部署指南、配置参考、架构说明、API 参考、故障排查。

许可证
MIT
