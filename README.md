# Binance Agentic Wallet Dashboard

**Binance Agentic 钱包自托管看板｜中文优先**

> 本项目是独立自托管工具，不是 Binance 官方产品，与 Binance 无隶属关系。

---

# 中文版

## 简介

这是一个运行在 BNB Smart Chain（BSC）上的 **Binance Agentic Wallet（BAW）多钱包操作看板**。

它把多个 Agentic 钱包集中到一个页面里：总资产、各钱包余额、实时 DEX 价格、最优池卖出保护、发送、钱包互转、本地地址簿、收款二维码和后台保活。

适合自用、自己部署、自己掌握会话和私钥边界。

## 功能

- **多钱包总览**：全部钱包汇总，也可以只看单个钱包。
- **实时价格**：使用 DexScreener 最深流动性池，显示池子、流动性和交易税。
- **卖出保护**：BAW 报价和最优池价格对比，低于最优池 8% 自动拦截。
- **按比例 / 按数量卖出**：两种模式二选一；每个比例下面显示对应数量，输入数量时显示约占比。
- **发送**：支持手动数量或百分比；可以从一个钱包发给另一个自己的 Agentic 钱包。
- **钱包互转**：发送弹窗里直接选择 W1 → W2、W1 → W3、W1 → W4。
- **本地地址簿**：只存在浏览器 localStorage；支持新增、修改、复制、发送、删除。
- **完整地址复制**：钱包管理和地址簿都复制完整 `0x` 地址，不复制缩略地址。
- **收款二维码**：服务端按当前钱包完整地址生成二维码，二维码下方再次显示完整地址。
- **风险过滤**：默认隐藏小额、低流动性、貔貅、高税代币；过滤详情可展开。
- **交易税提醒**：买卖税显示在持仓表和卖出弹窗；卖出税高于 10% 会再额外确认一次。
- **后台保活**：每小时对每个会话执行 `baw wallet status`；右上角绿/红状态，点开看每个钱包。
- **钱包管理**：重命名、复制地址、扫码添加、从看板移除。
- **双端布局**：桌面端、手机竖屏、手机横屏分别适配。

## 工作方式

```text
浏览器（单页看板）
        |
        | HTTPS + 你自己的登录鉴权
        v
Caddy / 反向代理
        |
        | 127.0.0.1:8100
        v
Python 标准库 HTTP 服务（server.py）
        |
        +-- BAW CLI 会话（~/aw/w1 ... wN）
        +-- DexScreener API（价格 / 最深池）
        +-- honeypot.is API（买卖税 / 貔貅检测）
        +-- BSC RPC（原生余额）
```

浏览器只访问同源 `/api/*`；钱包地址、BAW 会话和配置文件不会离开服务器。

## 安全说明

- **不接触私钥**：不导入、不保存私钥或助记词，只调用已经登录的 BAW 会话。
- **不内置登录**：必须放在你自己的鉴权、SSO、VPN 或 forward-auth 后面，不要直接暴露 `8100` 端口。
- **仓库不含真实地址**：默认钱包列表为空；请通过页面扫码或自己的 `data/wallets.json` 配置。
- **二维码只编码地址**：`/api/qr` 只接受合法的 `0x` + 40 位地址；二维码下方同时显示完整地址，扫码后仍需核对。
- **交易保护**：报价偏离最优池超过 8% 会拦截；高税代币需要额外确认。
- **本地地址簿**：地址簿只保存在当前浏览器，不上传服务器。
- **不可撤销**：链上转账和交易不可撤销，提交前请核对地址、网络、代币、数量、滑点和税。

## 环境要求

- Linux（Ubuntu 22.04 / 24.04 测试通过）
- Python **3.12+**
- Binance Agentic Wallet CLI **`baw` 1.12.0**
- 每个钱包一个 BAW 会话目录，例如 `~/aw/w1`、`~/aw/w2`
- 可访问 DexScreener、honeypot.is 和 BSC RPC 的网络

Python 依赖：

```bash
pip install -r requirements.txt
```

## 快速开始

```bash
git clone https://github.com/Ssw7777/binance-agentic-wallet-dashboard.git
cd binance-agentic-wallet-dashboard

python3 -m venv .venv
. .venv/bin/activate
pip install -r requirements.txt

mkdir -p data
cp config/wallets.example.json data/wallets.json

# 为每个钱包登录一个独立 BAW 会话（示例）：
# BINANCE_BAW_DIR="$HOME/aw/w1" baw auth signin

python3 server.py
```

打开 `http://127.0.0.1:8100/`，或者通过你自己的带鉴权反向代理访问。

### 添加钱包

1. 安装并登录 BAW。
2. 打开看板 → **钱包管理 → 添加**。
3. 输入显示名称，用币安 App 扫码。
4. 服务端会创建新的 `wN` 会话目录，并写入 `data/wallets.json`。

也可以直接编辑 `data/wallets.json`：

```json
{
  "nextSlot": 7,
  "wallets": [
    {
      "id": "w1",
      "label": "主钱包",
      "agent": "main",
      "address": "0x0000000000000000000000000000000000000000",
      "color": "#d4654a",
      "trade": true
    }
  ]
}
```

`address` 必须是该 BAW 会话对应的真实 BSC 地址。示例文件故意使用空钱包列表。

## 部署到 VPS（推荐：长期后台运行）

> 目标：部署在 VPS 上，24 小时可访问；服务器重启后自动启动；不需要你本地电脑开机。

下面以 Ubuntu 24.04、域名 `wallet.example.com`、运行用户 `ubuntu` 为例。

### 1. 准备 VPS 和域名

- 一台 1 核 1G 以上的 Linux VPS
- Ubuntu **24.04**（自带 Python 3.12；22.04 需要自行安装 Python 3.12）
- 域名 A 记录指向 VPS 公网 IP
- 安装 Caddy（用于 HTTPS 和反向代理）
- 按 Binance Agentic Wallet 官方方式安装 `baw`，确认 `baw --version` 可用

```bash
sudo apt update
sudo apt install -y python3 python3-venv python3-pip git curl ufw
```

Caddy 请通过官方 apt 源安装，安装后确认：

```bash
caddy version
```

### 2. 拉取项目并安装依赖

```bash
sudo mkdir -p /opt/binance-agentic-wallet
sudo chown "$USER":"$USER" /opt/binance-agentic-wallet

git clone https://github.com/Ssw7777/binance-agentic-wallet-dashboard.git /opt/binance-agentic-wallet
cd /opt/binance-agentic-wallet

python3 -m venv .venv
. .venv/bin/activate
pip install -r requirements.txt

mkdir -p data
cp config/wallets.example.json data/wallets.json
```

### 3. 登录 BAW 会话

每个钱包一个独立会话目录：

```bash
export BINANCE_BAW_DIR="$HOME/aw/w1"
baw auth signin

export BINANCE_BAW_DIR="$HOME/aw/w2"
baw auth signin
```

重复到所有钱包都登录完成。二维码登录需要在币安 App 确认。

### 4. 先手动启动测试

```bash
cd /opt/binance-agentic-wallet
. .venv/bin/activate
python3 server.py
```

另开一个终端：

```bash
curl http://127.0.0.1:8100/api/health
```

看到正常 JSON 后，按 `Ctrl+C` 停止。服务只监听 `127.0.0.1:8100`，不要直接开放这个端口。

### 5. 用 systemd 常驻 + 开机自启

项目已提供 `deploy/agentic-wallet.service`：

```bash
sudo cp deploy/agentic-wallet.service /etc/systemd/system/agentic-wallet.service
sudo systemctl daemon-reload
sudo systemctl enable --now agentic-wallet
```

检查状态和日志：

```bash
systemctl status agentic-wallet
journalctl -u agentic-wallet -f
```

关键配置：

- `Restart=always`：进程异常退出会自动拉起
- `systemctl enable`：VPS 重启后自动启动
- `ReadWritePaths`：只允许写 `data/` 和 `~/aw/`
- `UMask=0077`：数据文件默认仅服务用户可读

如果你的用户名不是 `ubuntu`，请同步修改 service 文件里的 `User`、`Group`、`HOME`、`WorkingDirectory` 和 `ReadWritePaths`。

### 6. 配置域名、HTTPS 和登录鉴权

**本项目没有内置登录系统**，必须加鉴权。

#### 方案 A：独立子域名（最简单）

```caddyfile
wallet.example.com {
    encode zstd gzip

    # 必须加鉴权：forward_auth / SSO / VPN / basic_auth 任选一种
    basic_auth {
        admin <用 caddy hash-password 生成的密码哈希>
    }

    reverse_proxy 127.0.0.1:8100
}
```

生成密码哈希：

```bash
caddy hash-password
```

把输出填到 `basic_auth` 里。更推荐把 `basic_auth` 换成你现有的 forward-auth / SSO / VPN。

```bash
sudo systemctl reload caddy
```

访问：`https://wallet.example.com/`

#### 方案 B：挂到现有网站的 `/agentic/` 路径

如果已有网站 `https://example.com`，可以在现有 Caddy site block 中加入：

```caddyfile
example.com {
    # ...你原有的站点配置...

    redir /agentic /agentic/ 308

    handle_path /agentic/* {
        # 这里也建议复用你原有的 forward_auth / SSO
        reverse_proxy 127.0.0.1:8100
    }
}
```

访问：`https://example.com/agentic/`

注意：

- 必须保留结尾斜杠 `/agentic/`
- `handle_path` 会去掉 `/agentic` 前缀，后端才能收到 `/api/*`
- 前端使用相对路径 `api/...`，所以子路径部署可以正常工作
- 不要配置成把 `/agentic` 原样转给后端，否则 API 会 404

### 7. 防火墙

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

**不要开放 8100 端口。** 只允许 Caddy 在本机访问它。

### 8. 更新版本

```bash
cd /opt/binance-agentic-wallet
git pull
. .venv/bin/activate
pip install -r requirements.txt
sudo systemctl restart agentic-wallet
```

### 9. 备份和会话过期

- 备份 `data/`（钱包配置、历史缓存、审计记录）
- 备份 `~/aw/` 前先确认权限；里面是 BAW 会话，属于敏感数据
- Binance 会话过期时，手机 App 重新扫码登录对应 `wN`
- 保活每小时执行一次 `baw wallet status`，但**不能代替币安自身的登录有效期**
- 不要把 `data/`、`~/aw/`、Cookie、密码或二维码截图提交到 GitHub

### 10. 常驻运行检查清单

- [ ] `systemctl is-enabled agentic-wallet` 返回 `enabled`
- [ ] `systemctl is-active agentic-wallet` 返回 `active`
- [ ] `curl http://127.0.0.1:8100/api/health` 返回 200
- [ ] 域名 HTTPS 正常
- [ ] 未登录访问会被鉴权拦截
- [ ] 8100 没有对公网开放
- [ ] 服务器重启后页面仍能打开

## 反向代理和鉴权

本项目**没有内置用户登录**，请放在带鉴权的代理后面。

最小 Caddy 示例：

```caddyfile
wallet.example.com {
    redir /agentic /agentic/ 308
    handle_path /agentic/* {
        reverse_proxy 127.0.0.1:8100
    }
}
```

生产环境请加 forward-auth、SSO、VPN 或等效鉴权。更多示例见 `deploy/Caddyfile.example`。

### systemd

```bash
sudo cp deploy/agentic-wallet.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now agentic-wallet
```

按你的机器修改 `User`、`WorkingDirectory`、`ReadWritePaths` 和 `HOME`。

## 配置和运行数据

运行数据不进入 Git：

```text
data/
├── wallets.json          # 钱包配置（包含地址）
├── keepalive.json
├── history/              # 每个钱包的交易记录缓存
├── audit.jsonl           # 本地交易审计
├── balance-cache.json    # 短期余额快照
└── market-cache.json     # 短期价格快照
```

`data/`、`.env` 和 BAW 会话目录都不要提交。不要提交真实钱包地址、Cookie、会话文件或凭证。

## API 一览

| 方法 | 路径 | 用途 |
| --- | --- | --- |
| `GET` | `/api/portfolio` | 汇总资产、价格、税、保活 |
| `GET` | `/api/history?wallet=w1` | 缓存的 BAW 交易记录 |
| `GET` | `/api/wallets` | 钱包配置 |
| `POST` | `/api/wallets/start` | 开始扫码添加钱包 |
| `GET` | `/api/wallets/pending?slot=w7` | 查询扫码状态 |
| `POST` | `/api/wallets/<id>/rename` | 重命名钱包 |
| `DELETE` | `/api/wallets/<id>` | 从看板移除钱包 |
| `POST` | `/api/quote` | BAW 卖出报价 + 最优池校验 |
| `POST` | `/api/sell` | 提交卖出任务 |
| `POST` | `/api/send` | 提交发送任务 |
| `GET` | `/api/task/<id>` | 查询任务进度 |
| `GET` | `/api/qr?text=0x…` | 收款二维码（只接受完整地址） |
| `GET` | `/api/health` | 服务健康状态 |

修改类请求需要 `X-Requested-With: agentic`，请求体为 JSON。

## 交易规则摘要

- 价格参考：DexScreener 最深流动性池。
- 卖出：BAW `market-order swap`；发送：BAW `wallet send`。
- 报价偏离最优池低于 **-8%**：拦截。
- 偏离在 **-3% 到 -8%**：提示。
- 卖出税 **> 10%**：卖出前额外确认。
- 貔貅 / 卖出税 **> 50%**：默认隐藏。
- 保活只执行 `wallet status`，不会自动买卖或转账。

## 免责声明

本项目仅供自托管、个人学习和自用。加密资产和链上交易不可撤销；价格、流动性、税费、报价和余额都可能延迟或出错。每次操作前请在币安 App 和区块浏览器核对收款地址、网络、代币合约、数量、滑点和税。作者不对任何资金损失负责。

本项目不是 Binance 官方产品，与 Binance 无隶属或背书关系。

## 许可证

MIT，详见 [LICENSE](LICENSE)。

**唯一贡献者：Ssw7777**

---

# English Version

## Overview

A self-hosted **Binance Agentic Wallet (BAW)** dashboard for **BNB Smart Chain (BSC)**.

It brings multiple Agentic wallets into one operational view: total portfolio, per-wallet balances, live DEX prices, guarded sells, transfers, wallet-to-wallet recipients, a local address book, receive QR codes, and background keepalive.

Independent project. Not affiliated with or endorsed by Binance.

## Features

- Multi-wallet portfolio, aggregate or single-wallet view.
- Live DexScreener pricing using the deepest-liquidity pool.
- Best-pool sell guard: BAW quotes more than 8% below the reference are blocked.
- Sell by percentage or by token quantity, with quantity labels and estimated percentages.
- Manual or percentage-based sends.
- Send directly to another connected Agentic wallet (W1 → W2, W1 → W3, etc.).
- Local browser address book with add, edit, copy, send, and delete.
- Full-address copy in wallet management and the address book.
- Server-generated receive QR codes from the exact wallet address, with the full address shown underneath.
- Risk filtering for dust, low liquidity, honeypots, and high-tax tokens, with collapsible details.
- Buy/sell tax visibility; sells above 10% tax require a second confirmation.
- Hourly keepalive using `baw wallet status`, with green/red status in the header.
- Wallet rename, address copy, QR onboarding, and dashboard removal.
- Desktop, phone portrait, and phone landscape layouts.

## Architecture

```text
Browser (single-page UI)
        |
        | HTTPS + your auth layer
        v
Caddy / reverse proxy
        |
        | 127.0.0.1:8100
        v
Python stdlib HTTP server (server.py)
        |
        +-- BAW CLI sessions (~/aw/w1 ... wN)
        +-- DexScreener API      (price / deepest pool)
        +-- honeypot.is API      (buy/sell tax / honeypot simulation)
        +-- BSC RPC              (native balance)
```

## Security

- **No private keys.** The dashboard drives existing BAW sessions through the `baw` CLI.
- **No built-in authentication.** Run it behind your own SSO, forward-auth, VPN, or equivalent. Never expose port 8100 directly.
- **No real wallet addresses in this repository.** `DEFAULT_WALLETS` is empty; configure via the QR flow or `data/wallets.json`.
- **Exact-address QR.** `/api/qr` accepts only a valid `0x` + 40-hex address and displays the full address below the QR.
- **Trade guard.** Sells below the -8% best-pool threshold are blocked; high-tax sells require an extra confirmation.
- **Local address book.** Saved addresses stay in browser `localStorage`.

## Requirements

- Linux (tested on Ubuntu 22.04/24.04)
- Python 3.12+
- Binance Agentic Wallet CLI `baw` 1.12.0
- One BAW session directory per wallet, e.g. `~/aw/w1`
- Outbound access to DexScreener, honeypot.is, and BSC RPC

## Quick start

```bash
git clone https://github.com/Ssw7777/binance-agentic-wallet-dashboard.git
cd binance-agentic-wallet-dashboard

python3 -m venv .venv
. .venv/bin/activate
pip install -r requirements.txt

mkdir -p data
cp config/wallets.example.json data/wallets.json

# Authenticate one BAW session per wallet:
# BINANCE_BAW_DIR="$HOME/aw/w1" baw auth signin

python3 server.py
```

Open `http://127.0.0.1:8100/` locally, or put it behind an authenticated reverse proxy.

### Add wallets

1. Install and authenticate BAW.
2. Open the dashboard → **Wallet management → Add**.
3. Enter a display name and scan the Binance App QR.
4. The server creates a new `wN` session directory and updates `data/wallets.json`.

Wallets can also be configured directly:

```json
{
  "nextSlot": 7,
  "wallets": [
    {
      "id": "w1",
      "label": "Main wallet",
      "agent": "main",
      "address": "0x0000000000000000000000000000000000000000",
      "color": "#d4654a",
      "trade": true
    }
  ]
}
```

The example file intentionally contains an empty wallet list.

## Deploy on a VPS (recommended, always-on)

> Goal: run on a VPS 24/7, survive reboots, and not depend on your laptop being powered on.

Example assumptions: Ubuntu 24.04, domain `wallet.example.com`, Linux user `ubuntu`.

### 1. Prepare the host

- Linux VPS with at least 1 CPU / 1 GB RAM
- Ubuntu 24.04 includes Python 3.12; Ubuntu 22.04 requires a separate Python 3.12 install
- DNS A record for the dashboard domain
- Caddy installed from the official apt repository
- `baw` installed and authenticated; verify with `baw --version`

```bash
sudo apt update
sudo apt install -y python3 python3-venv python3-pip git curl ufw
```

### 2. Install the app

```bash
sudo mkdir -p /opt/binance-agentic-wallet
sudo chown "$USER":"$USER" /opt/binance-agentic-wallet

git clone https://github.com/Ssw7777/binance-agentic-wallet-dashboard.git /opt/binance-agentic-wallet
cd /opt/binance-agentic-wallet

python3 -m venv .venv
. .venv/bin/activate
pip install -r requirements.txt

mkdir -p data
cp config/wallets.example.json data/wallets.json
```

### 3. Authenticate BAW sessions

```bash
export BINANCE_BAW_DIR="$HOME/aw/w1"
baw auth signin

export BINANCE_BAW_DIR="$HOME/aw/w2"
baw auth signin
```

Repeat for every wallet. Confirm each login in the Binance App.

### 4. Smoke test

```bash
cd /opt/binance-agentic-wallet
. .venv/bin/activate
python3 server.py
```

In another shell:

```bash
curl http://127.0.0.1:8100/api/health
```

Stop with `Ctrl+C` after a successful response. The server binds to `127.0.0.1:8100` only.

### 5. Run as a systemd service (auto-start on boot)

```bash
sudo cp deploy/agentic-wallet.service /etc/systemd/system/agentic-wallet.service
sudo systemctl daemon-reload
sudo systemctl enable --now agentic-wallet
```

```bash
systemctl status agentic-wallet
journalctl -u agentic-wallet -f
```

The supplied unit uses `Restart=always`, `UMask=0077`, and restricts writes to `data/` and `~/aw/`. Adjust `User`, `Group`, `HOME`, `WorkingDirectory`, and `ReadWritePaths` if your user or paths differ.

### 6. Domain, HTTPS, and authentication

The app has **no built-in login**. Add authentication in front of it.

#### Option A: dedicated subdomain

```caddyfile
wallet.example.com {
    encode zstd gzip

    # Replace with forward-auth / SSO / VPN or basic_auth.
    basic_auth {
        admin <hash generated by caddy hash-password>
    }

    reverse_proxy 127.0.0.1:8100
}
```

```bash
caddy hash-password
sudo systemctl reload caddy
```

Open `https://wallet.example.com/`.

#### Option B: existing website at `/agentic/`

```caddyfile
example.com {
    # ...your existing site configuration...

    redir /agentic /agentic/ 308

    handle_path /agentic/* {
        # Reuse your existing forward_auth / SSO here.
        reverse_proxy 127.0.0.1:8100
    }
}
```

Open `https://example.com/agentic/`.

Requirements: keep the trailing slash; let `handle_path` strip `/agentic` so the backend receives `/api/*`; do not proxy `/agentic` unchanged or the API will 404.

### 7. Firewall

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

Do **not** expose port 8100 to the internet.

### 8. Updates

```bash
cd /opt/binance-agentic-wallet
git pull
. .venv/bin/activate
pip install -r requirements.txt
sudo systemctl restart agentic-wallet
```

### 9. Backups and session expiry

- Back up `data/` (wallet config, history cache, audit log).
- Treat `~/aw/` as sensitive BAW session state.
- Re-scan the Binance App QR when a session expires.
- Hourly keepalive runs `baw wallet status`; it does not replace Binance login expiry.
- Never commit `data/`, `~/aw/`, cookies, passwords, or QR screenshots.

### 10. Always-on checklist

- [ ] `systemctl is-enabled agentic-wallet` → `enabled`
- [ ] `systemctl is-active agentic-wallet` → `active`
- [ ] `curl http://127.0.0.1:8100/api/health` → 200
- [ ] HTTPS domain works
- [ ] Unauthenticated access is blocked
- [ ] Port 8100 is not publicly reachable
- [ ] The dashboard still loads after a VPS reboot

## Reverse proxy and auth

The app has **no built-in user authentication**. Put it behind an authenticated proxy.

Minimal Caddy example:

```caddyfile
wallet.example.com {
    redir /agentic /agentic/ 308
    handle_path /agentic/* {
        reverse_proxy 127.0.0.1:8100
    }
}
```

Add forward-auth / SSO / VPN in production. See `deploy/Caddyfile.example`.

### systemd

```bash
sudo cp deploy/agentic-wallet.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now agentic-wallet
```

Adjust `User`, `WorkingDirectory`, `ReadWritePaths`, and `HOME` for your host.

## API

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/api/portfolio` | Portfolio, prices, taxes, keepalive |
| `GET` | `/api/history?wallet=w1` | Cached BAW transaction history |
| `GET` | `/api/wallets` | Wallet configuration |
| `POST` | `/api/wallets/start` | Start QR onboarding |
| `GET` | `/api/wallets/pending?slot=w7` | Pending QR status |
| `POST` | `/api/wallets/<id>/rename` | Rename a wallet |
| `DELETE` | `/api/wallets/<id>` | Remove a wallet from the dashboard |
| `POST` | `/api/quote` | BAW sell quote with best-pool guard |
| `POST` | `/api/sell` | Queue a guarded sell |
| `POST` | `/api/send` | Queue a token send |
| `GET` | `/api/task/<id>` | Poll task progress |
| `GET` | `/api/qr?text=0x…` | Receive QR (address only) |
| `GET` | `/api/health` | Service health |

Mutating requests require `X-Requested-With: agentic` and JSON bodies.

## Trading rules

- Price reference: DexScreener deepest-liquidity pool.
- Sells use BAW `market-order swap`; sends use `wallet send`.
- Quote below **-8%** vs. best pool: blocked.
- **-3% to -8%**: warning.
- Sell tax **> 10%**: extra confirmation.
- Honeypot / sell tax **> 50%**: hidden by default.
- Keepalive only runs `wallet status`; it never trades or transfers.

## Disclaimer

Provided for self-hosted personal use and learning. Crypto assets and on-chain transactions are irreversible. Prices, liquidity, taxes, quotes, and balances may be delayed or wrong. Always verify the recipient address, network, token contract, amount, slippage, and tax in Binance and on-chain before confirming. The authors are not responsible for financial loss.

Not affiliated with or endorsed by Binance.

## License

MIT — see [LICENSE](LICENSE).

**Sole contributor: Ssw7777**
