# RedCircle Target Product Catalog Integration

## Status

- Repository: `jmarrujo91-lab/Good-Buy`
- Working branch: `feature/redcircle-target-catalog`
- Phase: design and safe integration preparation
- Environment: sandbox/test only; no production deployment
- API credentials: not committed; configure a rotated RedCircle key in the hosting provider's secret manager after verifying the exact RedCircle authentication contract.

## Goals

Extend Good-Buy's retail product-comparison experience with Target product data supplied by RedCircle. Preserve the existing application, ownership, deployment configuration, and shopping-list behavior.

## Data-source boundary

RedCircle is a Target product-data source. A Target listing, BuyBox offer, or store-stock response does not prove that a product is a free sample, that the user may resell it, or that inventory is available for this application's own fulfillment. Keep source observations distinct from merchant-owned catalog and inventory records.

## Initial data model

- Product identity: source, TCIN, GTIN/UPC, DPCI (when supplied), canonical source URL.
- Product content: title, brand, description, image references, category, specifications.
- Offer snapshot: seller, price, currency, sponsored flag, maximum order quantity, protection plans, return policy, shipping details, observed-at timestamp.
- Store-stock observation: ZIP code, store identifier/name (when returned), stock status/quantity (only when returned), observed-at timestamp.
- Provenance: request type, source response timestamp, retrieval timestamp, freshness/expiry policy.

Do not fabricate missing prices, quantities, seller details, shipping charges, or store availability. Preserve raw source fields only where permitted by the RedCircle terms and data-retention rules.

## API integration requirements

1. Verify the current RedCircle API documentation for base URL, request types, required parameters, authentication, rate limits, credit costs, response schemas, and permitted caching/redistribution.
2. Implement a server-side adapter only after those details are verified. Never call RedCircle directly from browser code.
3. Read the API key from a server-side secret named `REDCIRCLE_API_KEY`; do not place it in source files, frontend bundles, logs, error messages, URLs in telemetry, or commits.
4. RedCircle's documented authentication method must be reviewed carefully before implementation. If the key is transmitted as a query parameter, redact query strings in HTTP logs, traces, proxy logs, and error reporting.
5. Add timeouts, bounded retries, rate limiting, request validation, credit-aware caching, and safe error handling.
6. Start with a small test lookup set. Do not enable bulk imports until credit consumption and data-use permissions are confirmed.
7. Validate `skip_gtin_cache` behavior and its additional credit cost before exposing it in the UI.

## Suggested application endpoints

These are proposed internal routes, not verified RedCircle routes:

- `GET /api/target/products/search`
- `GET /api/target/products/:tcin`
- `GET /api/target/products/:tcin/offers`
- `GET /api/target/products/:tcin/store-stock?zipcode=...`

All routes must authenticate/authorize the caller and validate inputs. Internal routes must not expose RedCircle credentials.

## UI behavior

- Label results as Target source data and show when each observation was retrieved.
- Distinguish product detail, seller offer, shipping/returns, and store-stock observations.
- Mark unknown or stale fields as unknown/stale rather than implying availability.
- Keep product comparison separate from purchasing, checkout, and fulfillment.
- Do not create Shopify products or write to any connected commerce platform as part of this integration without separate approval.

## Security and release gates

- Rotate the API key previously pasted into chat; use only the replacement in a secure secret manager.
- Keep test and production credentials separate.
- No live payments, order placement, reward issuance, transfers, or payouts.
- No production deploy, domain/DNS change, or public release without separate explicit approval.
- Add tests for authentication, authorization, schema validation, secret redaction, stale data, missing fields, and credit/error handling.
- Review the existing repository architecture and tests before modifying application code.
