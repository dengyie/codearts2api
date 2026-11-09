<!-- markdownlint-disable-file MD033 MD045 -->
<div align="center">

# CodeArts2API

**High-Performance OpenAI-Compatible Gateway Bridging Huawei Cloud CodeArts Agent into Standard LLM APIs**

[![License](https://img.shields.io/badge/license-MIT-green.svg?style=flat-square)](LICENSE)
[![Go Version](https://img.shields.io/badge/Go-1.24+-00ADD8.svg?style=flat-square&logo=go&logoColor=white)](https://go.dev/)
[![Docker](https://img.shields.io/badge/Docker-Available-2496ED.svg?style=flat-square&logo=docker&logoColor=white)](Dockerfile)
[![Protocol](https://img.shields.io/badge/Protocol-OpenAI_Compatible-412991.svg?style=flat-square&logo=openai&logoColor=white)](https://platform.openai.com/)
[![Upstream](https://img.shields.io/badge/Upstream-Huawei_Cloud_CodeArts-C7000B.svg?style=flat-square&logo=huawei&logoColor=white)](https://www.huaweicloud.com/)
[![Tests](https://img.shields.io/badge/tests-passing-brightgreen.svg?style=flat-square)]()

[English](./README.md) · [简体中文](./README.zh-CN.md) · [Core Features](#core-features) · [Architecture](#system-architecture) · [Quick Start](#quick-start) · [Model Catalog](#model-catalog--benefit-tokens) · [Configuration](#configuration-reference)

<br>

<p align="center">
  <b>Terminates proprietary client locks and desktop dependency for Huawei Cloud CodeArts Agent.</b><br>
  Engineered in pure Go with zero external runtime dependencies, native DPoP OAuth token rotation, atomic state synchronization, and adaptive upstream queue recovery.<br>
  Delivers OpenAI-compatible SSE streaming, native tool calling, multi-account pool load balancing with model-level fallback, and automated daily credit check-ins.
</p>

</div>

---

> **Disclaimer**: This project is based on protocol reverse engineering for personal research and academic testing. It does not guarantee commercial availability. Automated API access may be subject to Huawei Cloud terms of service; users assume full operational responsibility.

---

## Motivation & Architecture Comparison

Huawei Cloud CodeArts Agent (Pangu assistant / Madao) provides competitive coding and reasoning models, but officially restricts access to proprietary IDE extensions and desktop binaries. CodeArts2API decouples these models into universal OpenAI-compatible endpoints:

| Evaluation Dimension | CodeArts2API Gateway | Official IDE Extensions | Ad-Hoc Reverse Proxies |
| :--- | :--- | :--- | :--- |
| **Runtime Footprint** | Lightweight native Go binary (<25MB RSS), zero external deps | Requires full IDE (VS Code / IntelliJ) with heavy background plugins | Heavy Python/Node.js interpreters with excessive memory draw |
| **Protocol Fidelity** | 100% OpenAI-compliant SSE, structured tool calls, and accurate usage chunks | Proprietary RPC / internal HTTP protocols isolated from third-party tools | Fabricates usage objects or omits terminal events, corrupting streams |
| **Account Orchestration** | Intelligent concurrency gating, dynamic account pooling, and model-level failover | Single-account binding locked inside IDE state | Simple round-robin failing immediately when an account lacks a model |
| **Credential Lifecycle** | Enforces DPoP private key pairing with automatic 30s timeout refresh cycles | Manual browser re-login upon session expiry | Fakes 24h validity, fatally failing token refresh after 2 hours |
| **Health Telemetry** | Mutex-guarded micro-probes with 30m self-healing cache without penalizing healthy accounts | None (crashes or displays UI error alerts) | Probes compete directly with user chat requests for upstream session slots |
| **Benefit & Quota Ops** | Auto-discovers and claims free-tier benefit models and daily check-in credits | Requires manual daily web console navigation | Lacks automated claim pipelines |

---

## Core Features

| Module | Feature Highlight | Technical Implementation & Advantage |
| :--- | :--- | :--- |
| **Concurrency Hardening** | Eliminates concurrent map write panics | Session state files are snapshotted inside read-write locks before serialization; credentials use atomic asynchronous writes with shutdown flush. |
| **Error Classification** | Granular failure routing | Distinguishes between account outages, unregistered models, and client param errors. Rotates transparently to eligible accounts; only returns 404 when exhausted. |
| **High-Fidelity Streaming** | Exact stream termination & usage parity | Returns `STREAM_TRUNCATED` on upstream drop without fabricating `[DONE]`; strictly adheres to `stream_options.include_usage` contract. |
| **DPoP Token Validation** | Strict cryptographic credential integrity | Rejects tokens lacking matching DPoP private keys to prevent expired accounts from wedging the pool; refreshes feature a 30s timeout guard. |
| **Model Probing Engine** | Isolated non-interfering background discovery | Runs periodic health probes independently from user sessions; filters out ghost models announced by upstream that fail real inference. |
| **Benefit Auto-Claim** | Resolves benefit not found errors | Idempotently claims free-tier promotional models and dynamically signs `maas_type: benefit` request headers per account capability. |
| **Daily Credit Watchdog** | Automated check-in and new-user bonus | Background watchdog claims daily 1,000 credit allowances and pending rewards with auto-confirmation; exposes admin REST and WebUI hooks. |
| **Integrated WebUI** | Real-time fleet dashboard & testing | Built-in zero-dependency management interface for inspecting account health, token remaining time, and running interactive chat sessions. |

---

## System Architecture

```text
┌────────────────────────────────────────────────────────────────────────┐
│                        Client Applications                             │
│   NextChat · LobeChat · Chatbox · NewAPI · OneAPI · Cherry Studio      │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ Standard OpenAI API (POST /v1/chat/completions)
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                   CodeArts2API Production Gateway (:7866)              │
│  ┌──────────────────────┐ ┌──────────────────────┐ ┌────────────────┐  │
│  │ OpenAI Protocol Hub  │ │ Dynamic Account Pool │ │ Probing Cache  │  │
│  │ SSE Formatter / Tools│ │ Failover & Rotation  │ │ 30m TTL Cache  │  │
│  └──────────┬───────────┘ └──────────┬───────────┘ └────────┬───────┘  │
│             │                        │                      │          │
│             ▼                        ▼                      ▼          │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │          Huawei Cloud CodeArts Protocol Translation Layer        │  │
│  │    DPoP Key Signing · Token Auto-Refresh · Watchdog Scheduler    │  │
│  └──────────────────────────────────┬───────────────────────────────┘  │
└─────────────────────────────────────┼──────────────────────────────────┘
                                      │ HTTPS / SSE
                                      ▼
┌────────────────────────────────────────────────────────────────────────┐
│                     Huawei Cloud CodeArts Cloud Fleet                  │
│       Pangu Models · GLM Suite · Qwen Suite · Credit System            │
└────────────────────────────────────────────────────────────────────────┘
```

---

## Quick Start

### 1. Build from Source

```bash
# Compile static Linux binary
make linux

# Run full test suite
make test
```

### 2. Account Authentication (Huawei Cloud)

```bash
# On machines with a desktop browser
./login.sh

# On headless servers: prints authorization link to open locally; polls ticket automatically
./login.sh -print-only

# Saved credentials persist to auths/codearts-{user_id}.json
```

### 3. Launch Gateway

```bash
# Copy example configuration
cp config.example.json config.json

# Set mandatory API key
export CA2A_API_KEY=$(openssl rand -hex 24)

# Launch gateway daemon
./bin/codearts2api -config config.json
```

### 4. Verification & Usage

```bash
# Health check
curl http://127.0.0.1:7866/healthz

# List verified active models
curl http://127.0.0.1:7866/v1/models -H "Authorization: Bearer $CA2A_API_KEY"

# Chat completion request
curl -X POST http://127.0.0.1:7866/v1/chat/completions \
  -H "Authorization: Bearer $CA2A_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "glm-5.2",
    "messages": [{"role": "user", "content": "Hello, describe your capabilities."}]
  }'
```

Access `http://127.0.0.1:7866/` in a web browser to view account pool health, monitor token expiry, and run interactive streaming dialogues.

---

## Model Catalog & Benefit Tokens

`/v1/models` outputs upstream **case-sensitive model IDs** (e.g. `GLM-5.2`, `Qwen3-VL-235B`), alongside automatic lowercase aliases (`glm-5.2`):

- **Promotional Benefit Models**: Free-tier models carry `benefit: true`. During chat dispatch, the gateway automatically appends upstream's required `maas_type: benefit` header based on the selected account's entitlement.
- **Mandatory Pre-Claim**: Unclaimed benefit models reject requests with `InferHub.4004.200 benefit not found`. The gateway automatically claims them upon discovery (`benefit_auto_claim: true`).

```bash
go run ./cmd/models                 # Inspect available models for current credentials
go run ./cmd/models -json           # Machine-readable output
go run ./cmd/models -claim          # Perform idempotent claim prior to query
```

---

## Automated Check-in & Daily Credits

The internal watchdog periodically inspects promotional campaigns and performs automated check-ins:
- Awards 1,000 daily check-in credits for credit-billing enrolled accounts (valid 30 days);
- Awards 4,000 new user onboarding credits;
- Includes auto-confirmation guarantees; campaign hiccups never throttle inference traffic;
- Manual trigger available via WebUI or via authenticated administrative API: `POST /admin/api/checkin`.

---

## Configuration Reference

Settings can be managed via `config.json` or overridden via environment variables:

| Environment Variable | Description | Default |
| :--- | :--- | :--- |
| `CA2A_API_KEY` | Gateway API bearer access token | Required (refuses launch without key) |
| `CA2A_LISTEN` | Service bind address and port | `:7866` |
| `CA2A_AUTH_DIR` | Directory containing account JSON credentials | `./auths` |
| `CA2A_STATE_FILE` | Runtime state and session cache path | `./data/state.json` |
| `CA2A_DEFAULT_MODEL` | Fallback model identifier | `glm-5.2` |
| `CA2A_OAUTH_CALLBACK_HOST` | External hostname for OAuth redirection | Auto-detected |
| `CA2A_WATCH_ENABLED` | Master toggle for background watchdog | `true` |
| `CA2A_WATCH_POLL_MINUTES` | Watchdog polling interval in minutes | `30` |
| `CA2A_WATCH_REFRESH_SKEW` | Proactive token renewal buffer in minutes | `30` |
| `CA2A_WATCH_KEEPALIVE_INTERVAL` | Micro-heartbeat interval in minutes | `15` |
| `CA2A_MAX_CONCURRENT` | Maximum concurrent upstream slots per account | `1` |
| `CA2A_BENEFIT_AUTO_CLAIM` | Idempotently claim benefit models on discovery | `true` |
| `CA2A_CREDITS_AUTO_CLAIM` | Auto-claim daily check-in and onboarding credits | `true` |

---

## Deployment Modes

### Docker Compose

```bash
export CA2A_API_KEY=$(openssl rand -hex 24)
mkdir -p auths data
docker compose up -d --build
```

### Systemd Service

```bash
sudo mkdir -p /opt/codearts2api && sudo cp -r bin config.example.json auths /opt/codearts2api/
sudo cp deploy/codearts2api.service /etc/systemd/system/
sudo systemctl daemon-reload && sudo systemctl enable --now codearts2api
```

---

## Contributing & License

Contributions and pull requests are welcome. Please ensure all tests pass prior to submitting changes:

```bash
go test -v ./...
```

Licensed under the [MIT License](LICENSE).
