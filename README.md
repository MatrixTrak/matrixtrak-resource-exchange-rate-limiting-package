# Exchange rate limiting & backoff strategy package

Production-focused companion repository for a MatrixTrak resource.

## What This Repository Is

This repository is the public distribution surface for the linked MatrixTrak resource.
It is designed for quick implementation support, community sharing, and stable versioned references.

## Canonical MatrixTrak Links

- Resource page (canonical): https://matrixtrak.com/resources/exchange-rate-limiting-package
- Primary blog posts:
  - https://matrixtrak.com/blog/trading-bot-429-after-deploy-stop-rate-limit-storms

## Resource Summary

YAML config templates for Binance, Kraken, Coinbase, Bybit + decision checklist + 429 logging schema. Know when you're being rate-limited before your bot crashes.

## Repository Contents

- `resources/` contains shipped files copied from MatrixTrak public ship assets when available
- `docs/post-mapping.md` maps this resource to related blog posts
- `docs/resource-files.md` lists included files and source mapping
- Included shipped files:
  - resources/manifest.json
  - resources/rate-limit-429-logging-schema.json
  - resources/rate-limit-config-template.yaml
  - resources/rate-limit-decision-checklist.md
  - resources/README.md

## Who This Is For

- Engineers handling production incidents and reliability gaps
- Teams implementing or validating practical safeguards
- Readers coming from community channels who need canonical references

## Included Mapping

Primary mapping (post frontmatter resources):
- trading-bot-429-after-deploy-stop-rate-limit-storms - Trading bot keeps getting 429s after deploy: stop rate limit storms

Secondary mapping (resource relatedPosts):
- exchange-api-bans-how-to-prevent - API key suddenly forbidden: why exchange APIs ban trading bots without warning

## Usage Notes

- Treat MatrixTrak pages as the canonical long-form guidance.
- Use this repo for practical implementation support and sharing.
- For updates, always check the canonical resource page first.

## Attribution
Use MatrixTrak canonical links above for the full context and updates.
