---
name: track-a-bus
description: Follow live vehicle positions on one transit route using Cubic's Umo IQ Public Feed, polling incrementally without exceeding the provider's published limits.
api: Umo IQ Public Feed API
provider: Cubic Corporation
operations:
  - getPublicJsonFeed
commands:
  - agencyList
  - routeList
  - vehicleLocations
  - vehicleLocation
generated: '2026-09-19'
method: generated
source: openapi/cubic-umo-iq-public-feed-openapi.yml, https://retro.umoiq.com/xmlFeedDocs/NextBusXMLFeed.pdf
---

# Track vehicles on a route

Base URL: `https://retro.umoiq.com/service`. No authentication — do not send an API key or
Authorization header; there is none to send.

## 1. Find the agency tag

```
GET /publicJSONFeed?command=agencyList
```

Pick the `tag` of your agency from `agency[]`. **Do not hardcode `sf-muni` from the provider's
documentation** — it has left the feed and now returns an invalid-agency error. Seventeen
agencies were live on 2026-09-19, including `ttc`, `omnitrans`, `stl` and `ccrta`.

## 2. Find the route tag

```
GET /publicJSONFeed?command=routeList&a=<agency tag>
```

## 3. Poll vehicle locations incrementally

First call with `t=0`, which returns the last 15 minutes:

```
GET /publicJSONFeed?command=vehicleLocations&a=<agency>&r=<route>&t=0
```

The response carries `lastTime.time`. **Pass that value back as `t` on the next call** and you
receive only reports newer than it. This is the provider's documented way to stay inside the
data cap; polling with `t=0` every time will burn the budget on data you already have.

Each `vehicle` gives `id`, `lat`, `lon`, `heading` (degrees, negative when unavailable),
`speedKmHr`, `dirTag`, and `secsSinceReport` — **check `secsSinceReport`**, because a GPS report
can be several minutes old, and `predictable`, which tells you whether the vehicle is currently
being used for predictions.

To follow a single vehicle instead of a route, use `command=vehicleLocation&a=<agency>&v=<id>`.

## Rules you must follow

- **Poll at most once every 10 seconds.** The provider states this twice, in the licence and in
  the vehicleLocations section.
- **Stay under 2 MB per 20 seconds per IP.** Send `Accept-Encoding: gzip, deflate`; the provider
  documents a 50-85% reduction.
- `vehicleLocations` will not return more than a 5-minute span.

## Error handling

Every response is **HTTP 200**, including failures. Do not branch on the status code. Check for
an `Error` member instead:

- `shouldRetry: "true"` — the agency server was initialising. Wait 10 seconds and retry the same
  URL.
- `shouldRetry: "false"` — your request is wrong. Fix the parameters; retrying will not help.

On the JSON feed, `Error` has been observed as an **array** of identical objects as well as a
single object. Handle both. The same applies to `vehicle`: one vehicle is an object, several are
an array.

All values are **strings**, including `lat`, `lon`, `heading` and `predictable`. Coerce them.

See `errors/cubic-problem-types.yml` and `conventions/cubic-conventions.yml`.
