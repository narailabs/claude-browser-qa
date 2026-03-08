# Performance Awareness Checks

Observational checks using Chrome MCP tools. Not a full profiling suite — catches obvious problems. Run with `--perf` or `--mode full`.

**Performance findings are advisory. They do NOT trigger auto-fix agents.**

## Contents
- [Slow Requests](#1-slow-network-requests)
- [Large Payloads](#2-large-response-payloads)
- [Excessive DOM](#3-excessive-dom-size)
- [Request Count](#4-too-many-network-requests)
- [Asset Caching](#5-uncompressed-or-uncached-assets)
- [Duplicate Requests](#6-duplicate-api-calls)
- [Render-Blocking](#7-render-blocking-resources)
- [Reporting](#reporting)

## 1. Slow Network Requests
- `read_network_requests` — review all requests during page load
- \>3 seconds → **warning**
- \>10 seconds → **minor bug**
- Report: URL, method, duration

## 2. Large Response Payloads
- Check response sizes from network data
- \>1 MB single response → **warning**
- \>5 MB single response → **minor bug**
- Pay attention to API/JSON responses — large JSON often means missing pagination or over-fetching

## 3. Excessive DOM Size
- `read_page` — assess page complexity qualitatively
- Heavily nested structures or hundreds of repeated rows without virtualization → flag
- This is qualitative (exact counts not available) — flag only obvious cases

## 4. Too Many Network Requests
- Count total requests per page load
- \>50 requests → **warning**
- \>100 requests → **minor bug**
- Note duplicate or redundant requests to the same endpoint

## 5. Uncompressed or Uncached Assets
- Look for large JS/CSS bundles in network data
- Note if same resources re-fetched on navigation (no caching)

## 6. Duplicate API Calls
- Check for identical requests (same URL, method) fired within the same page load
- Two+ identical API calls on a single page → **warning**
- Common cause: multiple components independently fetching the same data

## 7. Render-Blocking Resources
- Note large synchronous scripts loaded in `<head>` without `async` or `defer`
- Large blocking scripts delay page render → **observation**

## Reporting

Performance findings go in a **Performance Observations** section of the report:
- Observation description (e.g., "GET /api/tasks took 4.2 seconds")
- Screen and request/element
- Brief recommendation (e.g., "Consider pagination for this endpoint")
- Severity is always `minor` or `observation`
