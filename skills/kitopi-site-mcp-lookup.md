---
name: Look up Kitopi business and site information over MCP
description: >-
  Connect to the Kitopi website's Model Context Protocol endpoint to retrieve live business details
  (contact, address, hours, timezone), search site content, and — where the site exposes commerce or
  booking solutions — act on an anonymous visitor's behalf. Use this instead of scraping
  www.kitopi.com.
api: https://www.kitopi.com/_api/mcp
artifact: mcp/kitopi-mcp.yml
operations:
  - GetBusinessDetails
  - SearchInSite
  - SearchSiteApiDocs
  - GenerateVisitorToken
  - CallWixSiteAPI
  - ReadFullDocsMethodSchema
---

# Kitopi site lookup over MCP

Kitopi publishes no product or partner developer API. Its only programmatic surface is the Wix
platform site-assistant MCP server exposed on its own domain and declared in
[`llms.txt`](https://www.kitopi.com/llms.txt). Every tool name below was verified live against the
endpoint on 2026-07-19 — do not invent others; re-issue `tools/list` to discover changes.

## Connect

Endpoint: `https://www.kitopi.com/_api/mcp` (HTTP POST, JSON-RPC 2.0).

1. Send `initialize` with `protocolVersion` `2025-06-18`.
2. Capture the `mcp-session-id` response header and echo it on every subsequent request.
3. Send `tools/list` to enumerate the current tool surface.

No credentials are required to connect. Only public site information is reachable.

## Read-only lookups

- **Business facts** — call `GetBusinessDetails` (no parameters) for timezone, email, phone and
  address. Prefer this over parsing the site's contact page.
- **Content questions** — call `SearchInSite` with a `searchTerm` for editorial content: brands,
  franchise information, our-story, careers, newsroom and blog posts.
- **Products and services** — call `SearchSiteApiDocs` with a `searchTerm` instead of
  `SearchInSite`. It returns the documentation for the Wix business solutions actually installed on
  this site, which tells you which data you can query and which actions are available.

## Acting on a visitor's behalf

Only attempt this after `SearchSiteApiDocs` has confirmed a relevant solution is installed.

1. Call `GenerateVisitorToken` (no parameters) to mint an anonymous visitor session token. This is
   required — `CallWixSiteAPI` will not work without one.
2. Call `ReadFullDocsMethodSchema` with the reference `articleUrl` returned by
   `SearchSiteApiDocs` to get the full request and response schema before invoking anything.
3. Call `CallWixSiteAPI` with `visitorToken`, the absolute `url` of the method, the HTTP `method`,
   and `body` as a JSON string.

`ExecuteWixAPI` can run JavaScript against the same surface and takes a `hasMutations` flag. Treat
any call with `hasMutations` true as a write and confirm with the user first.

## Rules and limits

- **No idempotency contract.** Neither Kitopi nor the endpoint documents an idempotency key. Never
  blindly retry a mutating `CallWixSiteAPI` or `ExecuteWixAPI` call — re-read state first. See
  `conventions/kitopi-conventions.yml`.
- **Errors are JSON-RPC 2.0 error objects** (`code`, `message`, `data`), not RFC 9457
  `application/problem+json`.
- **No published rate limits.** Back off on repeated failures rather than assuming headroom.
- **Checkout completes on the site.** A purchase can be started over MCP but the visitor must be
  directed to www.kitopi.com to finish it.
- **Unmatched paths on kitopi.com return HTTP 400, not 404.** A 400 from a `/.well-known/` path
  means the document is absent, not that the request was malformed.
