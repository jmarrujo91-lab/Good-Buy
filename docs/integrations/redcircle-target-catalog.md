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


## Cross-platform shipping, receiving, and delivery integration

The catalog must support inbound supplier parcels as well as outbound customer shipments. Keep platform responsibilities separate:

- Shopify is the connected commerce/order/fulfillment source of truth for approved store workflows. The connected store is Via del oro. Do not assume order, inventory, fulfillment, or carrier records are already synced.
- Stripe is the payment-state source only. Use the verified sandbox context for testing; never use the separate live context for this project. Payment events must be signature-verified, idempotent, and reconciled with the internal order before fulfillment is eligible. Stripe does not provide parcel tracking.
- Notion is the durable project register for decisions, integration status, carrier setup references, receiving/dispatch checklists, and exceptions. Do not copy API keys, webhook secrets, full payment details, or unnecessary customer shipping PII into Notion.

### Inbound receiving (supplier to business)

Model purchase-order/reference, supplier, carrier, tracking number, shipped time, estimated arrival (explicitly an estimate), carrier scans, carrier-delivered time, physical received time, receiving location, received-by, package condition, SKU, expected quantity, counted quantity, discrepancy/damage/shortage, inspection result, and inventory-posting state.

A carrier-delivered scan is not proof that contents were physically received or accepted. Increase available inventory only after a recorded count and acceptance. Quarantine damaged, missing, disputed, or uninspected quantities. Never invent tracking numbers, dates, counts, or receiving addresses.

### Outbound fulfillment (business to customer)

Model order/fulfillment reference, package ID, carrier/service, tracking number and URL, label-created time, packed/handed-to-carrier time, carrier scan events, estimated delivery, delivery confirmation, failed attempt/exception, return-to-sender, return authorization, and refund/replacement state.

- Support multiple packages/split shipments per order.
- Keep pickup/local delivery separate from carrier shipping and record actual handoff/collection.
- A label-created or fulfilled status is not proof of delivery. Use authoritative carrier-confirmed events.
- Refunds, replacements, returns, and restocking are separately authorized and reconciled events.

### Shared shipment event model

Use an append-only, provider-neutral event record: provider/source, event ID, direction (inbound/outbound), order or purchase-order reference, package reference, raw provider status, normalized status, event timestamp, observed timestamp, optional source location, and deduplication key. Preserve raw status and source provenance. Authenticate webhooks, verify signatures where supported, process idempotently, retry safely, and place failures into an operator review queue.

Suggested outbound normalized statuses: label_created, carrier_received, in_transit, out_for_delivery, delivered; exception statuses: delayed, failed_attempt, damaged, lost, returned_to_sender, return_in_transit, returned. Inbound receipt lifecycle: expected, shipped, carrier_delivered, physically_received, inspected, accepted_or_quarantined, inventory_posted.

Do not conflate RedCircle Target store-stock observations with the merchant's own stock. Keep Shopify orders/fulfillments, Stripe payment events, carrier scans, RedCircle product observations, and physical receiving counts as separate source records linked by stable references.

### Current connection and release boundaries

- Shopify connection read-verified: Via del oro (via-del-oro.myshopify.com), USD, United States. No store writes were made.
- Stripe account discovery found a Via del oro sandbox and a separate live account. Use sandbox only; no live changes or transactions.
- Notion project register has been updated with the cross-platform shipping/receiving requirements.
- Carrier accounts, API credentials, webhook endpoints, receiving address, and shipping services are not yet verified/configured.
- No labels purchased, shipments booked, packages dispatched, deliveries marked, inventory received, or production deployment performed.
- Carrier API integrations and any Shopify/Stripe writes require their own connection verification and explicit approval. Keep all development/test work private and non-production until separate release approval.


## Carrier priority and access scope — 2026-11-04

User-selected carrier priority:
- Primary carriers: USPS and FedEx.
- Additional supported carriers: UPS and DHL.
- Scope: both inbound supplier receiving and outbound customer fulfillment, with tracking events, delivery exceptions, returns, and shipment status reconciliation.

Implement one provider adapter per carrier behind the shared shipment-event interface. Normalize carrier-specific tracking states without discarding raw source events. Support multiple packages per order and supplier shipment. Keep pickup/local delivery separate from carrier shipping.

**Access status:** this records the requested carrier scope; it does not establish that any USPS, FedEx, UPS, or DHL account/API credentials, OAuth grants, webhooks, billing profiles, negotiated rates, or label permissions have been connected or verified. Discover and request only the minimum permissions required. Do not store carrier secrets in GitHub or Notion. Do not buy labels, tender packages, schedule pickups, or alter shipping charges without separate approval.

**Implementation order:** USPS + FedEx first; UPS + DHL adapter compatibility and test coverage next. Use sandbox/test credentials or carrier test endpoints where available. If a carrier offers no sandbox for a specific feature, keep that feature read-only or mocked until separately approved. Production activation requires account ownership/access verification, webhook authentication, rate/service validation, address validation, test evidence, and explicit release approval.


## Coordinated launch sequence and notification reliability — 2026-11-04

The user wants the projects to operate concurrently. Recommended approach: run implementation/readiness workstreams in parallel, but release in gated order so unresolved dependencies in one project do not block the others.

1. **Via del oro — first production candidate.** Existing Shopify store and Stripe sandbox context were previously read-verified. Complete live-account ownership/permission review, product and pricing validation, tax/shipping/refund policy, checkout tests, fulfillment controls, payout eligibility/destination review, and operational monitoring before any production activation. No live changes are implied or authorized by this sequence.
2. **Good-Buy / Target catalog — second release candidate.** The repository currently contains planning documentation on feature/redcircle-target-catalog; application code, rotated RedCircle key, carrier adapters, end-to-end tests, and production deployment remain incomplete/unverified. Build and test while Via del oro readiness proceeds.
3. **OMG Sweeps — third release candidate.** Keep separate from the other projects. Prior context records unresolved repository visibility and Render service verification, plus provider written eligibility/compliance requirements. Complete repo/deploy access, sweepstakes/legal/provider eligibility, funds-flow, Shopify fulfillment and Stripe sandbox tests before production review.

All three workstreams may proceed in parallel; production launch order is gated as above and may be revised only if verified readiness evidence changes.

### Notification and event-delivery reliability

The user requests notifications enabled and no stuck routing. This is a requirement, not a verified setting. Current checks found no general ChatGPT notification toggle exposed through the available settings tools. Device-level Android notification permission, battery/background restrictions, notification channels, and delivery status cannot be verified from this project connection.

Implement and test notification delivery with:
- Explicit event-to-recipient/channel routing and a visible delivery state (queued, sent, delivered/acknowledged where provider supports it, failed, retrying, dead-letter).
- Durable queue/outbox, idempotency keys, bounded exponential retries, dead-letter queue, replay tooling, and alerting for age-of-oldest-message and failure rate.
- Correlation IDs across Shopify, Stripe, carrier, app, and Notion events; preserve source event IDs and timestamps.
- Alert on missing webhook acknowledgements, invalid signatures, repeated failures, route misconfiguration, expired credentials, and webhook backlog.
- Separate operational alerts (payment/fulfillment failures, shipment exceptions, payout holds) from informational updates.
- Do not claim push/email/SMS delivery merely because an event was queued or a webhook was accepted. Do not send test or customer notifications without an approved destination and test plan.

Before production, run an end-to-end notification test for each approved channel and carrier integration, verify retries and dead-letter recovery, and document the results. No notification preference or device setting was changed by this update.
