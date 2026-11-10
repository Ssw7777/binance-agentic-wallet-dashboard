# Binance Agentic Wallet Dashboard

A self-hosted dashboard for **Binance Agentic Wallet (BAW)** on **BNB Smart Chain (BSC)**.

It brings multiple Agentic wallets into one operational view: portfolio balances, live DEX prices, guarded sells, transfers, wallet-to-wallet recipients, a local address book, and receive QR codes.

> This is an independent, self-hosted tool. It is not an official Binance product and is not affiliated with Binance.

---

## Highlights

- **Multi-wallet portfolio** — aggregate or per-wallet balances.
- **Live DEX pricing** — DexScreener deepest-liquidity pool, with the selected pool and liquidity shown.
- **Pool-aware sells** — BAW quote is checked against the best pool; deviations below -8% are blocked.
- **Manual size controls** — sell by percentage or by token quantity; send by manual amount or percentage.
- **Wallet-to-wallet transfers** — choose one of your own Agentic wallets as the recipient.
- **Local address book** — stored in browser `localStorage`, with edit, copy, send, and delete actions.
- **Receive QR codes** — generated server-side from the exact wallet address and displayed with the full address underneath.
- **Wallet management** — rename, copy the full address, add via Binance App QR, or remove from the dashboard.
- **Risk filtering** — hides dust, low-liquidity, honeypot, and high-tax tokens by default; the filter details are collapsible.
- **Trading tax visibility** — buy/sell tax is shown in the holdings table and sell flow; a second confirmation is required above 10% sell tax.
- **Background keepalive** — hourly `baw wallet status` for each session, with a green/red status entry in the header.
- **Desktop + phone layout** — compact desktop view; phone portrait cards and landscape breakpoints for small screens.

---

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
        +-- DexScreener API      (prices / deepest pool)
        +-- honeypot.is API      (buy/sell tax / honeypot simulation)
        +-- BSC RPC              (native balance checks)
```

The browser only calls same-origin `/api/*` endpoints. Wallet data and BAW session paths never leave the server.

---

## Security model

- **No private keys.** The dashboard never imports or stores private keys. It drives existing Binance Agentic Wallet sessions through the `baw` CLI.
- **No wallet addresses are shipped in this repository.** `DEFAULT_WALLETS` is empty; configure wallets through the UI QR flow or `data/wallets.json`.
- **Loopback only.** `server.py` binds `127.0.0.1:8100`. Put it behind your own authentication (SSO, forward-auth, VPN, or equivalent). Do **not** expose port 8100 directly.
- **Exact-address QR.** `/api/qr` accepts only a `0x` + 40-hex address, generates the QR from that exact string, and returns `image/svg+xml`. The UI also prints the full address under the QR.
- **Copy is exact.** Wallet and address-book copy actions copy the complete address, never the truncated display form.
- **Trade guard.** Sells are blocked when the BAW quote is more than 8% below the DexScreener best-pool reference. High-tax sells require an additional confirmation.
- **Local address book.** Saved addresses live in the browser only; they are not uploaded to the server.

See [SECURITY.md](SECURITY.md) for operational guidance.

---

## Requirements

- Linux host (tested on Ubuntu 22.04/24.04)
- Python **3.12+**
- Binance Agentic Wallet CLI **`baw` 1.12.0** installed and authenticated
- One BAW session directory per wallet, e.g. `~/aw/w1`, `~/aw/w2`
- Outbound HTTPS access to DexScreener, honeypot.is, and BSC RPC endpoints

Python dependencies:

```bash
pip install -r requirements.txt
```

---

## Quick start

```bash
git clone https://github.com/<your-account>/binance-agentic-wallet-dashboard.git
cd binance-agentic-wallet-dashboard

python3 -m venv .venv
. .venv/bin/activate
pip install -r requirements.txt

mkdir -p data
cp config/wallets.example.json data/wallets.json

# Authenticate one BAW session per wallet (example):
# BINANCE_BAW_DIR=$HOME/aw/w1 baw auth signin

python3 server.py
```

Open `http://127.0.0.1:8100/` locally, or put it behind your authenticated reverse proxy.

### Add wallets

1. Install/authenticate BAW for the wallet.
2. Open the dashboard → **钱包管理 → 添加**.
3. Enter a display name and scan the Binance App QR.
4. The server creates a new `wN` session directory and adds the wallet to `data/wallets.json`.

Wallets can also be configured directly in `data/wallets.json`:

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

`address` is the BSC address displayed by the dashboard. Use the real address from the authenticated BAW session.

> The example file ships with an empty wallet list on purpose.

---

## Reverse proxy / authentication

The app has **no built-in user authentication**. It is designed to sit behind an authenticated proxy.

Minimal Caddy example:

```caddyfile
wallet.example.com {
    redir /agentic /agentic/ 308
    handle_path /agentic/* {
        reverse_proxy 127.0.0.1:8100
    }
}
```

For a production deployment, add forward-auth / SSO / VPN in front of this block. See `deploy/Caddyfile.example`.

### systemd

```bash
sudo cp deploy/agentic-wallet.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now agentic-wallet
```

Adjust `User`, `WorkingDirectory`, `ReadWritePaths`, and `HOME` for your host.

---

## API overview

All routes are same-origin and intended to be protected by the proxy.

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/api/portfolio` | Aggregated portfolio, prices, taxes, keepalive |
| `GET` | `/api/history?wallet=w1` | Cached BAW transaction history |
| `GET` | `/api/wallets` | Wallet configuration |
| `POST` | `/api/wallets/start` | Start QR wallet onboarding |
| `GET` | `/api/wallets/pending?slot=w7` | Pending QR status |
| `POST` | `/api/wallets/<id>/rename` | Rename a wallet |
| `DELETE` | `/api/wallets/<id>` | Remove a wallet from the dashboard |
| `POST` | `/api/quote` | BAW sell quote with best-pool guard |
| `POST` | `/api/sell` | Queue a guarded sell |
| `POST` | `/api/send` | Queue a token send |
| `GET` | `/api/task/<id>` | Poll task progress |
| `GET` | `/api/qr?text=0x…` | Receive QR (exact address only) |
| `GET` | `/api/health` | Service health |

Mutating requests require `X-Requested-With: agentic` and are JSON.

---

## Data and state

Runtime data is intentionally outside Git:

```text
data/
├── wallets.json          # wallet configuration (contains addresses)
├── wallets.json.tmp
├── keepalive.json
├── history/              # cached transaction history per wallet
├── audit.jsonl           # local trade audit trail
├── balance-cache.json    # short-lived balance snapshot
└── market-cache.json     # short-lived price snapshot
```

`data/` and `.env` are ignored by Git. Never commit real wallet addresses, cookies, session files, or BAW credentials.

BAW session directories live under `$HOME/aw/wN` and are **not** part of this repository.

---

## Operational notes

- The UI polls `/api/portfolio` every 3 seconds and refreshes prices on a short cache.
- Transaction history is pre-cached in `data/history/`; opening the History view should be immediate.
- Keepalive runs hourly, executes `baw wallet status` per wallet, and does not place orders.
- Sells use BAW `market-order swap`; sends use `wallet send`.
- Token addresses sent to BAW are lowercased where required.
- The server keeps its last balance/market snapshot in `data/` so a restart can paint immediately before the first background refresh.

---

## Disclaimer

This software is provided for self-hosted, personal use. Crypto assets and on-chain transactions are irreversible. Prices, liquidity, taxes, quotes, and balances can be delayed or wrong. Always verify the recipient address, network, token contract, amount, slippage, and tax in Binance and on-chain before confirming. The authors are not responsible for financial loss.

Not affiliated with or endorsed by Binance.

---

## License

MIT — see [LICENSE](LICENSE).

**Sole contributor:** Ssw7777
