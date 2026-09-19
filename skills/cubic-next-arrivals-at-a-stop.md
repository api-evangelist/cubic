---
name: next-arrivals-at-a-stop
description: Get upcoming arrival and departure predictions for a transit stop from Cubic's Umo IQ Public Feed, including the service messages a rider needs to see alongside them.
api: Umo IQ Public Feed API
provider: Cubic Corporation
operations:
  - getPublicJsonFeed
commands:
  - agencyList
  - routeList
  - routeConfig
  - predictions
  - predictionsForMultiStops
  - messages
generated: '2026-09-19'
method: generated
source: openapi/cubic-umo-iq-public-feed-openapi.yml, https://retro.umoiq.com/xmlFeedDocs/NextBusXMLFeed.pdf
---

# Next arrivals at a stop

Base URL: `https://retro.umoiq.com/service`. Anonymous — no credential of any kind.

## 1. Resolve agency → route → stop

```
GET /publicJSONFeed?command=agencyList
GET /publicJSONFeed?command=routeList&a=<agency>
GET /publicJSONFeed?command=routeConfig&a=<agency>&r=<route>&terse
```

Add `terse` unless you are drawing a map: it drops the polyline data and roughly halves the
payload. `routeConfig` without `r` returns every route, capped at 100 per request.

From the stop list you get two different identifiers, and the difference matters:

- **`tag`** uniquely identifies one physical stop. Use it with a route tag.
- **`stopId`** is a numeric id intended for phone and SMS input. It is **not unique** — one
  `stopId` can cover several bays at a terminal — but it lets you ask for every route serving
  that stop at once. Not every agency assigns them.

Stop tags may carry `_IB`, `_OB` or `_ar` suffixes at larger agencies, and the provider warns
they may not line up with the agency's GTFS ids.

## 2. Ask for predictions

Every route serving a stop:

```
GET /publicJSONFeed?command=predictions&a=<agency>&stopId=<stopId>
```

One route at one stop:

```
GET /publicJSONFeed?command=predictions&a=<agency>&r=<route>&s=<stop tag>
```

Several stops in one call (max 150 per route), repeating `stops` as `<route>|<stop>`:

```
GET /publicJSONFeed?command=predictionsForMultiStops&a=<agency>&stops=501|472&stops=501|7813
```

Add `useShortTitles=true` for small screens.

## 3. Read the predictions correctly

Predictions are grouped by `direction`, because routes turn back at different places and a rider
needs to know whether the vehicle is going where they are going. At most **5 predictions per
direction** are returned.

- Display **`minutes`**, rounding seconds down. Use `seconds` to know when the minute will tick.
- `epochTime` (milliseconds) is for showing a clock time.
- `isDeparture: "true"` means the vehicle is laid over at the stop and this is a departure time,
  not an arrival.
- `affectedByLayover: "true"` means the prediction leans on a scheduled departure and is less
  accurate. `isScheduleBased: "true"` means no GPS is involved at all. `delayed: "true"` means
  the vehicle is moving slower than expected. **Each of these appears only when true** — absent
  means false.
- If there are no predictions there will be no `direction`; the stop's direction name arrives in
  `dirTitleBecauseNoPredictions` so your UI can still label what the rider asked for.

## 4. Show the service messages

Prediction responses may carry `message` entries, and you should surface them — they carry
detours and service changes. For the full picture:

```
GET /publicJSONFeed?command=messages&a=<agency>&r=<route>
```

Route tag `all` holds agency-wide messages. Only currently active messages are returned. A
message may carry `textSecondaryLanguage` (a second language), `phonemeText` (for text-to-speech)
and `smsText`.

## Rules and error handling

- Anonymous, read-only. There is nothing to write, so there is nothing to undo.
- At most one poll every 10 seconds; 2 MB per 20 seconds per IP; send `Accept-Encoding: gzip`.
- **HTTP 200 on everything.** Check for an `Error` member, and branch on `shouldRetry`:
  `"true"` → wait 10 seconds and retry; `"false"` → fix the request.
- JSON returns one element as an object and several as an array. All values are strings.

See `conventions/cubic-conventions.yml`, `errors/cubic-problem-types.yml` and
`rate-limits/cubic-rate-limits.yml`.
