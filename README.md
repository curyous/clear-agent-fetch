# Clear

A public URL as markdown, paid in one HTTP 402 call. **$0.005 USDC on Base.** Sign first, fetch, cash only if markdown comes back. Fail is free.

- Live: https://clear-agent-fetch.fly.dev
- Skill: [SKILL.md](./SKILL.md)
- Machine docs: https://clear-agent-fetch.fly.dev/llms.txt
- OpenAPI: https://clear-agent-fetch.fly.dev/openapi.yaml
- gold-402: https://24klabs.ai/listing/clear
- x402scan: https://www.x402scan.com/server/clear-agent-fetch.fly.dev

```bash
curl -sS -D - -X POST https://clear-agent-fetch.fly.dev/v1/fetch \
  -H "content-type: application/json" \
  -d '{"url":"https://example.com"}'
```

Expect **402**. Pay USDC on Base (`amount` `"5000"`, `extra.name` `"USD Coin"`). Retry with `PAYMENT-SIGNATURE`. Public pages only. Not Playwright.

```
npx skills add https://github.com/curyous/clear-agent-fetch
```
