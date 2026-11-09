<!-- markdownlint-disable-file MD033 MD045 -->
<div align="center">

# CodeArts2API

**直连华为云 CodeArts Agent（盘古/码道）的生产级 OpenAI 兼容高吞吐代理网关**

[![License](https://img.shields.io/badge/license-MIT-green.svg?style=flat-square)](LICENSE)
[![Go Version](https://img.shields.io/badge/Go-1.24+-00ADD8.svg?style=flat-square&logo=go&logoColor=white)](https://go.dev/)
[![Docker](https://img.shields.io/badge/Docker-Available-2496ED.svg?style=flat-square&logo=docker&logoColor=white)](Dockerfile)
[![Protocol](https://img.shields.io/badge/Protocol-OpenAI_Compatible-412991.svg?style=flat-square&logo=openai&logoColor=white)](https://platform.openai.com/)
[![Upstream](https://img.shields.io/badge/Upstream-Huawei_Cloud_CodeArts-C7000B.svg?style=flat-square&logo=huawei&logoColor=white)](https://www.huaweicloud.com/)
[![Tests](https://img.shields.io/badge/tests-passing-brightgreen.svg?style=flat-square)]()

[English](./README.md) · [简体中文](./README.zh-CN.md) · [核心特性](#核心特性) · [系统架构](#系统架构全景) · [快速上手](#快速上手) · [模型矩阵](#模型列表与限时福利) · [配置参数](#环境变量配置)

<br>

<p align="center">
  <b>彻底摆脱必须运行华为云桌面客户端与单一专属环境绑定调用的笨重束缚。</b><br>
  采用纯 Go 语言直连华为云 API 原生接口，构建了具备多账号轮转、DPoP 鉴权绑定与原子持久化的现代化网关。<br>
  提供高保真 OpenAI 标准流式与工具调用、模型套餐级别智能路由兜底、自动探活自愈以及免人工每日积分签到。
</p>

</div>

---

> **免责声明**：本项目基于公开客户端协议的逆向工程研究，仅供学习与个人技术交流使用，不承诺商业可用性；自动化调用可能违反华为云服务条款，由此产生的账号风险与责任由使用者自行承担。

---

## 痛点与架构对比

华为云 CodeArts Agent（盘古大模型助手 / 码道）拥有优秀的工程与代码生成能力，但官方仅提供 IDE 插件与桌面封闭客户端，无法直接接入现有的开源 Web 客户端、聚合中转平台（如 OneAPI / NewAPI）或自主编排工作流。CodeArts2API 实现了完全解耦：

| 评估维度 | CodeArts2API 生产网关 | 官方 IDE 插件客户端 | 传统反代逆向脚本 |
| :--- | :--- | :--- | :--- |
| **运行时依赖** | 纯 Go 独立编译二进制或轻量 Docker，内存 <25MB | 强制依赖庞大 IDE（VS Code / IntelliJ）及常驻后台插件 | 依赖 Python / Node.js 繁重运行环境 |
| **协议标准度** | 100% 契合 OpenAI 标准（流式 SSE、Tool Calls、Usage） | 私有 RPC / WebDAV / 内部 HTTP 协议，与第三方生态绝缘 | 伪造 usage 或缺失终态事件，流截断易被当作完整回答 |
| **多账号编排** | 多账号池智能并发控流、按模型注册状态自适应轮转 | 单一 IDE 仅能登录绑定单账号 | 简单 Round-Robin，遇账号无模型时全盘报错 |
| **凭证生命周期** | 严格校验 DPoP 私钥配对，自动 30s 超时换约重刷 | 手动扫码重登，会话过期后直接断连 | 伪造 24h 有效期，DPoP 密钥缺失导致 2 小时后必然瘫痪 |
| **模型探活兜底** | 周期性后台微探针，30 分钟自愈缓存，故障不惩罚账号 | 无探活，请求失败直接弹窗报错 | 每次真实请求抢占会话槽位，极易触发上游并发锁 |
| **福利与积分** | 自动发现并幂等领取限时福利 Token，自动领取每日签到积分 | 依赖人工每日进入网页点击 | 无自动化签到链路 |

---

## 核心特性

| 模块 | 特性亮点 | 技术实现与工程优势 |
| :--- | :--- | :--- |
| **生产级并发安全** | 彻底消除并发 map 冲突 | 会话状态文件先在读写锁内快照再序列化；账号写盘采用带错误日志的原子异步写并在退出时同步 flush；并发槽位显式收敛。 |
| **细粒度错误归类** | 智能区分账号故障与模型未注册 | 准确识别「账号故障」「该账号未开通此模型」「客户端参数错误」。遇未开通账号无感换号重试，全部失败才报 404；客户端断开不惩罚账号。 |
| **高保真流式协议** | 严谨的断流判定与 usage 透传 | 上游异常断流返回 `STREAM_TRUNCATED` 且绝不补发 `[DONE]`；严守 `stream_options.include_usage` 契约，优先透传上游真实 usage。 |
| **DPoP 凭据强校验** | 拒绝缺私钥的残缺凭证入库 | 强制要求凭据必须携带与 `refresh_token` 配对的 DPoP 私钥，杜绝 2 小时后刷新必然失败的假死账号；刷新带 30 秒硬超时。 |
| **智能模型可用性探测**| 互斥独立探测，不抢业务槽位 | 后台带间隔与互斥运行微探测，过滤上游已声明但实际未开通的幻觉模型；动态学习真实请求结果并建立 30 分钟自愈缓存。 |
| **限时福利自动领取** | 解决 benefit not found 报错 | 自动检测并幂等领取上游免费套餐模型福利；请求时按命中账号的真实开通情况自动补全 `maas_type: benefit` 标头。 |
| **自动化签到与积分** | 每日 1000 积分与新手奖励领取 | Watchdog 周期守护自动执行每日签到并补全确认流程；提供 Admin API 与 WebUI 手动触发接口，异常不影响推理业务。 |
| **轻量级 WebUI 控制台** | 实时账号池看板与多轮交互测试 | 内置开箱即用 Web 仪表盘，支持账号实时健康度查看、Token 余量监控、流式/非流式即时调试与 chat_id 续接。 |

---

## 系统架构全景

```text
┌────────────────────────────────────────────────────────────────────────┐
│                      客户端应用与生态集成接入                          │
│   NextChat · LobeChat · Chatbox · NewAPI · OneAPI · Cherry Studio      │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ 标准 OpenAI API (POST /v1/chat/completions)
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                   CodeArts2API 生产网关服务 (:7866)                    │
│  ┌──────────────────────┐ ┌──────────────────────┐ ┌────────────────┐  │
│  │ OpenAI 协议适配引擎  │ │ 动态多账号池调度器   │ │ 模型可用性缓存 │  │
│  │ SSE 格式化 / Tools   │ │ 故障降级 / 无感换号 │ │ 30m 探活自愈   │  │
│  └──────────┬───────────┘ └──────────┬───────────┘ └────────┬───────┘  │
│             │                        │                      │          │
│             ▼                        ▼                      ▼          │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │       华为云 CodeArts 上游通信适配层 (DPoP 签名与 OAuth 状态机)  │  │
│  │   Token 自动续期 · 每日签到 Watchdog · 幂等领取限时福利 Token   │  │
│  └──────────────────────────────────┬───────────────────────────────┘  │
└─────────────────────────────────────┼──────────────────────────────────┘
                                      │ HTTPS / SSE
                                      ▼
┌────────────────────────────────────────────────────────────────────────┐
│                      华为云 CodeArts 原生云端集群                      │
│      盘古大模型助理 · GLM 系列 · Qwen 系列 · CodeArts 积分与福利系统   │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 快速上手

### 1. 源码编译

```bash
# 编译 Linux 静态二进制
make linux

# 运行单元测试
make test
```

### 2. 账号登录授权（华为云账号）

```bash
# 本机具备桌面浏览器
./login.sh

# 无桌面服务器环境：打印授权 URL，在本地浏览器打开并授权，服务端自动轮询接收凭证
./login.sh -print-only

# 授权成功后，凭证将持久化存储于 auths/codearts-{user_id}.json
```

### 3. 服务启动

```bash
# 复制默认配置
cp config.example.json config.json

# 设定自定义 API Key
export CA2A_API_KEY=$(openssl rand -hex 24)

# 启动代理服务
./bin/codearts2api -config config.json
```

### 4. 验证与调用

```bash
# 健康状态检查
curl http://127.0.0.1:7866/healthz

# 获取可用模型列表
curl http://127.0.0.1:7866/v1/models -H "Authorization: Bearer $CA2A_API_KEY"

# 发起对话请求
curl -X POST http://127.0.0.1:7866/v1/chat/completions \
  -H "Authorization: Bearer $CA2A_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "glm-5.2",
    "messages": [{"role": "user", "content": "你好，请介绍一下你自己。"}]
  }'
```

访问 `http://127.0.0.1:7866/` 即可直接打开内置 WebUI，查看各账号健康状态、配额余量并直接在网页中发起测试。

---

## 模型列表与限时福利

`/v1/models` 返回上游**精确模型 ID**（区分大小写，如 `GLM-5.2`、`Qwen3-VL-235B`），同时为含大写的 ID 自动补充小写别名（如 `glm-5.2`），两者均可直接调用。

- **限时福利模型**：免费套餐模型额外标注 `benefit: true`，发起推理时网关自动追加上游要求的 `maas_type: benefit` 标头（且严格隔离各账号的套餐归属，避免串号）。
- **必须先领取才能调用**：未领取的福利模型上游返回 `InferHub.4004.200 benefit not found`。网关默认开启自动领取（`benefit_auto_claim: true`），如需纯读模式可通过 `"benefit_auto_claim": false` 关闭。

```bash
go run ./cmd/models                 # 查看当前账号可用模型（内置 + 福利）
go run ./cmd/models -json           # 机器可读 JSON 格式输出
go run ./cmd/models -claim          # 先执行幂等领取再查询
```

---

## 自动化签到与每日积分

服务内置 Watchdog 轮询守护，自动完成官方每日签到与新用户积分领取：
- 官方每日签到为积分计费用户发放 1000 积分（30 天有效）；
- 新用户注册发放 4000 积分；
- 具备领取但未确认补发保护；异常不影响正常推理转发；
- 支持在 WebUI 手动触发或经管理 API `POST /admin/api/checkin` 远程调度。

---

## 环境变量配置

支持通过配置文件 `config.json` 或环境变量进行定制：

| 环境变量 | 说明 | 默认值 |
| :--- | :--- | :--- |
| `CA2A_API_KEY` | 网关 API 鉴权访问密钥 | 必填（无密钥拒绝启动） |
| `CA2A_LISTEN` | HTTP 服务监听地址与端口 | `:7866` |
| `CA2A_AUTH_DIR` | 账号凭证文件持久化目录 | `./auths` |
| `CA2A_STATE_FILE` | 会话与调度状态文件路径 | `./data/state.json` |
| `CA2A_DEFAULT_MODEL` | 默认回退模型名称 | `glm-5.2` |
| `CA2A_OAUTH_CALLBACK_HOST` | OAuth 授权回调外部主机名 | 自动探测 |
| `CA2A_WATCH_ENABLED` | 后台 Watchdog 调度器总开关 | `true` |
| `CA2A_WATCH_POLL_MINUTES` | 凭证刷新与签到轮询周期（分钟） | `30` |
| `CA2A_WATCH_REFRESH_SKEW` | 提前换约刷新缓冲时间（分钟） | `30` |
| `CA2A_WATCH_KEEPALIVE_INTERVAL` | 保活微心跳间隔时间（分钟） | `15` |
| `CA2A_MAX_CONCURRENT` | 单账号上游最大允许并发槽位数 | `1` |
| `CA2A_BENEFIT_AUTO_CLAIM` | 发现模型时自动领取限时福利 | `true` |
| `CA2A_CREDITS_AUTO_CLAIM` | 自动领取每日签到与新手积分 | `true` |

---

## 部署模式

### Docker 容器化部署

```bash
export CA2A_API_KEY=$(openssl rand -hex 24)
mkdir -p auths data
docker compose up -d --build
```

### Systemd 守护进程

```bash
sudo mkdir -p /opt/codearts2api && sudo cp -r bin config.example.json auths /opt/codearts2api/
sudo cp deploy/codearts2api.service /etc/systemd/system/
sudo systemctl daemon-reload && sudo systemctl enable --now codearts2api
```

---

## 参与贡献与许可证

欢迎提交 Issue 与 Pull Request！在提交代码变更前，请确保全量测试套件执行通过：

```bash
go test -v ./...
```

本项目采用 [MIT License](LICENSE) 开源协议。
