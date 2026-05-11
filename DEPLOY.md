# Valdex — Deploy & Connect Guide

Two self-contained HTML files plus a few static assets. No build step, no Node, no framework. Drop into any static host and it runs.

---

## 1. What's in the box

```
github-deploy/
├── index.html              ← Landing page (1.75 MB, fully self-contained)
├── exchange/
│   └── index.html          ← Trading app  (1.47 MB, fully self-contained)
├── robots.txt              ← Search-engine rules
├── sitemap.xml             ← For Google/Bing
└── DEPLOY.md               ← This file
```

Both HTML files have **every asset inlined** — fonts, images, React, the chart engine, all CSS. They work fully offline if you just double-click them on your laptop. The only thing you have to do is put them behind a real domain so wallets and APIs trust the origin.

---

## 2. Deploy to a static host (pick one)

### Option A — GitHub Pages (free, easiest)
1. Push `github-deploy/` to a public repo, e.g. `valdex/valdex.me`.
2. Settings → Pages → Source: `Deploy from a branch` → `main` → `/ (root)`.
3. Add a `CNAME` file with `valdex.me` if you have the domain.
4. Done in ~2 min. URL: `https://<user>.github.io/<repo>/` or your custom domain.

### Option B — Cloudflare Pages (free, CDN, recommended for production)
1. `npx wrangler pages deploy github-deploy --project-name valdex`
2. Or connect the repo in Cloudflare dashboard → Pages → Create.
3. Custom domain in dashboard. SSL auto.

### Option C — Vercel / Netlify
1. `vercel github-deploy` or `netlify deploy --dir github-deploy --prod`.
2. Both work with zero config — they detect the static HTML.

### Option D — Your own server (nginx)
```nginx
server {
  listen 443 ssl http2;
  server_name valdex.me;
  root /var/www/valdex;
  index index.html;

  # SPA-style: serve index.html for unknown paths
  location / { try_files $uri $uri/ /index.html; }
  location /exchange/ { try_files $uri $uri/ /exchange/index.html; }

  # Long cache for the giant bundled HTML (it never changes between deploys)
  location ~ \.html$ {
    add_header Cache-Control "public, max-age=300, must-revalidate";
  }

  # Security headers
  add_header X-Frame-Options "SAMEORIGIN";
  add_header Referrer-Policy "strict-origin-when-cross-origin";
  add_header Permissions-Policy "geolocation=(), camera=(), microphone=()";
}
```

---

## 3. DNS

| Record | Host | Value                              | Purpose                |
|--------|------|------------------------------------|------------------------|
| A      | @    | `<your host's IP>`                 | Apex domain            |
| CNAME  | www  | `valdex.me.`                       | www redirect           |
| TXT    | @    | `v=spf1 -all`                      | No mail from this domain (you don't send) |
| CAA    | @    | `0 issue "letsencrypt.org"`        | Lock SSL issuer        |

---

## 4. APIs that need to be wired up

Right now the exchange ships with **simulated data** — candles, order book, fills, latency are all generated client-side so the UI is reviewable end-to-end. Before mainnet you need to point it at real endpoints. There are five integrations, listed in the order you should tackle them.

### 4.1 — Wallet connection (RainbowKit / WalletConnect)
**What it does:** lets users connect MetaMask, Trust Wallet, Rabby, Coinbase Wallet, hardware wallets, and mobile wallets via QR code.

**Sign up:** https://cloud.walletconnect.com → create a project → copy the **Project ID**.

**Install:** add the WalletConnect provider script to `exchange/index.html` (or migrate to RainbowKit if you switch to a build step):
```html
<script src="https://unpkg.com/@walletconnect/web3-provider@1.8.0/dist/umd/index.min.js"></script>
```

**Wire it up:** in the `WalletDrawer` component, replace the simulated `connectWallet()` with:
```js
const provider = new WalletConnectProvider({
  projectId: 'YOUR_WALLETCONNECT_PROJECT_ID',
  chains: [56],              // BNB Chain mainnet
  rpc: { 56: 'https://bsc-dataseed.binance.org/' }
});
await provider.enable();
const address = provider.accounts[0];
```

**Cost:** free up to 10k connections/month, then $0.003 per connection.

---

### 4.2 — BNB Chain RPC (the on-chain settlement layer)
**What it does:** reads margin balances, submits signed orders to the matching contract, fetches confirmation receipts.

Pick one provider (in order of recommendation for a perpetuals exchange):

| Provider | Free tier | URL | Notes |
|---|---|---|---|
| **QuickNode** | 10M reqs/mo | https://www.quicknode.com | Best p99 latency for BSC, has dedicated WebSocket. **Recommended.** |
| **Ankr** | 30 rps | https://www.ankr.com/rpc/bsc | Cheapest paid tier, good free limits. |
| **NodeReal MegaNode** | 3M cu/day | https://nodereal.io | BNB Chain–native, run by Binance ecosystem. |
| **Public RPC** | none, rate-limited | `https://bsc-dataseed.binance.org/` | Fine for dev, do **not** ship to prod. |

**Wire it up:** in `exchange/index.html`, search for `const RPC_URL` (or add it near the top of the App component) and replace with your endpoint:
```js
const RPC_URL = 'https://bsc-mainnet.example.quiknode.pro/YOUR_KEY/';
const WS_URL  = 'wss://bsc-mainnet.example.quiknode.pro/YOUR_KEY/';
```

You'll want ethers.js v6 added to the bundle for contract calls:
```html
<script src="https://cdn.jsdelivr.net/npm/ethers@6.13.0/dist/ethers.umd.min.js"></script>
```

---

### 4.3 — Price feed (oracle)
**What it does:** the live mark price shown in the ticker, used for unrealized PnL and liquidation distance.

Two paths:

**A) Chainlink BNB Chain price feeds** — free, on-chain, the canonical choice. Read the BTC/USD aggregator at `0x264990fbd0A4796A3E3d8E37C4d5F87a3aCa5Ebf`:
```js
const aggregatorAbi = ['function latestAnswer() view returns (int256)'];
const btc = new ethers.Contract(BTC_USD_AGG, aggregatorAbi, provider);
const price = Number(await btc.latestAnswer()) / 1e8;
```
Full list of feeds: https://docs.chain.link/data-feeds/price-feeds/addresses?network=bnb-chain

**B) Binance public REST/WebSocket** — off-chain, sub-100ms, free. Use this for the **chart** and the **tape**, and Chainlink for **settlement math** so traders can't game it.
```js
const ws = new WebSocket('wss://stream.binance.com:9443/ws/btcusdt@kline_1m/btcusdt@trade/btcusdt@depth20@100ms');
```

---

### 4.4 — Historical candles (for the chart)
**What it does:** populates the chart with the last N bars before the WebSocket takes over for live updates.

```js
const res = await fetch(`https://api.binance.com/api/v3/klines?symbol=BTCUSDT&interval=1m&limit=500`);
const candles = (await res.json()).map(([t, o, h, l, c, v]) => ({
  time: t / 1000, open: +o, high: +h, low: +l, close: +c, volume: +v
}));
chartSeries.setData(candles);
```

Wire this into the `Chart` component's `useEffect` — search for the `simulateCandles` function and swap it out.

No API key needed for Binance public market data (1200 reqs/min, more than enough).

---

### 4.5 — Your own backend (the matching engine + receipts)
**What it does:** signs fills, returns verifiable receipts, accepts withdrawal requests, serves the leaderboard.

The exchange UI expects these endpoints (you implement them). Suggested shape:

| Method | Path | Returns |
|---|---|---|
| `GET`  | `/api/v1/markets` | list of `{symbol, mark, funding, openInterest}` |
| `GET`  | `/api/v1/account?address=0x…` | `{equity, free, used, positions[], orders[]}` |
| `POST` | `/api/v1/order` | `{orderId, status}` — body: `{side, pair, size, leverage, type, price?, tpsl?}` |
| `POST` | `/api/v1/cancel` | `{ok}` — body: `{orderId}` |
| `GET`  | `/api/v1/fills?address=0x…` | `[{txHash, blockHash, signature, sequence, latency, …}]` for the Receipts rail |
| `WS`   | `/ws` | streams `order_update`, `fill`, `funding`, `liquidation` |

**Auth:** sign a nonce with the user's wallet (EIP-712), pass the signature in the `Authorization: Sig <sig>` header. Don't use API keys for retail — wallet sig is the whole point of being non-custodial.

Once your backend is up, search the exchange source for `const API_BASE` and set:
```js
const API_BASE = 'https://api.valdex.me';
const WS_BASE  = 'wss://api.valdex.me/ws';
```

---

## 5. Optional but recommended

| Service | Why | Cost |
|---|---|---|
| **Cloudflare** in front of everything | DDoS, bot mitigation, free SSL, free CDN | Free |
| **Plausible** or **Umami** analytics (not Google) | Privacy-respecting, GDPR-friendly | $9/mo or self-host |
| **Sentry** error tracking | Catches runtime JS errors in production | Free up to 5k events/mo |
| **Statuspage** (Atlassian) or **Instatus** | Public uptime page for `/status` link in footer | $20–29/mo |
| **Fleek** / **IPFS** mirror | Censorship-resistant copy of the UI | Free |

---

## 6. Sanity checklist before going live

- [ ] DNS resolves to your host and SSL cert is green
- [ ] Both pages load in <2s on a cold cache (Lighthouse ≥ 90)
- [ ] WalletConnect ProjectID set, MetaMask connects to BNB Chain (chainId 56)
- [ ] RPC endpoint responds to `eth_blockNumber` in < 200ms
- [ ] Chart shows real BTC/USDT data, not the simulated candles
- [ ] `robots.txt` allows `/`, disallows `/exchange/` and `/calendar/`
- [ ] `sitemap.xml` only lists public pages
- [ ] Open Graph card renders correctly on https://www.opengraph.xyz
- [ ] No console errors in DevTools on either page
- [ ] Footer `System Status`, `Documentation`, `API Reference` links point somewhere real (or are gated behind a "Soon" badge like the social links)

---

## 7. Updating

The HTML files are bundles built from `ui_kits/website/index.html` (source). To update:

1. Edit the source.
2. Re-bundle: any HTML bundling tool that inlines `<script src>`, `<link rel="stylesheet">`, and `<img src>` will work. Common choices:
   - `npx inline-source --compress false ui_kits/website/index.html > github-deploy/index.html`
   - `npx html-bundle ui_kits/website/index.html`
   - Or just hand-paste assets if you're touching one thing.
3. Commit the new `index.html`. CI/CD picks it up.

---

That's the whole deploy. Five services to wire up, one DNS record per subdomain, and you're live.
