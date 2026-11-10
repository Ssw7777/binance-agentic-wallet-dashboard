# Security

## Reporting

Do not open a public issue containing wallet addresses, QR payloads, session
files, cookies, API keys, or logs with personal data.

Report security issues privately to the repository owner.

## Deployment checklist

- [ ] Bind the dashboard to `127.0.0.1` only; never expose port 8100.
- [ ] Put an authenticated reverse proxy in front of it.
- [ ] Keep BAW sessions under a dedicated OS user with restrictive permissions.
- [ ] Do not commit `data/`, `.env`, BAW session directories, or exported logs.
- [ ] Verify the full recipient address before every transfer.
- [ ] Treat QR codes as address-only; re-check the full address shown under the QR.
- [ ] Rotate credentials immediately if a session, cookie, or token is exposed.

## What this project does not do

- It does not import private keys.
- It does not store seed phrases.
- It does not bypass Binance risk controls or phone confirmation.
- It does not guarantee price, quote, tax, or liquidity accuracy.

On-chain transactions are irreversible.
