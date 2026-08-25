---
name: clear
description: >
  Pay and fetch a public URL through Clear. An agent pays HTTP 402 in one
  call (USDC on Base via x402 / PayAI) and gets markdown extracted from the
  public HTML. No signup, no API key. Use when the task needs the actual
  public page, not a challenge or blank-JS shell. Do not use for credential
  stuffing, 2FA bypass, bulk account creation, or private URLs.
---

# Clear

Live host: https://clear-agent-fetch.fly.dev

A page an AI agent can buy in one HTTP call. V1 is one public URL per paid call. $0.005 USDC on Base. Charge only on a successful extract.

v1 extracts the public HTML with Readability + Turndown. JS-only shells fail and are not billed. This is not Playwright / hosted Chrome.

Agents: use `x402-fetch` / `@x402/fetch` and `PAYMENT-SIGNATURE`. `/try` is a human MetaMask page, not the agent path.

## Before you pay

1. Read `/llms.txt` and `/openapi.yaml` if you can reach the site.
2. Confirm the URL is **public** — not behind a login, paywall, or cookie the agent does not have a right to.
3. If the job is credential stuffing, 2FA bypass, bulk account creation, or reading someone else's private page: **stop**.
4. Do not ask for a method to bypass anti-bot systems.

## Fetch a URL

Price: **$0.005 USD per successful URL**. Failed extracts and policy refusals are not billed.

1. `POST https://clear-agent-fetch.fly.dev/v1/fetch` with `content-type: application/json` and body `{"url":"https://example.com"}`. No payment header.
2. Expect **402**. Read `accepts` (body and/or `PAYMENT-REQUIRED` header, x402 v2).
3. Pay the only rail: USDC on Base — `scheme: exact`, `network: eip155:8453`, amount `"5000"`, asset `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913`, payTo `0x2382EF343ba76a6D56efC8C04920d7df2F98A54D`, `extra.name` `"USD Coin"` version `"2"` (on-chain EIP-712 name, not `"USDC"`). Use an x402 v2 client (`x402-fetch` / `@x402/fetch`) with USDC on Base in a wallet that can sign x402 (EIP-3009 TransferWithAuthorization). Facilitator: PayAI (`https://facilitator.payai.network`). Echo the 402 `extensions` (bazaar) into the payment payload.
4. Retry the **same** `POST` with `PAYMENT-SIGNATURE`.
5. On **200**, use `markdown` (and `title`, `final_url`). `charged` is true. If `success` is false, it was not billed.

```bash
curl -sS -D - -X POST https://clear-agent-fetch.fly.dev/v1/fetch \
  -H "content-type: application/json" \
  -d '{"url":"https://example.com"}'
```


## x402-fetch (copy-paste)

Sign, then fetch, then settle only on success. Fail is free (`charged: false`).

Constants an agent must use (do not invent):

- Host: `https://clear-agent-fetch.fly.dev`
- `POST /v1/fetch` body `{"url":"https://example.com"}`
- amount `"5000"` (atomic USDC, $0.005)
- `extra.name` `"USD Coin"` version `"2"` (on-chain EIP-712 name, not `"USDC"`)
- payTo `0x2382EF343ba76a6D56efC8C04920d7df2F98A54D`
- asset `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913`
- network `eip155:8453` (Base)
- facilitator `https://facilitator.payai.network`

```js
import { wrapFetchWithPayment, x402Client } from "@x402/fetch";
import { ExactEvmScheme } from "@x402/evm/exact/client";
import { privateKeyToAccount } from "viem/accounts";

const signer = privateKeyToAccount(process.env.EVM_PRIVATE_KEY);
const client = new x402Client();
client.register("eip155:8453", new ExactEvmScheme(signer));
const fetchWithPay = wrapFetchWithPayment(fetch, client);

const res = await fetchWithPay("https://clear-agent-fetch.fly.dev/v1/fetch", {
  method: "POST",
  headers: { "content-type": "application/json" },
  body: JSON.stringify({ url: "https://example.com" }),
});
const page = await res.json();
// 200: page.markdown, page.charged === true (settled)
// 422/403: page.charged === false (not billed)
```

The wrapper reads the 402, signs EIP-3009 with `extra.name` `"USD Coin"`, retries with `PAYMENT-SIGNATURE`, and echoes bazaar `extensions`. `/try` is MetaMask for humans, not this path.

## Responses

- **200** — successful extract. `markdown` is from the public HTML. Payment settled.
- **402** — unpaid or verify failed. Sign, retry the same body.
- **400** — missing or non-http(s) `url`.
- **403** — policy refuse (robots.txt or non-public target). `charged` is false.
- **405** — GET. Use POST.
- **422** — extract failed (not a usable public page). `charged` is false.
- **429** — one in-flight fetch per wallet, or 5 unsettled failures in 60 minutes for that wallet.

V1 is one URL per call. No batch, no crawl, no screenshot.

## Rules

- Public pages only. Do not send authenticated or private URLs.
- Do not use Clear for credential stuffing, 2FA bypass, or bulk account creation.
- We respect robots.txt. Do not try to work around a refuse.
