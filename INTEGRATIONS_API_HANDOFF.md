# Exsqueektion API — developer handoff

**Production base URL:** `https://squeek.win/api/integrations/v1`

Read-only, server-to-server API. All four endpoints use **GET**, return JSON, and require this header:

```http
Authorization: Bearer <INTEGRATIONS_API_KEY>
```

Get the **current production `INTEGRATIONS_API_KEY`** from the operator through a private channel. No wallet login, cookies, `Origin`, request body, Discord token, or webhook signing secret is needed. Keep the key in your backend environment; these endpoints do not enable browser CORS.

## Endpoints

| Method and path | Accepted query parameters | Returns |
| --- | --- | --- |
| `GET /results` | `mode`, `limit`, `cursor` | Completed runs, newest first, including extractions and losses. |
| `GET /leaderboard` | `mode`, `limit` | Best extracted run per wallet, ranked by score for the current season. |
| `GET /players/{address}` | `mode` | One wallet's lifetime totals and current-season best result. |
| `GET /status` | None | Game health, wheel state, active-run counts and mode availability. |

Query rules:

- `mode`: exactly `practice` or `live`; defaults to `practice`. Always specify it when you care about the distinction. There is no combined-mode query; request each mode separately.
- `limit`: integer **1–50**, default **25**. Supported only on results and leaderboard.
- `cursor`: opaque token from the previous results response. Omit it for the first page; send it unchanged, URL-encoded, with the **same mode** for subsequent pages.
- `address`: `0x` followed by 40 hexadecimal digits. Accepted addresses are normalized to checksum form in responses.
- Unknown/repeated query parameters, unsupported modes and invalid values return `400`.

## Copy-paste requests

Set `INTEGRATIONS_API_KEY` in your shell/backend environment first. These commands contain no credentials of their own.

```bash
# Latest Practice results; use mode=live for Live.
curl --fail-with-body \
  -H "Authorization: Bearer $INTEGRATIONS_API_KEY" \
  'https://squeek.win/api/integrations/v1/results?mode=practice&limit=10'

# Top 10 Live scores.
curl --fail-with-body \
  -H "Authorization: Bearer $INTEGRATIONS_API_KEY" \
  'https://squeek.win/api/integrations/v1/leaderboard?mode=live&limit=10'

# Replace the example wallet with the requested player's address.
curl --fail-with-body \
  -H "Authorization: Bearer $INTEGRATIONS_API_KEY" \
  'https://squeek.win/api/integrations/v1/players/0x19E7E376E7C213B7E7e7e46cc70A5dD086DAff2A?mode=practice'

# Current game/wheel status.
curl --fail-with-body \
  -H "Authorization: Bearer $INTEGRATIONS_API_KEY" \
  'https://squeek.win/api/integrations/v1/status'
```

## Response contract

These TypeScript shapes describe the complete successful response bodies. All timestamps are **Unix milliseconds**; scores, counts, versions and ticks are JSON numbers. `activeTicks / 60` is active duration in seconds. Example values below are illustrative, not a promise of current production data.

```ts
type Mode = "practice" | "live";
type Season = { scoringVersion: number; fromGenerator: number };

type GameResult = {
  runId: string;                // UUID; use for deduplication
  address: string;              // Public wallet address
  mode: Mode;
  outcome: "extracted" | "lost";
  startedAt: number;
  completedAt: number;
  activeTicks: number;
  scoringVersion: number;
  generatorVersion: number;
  score: {
    total: number;
    haul: number;
    killPoints: number;
    expeditionKillPoints: number;
    renewableKillPoints: number;
  };
};

type ResultsResponse = {
  schemaVersion: 1;
  mode: Mode;
  asOf: number;
  results: GameResult[];
  nextCursor: string | null;
};

type LeaderboardResponse = {
  schemaVersion: 1;
  mode: Mode;
  sort: "score";
  season: Season;
  asOf: number;
  entries: {
    rank: number;              // 1-based
    completedRuns: number;     // All completed runs in this mode, across seasons
    bestResult: GameResult;
  }[];
};

type PlayerResponse = {
  schemaVersion: 1;
  address: string;
  mode: Mode;
  season: Season;
  asOf: number;
  lifetime: {
    completedRuns: number;
    extractedRuns: number;
    lostRuns: number;
    bankedHaul: number;
    bankedKillPoints: number;
  };
  bestResult: GameResult | null;
};

type StatusResponse = {
  schemaVersion: 1;
  serverTime: number;
  game: {
    state: "starting" | "available" | "stale" | "unavailable";
    observedAt: number | null;
    ageMs: number | null;
    staleAfterMs: 5000;
    activeRuns: { practice: number; live: number };
    connectedPlayers: number;
    faultedRuns: number;
  };
  wheel: {
    state: "running" | "stopped" | "unknown";
    availability: "available" | "stale" | "unavailable";
    observedAt: number | null;
    ageMs: number | null;
    expiresAt: number | null;
    qualifying: boolean;
  };
  modes: { practiceAvailable: boolean; liveAvailable: boolean };
};
```

Example `GET /results?mode=practice&limit=1` response (`200 OK`):

```json
{
  "schemaVersion": 1,
  "mode": "practice",
  "asOf": 1793275201000,
  "results": [
    {
      "runId": "92182d45-438c-4691-9d1b-5600d4048891",
      "address": "0x19E7E376E7C213B7E7e7e46cc70A5dD086DAff2A",
      "mode": "practice",
      "outcome": "extracted",
      "startedAt": 1793275190000,
      "completedAt": 1793275200000,
      "activeTicks": 600,
      "scoringVersion": 3,
      "generatorVersion": 20,
      "score": {
        "total": 437,
        "haul": 400,
        "killPoints": 37,
        "expeditionKillPoints": 30,
        "renewableKillPoints": 7
      }
    }
  ],
  "nextCursor": null
}
```

## Pagination and interpretation

- Results include completed runs across all seasons, ordered by `completedAt` descending, then `runId` descending. They never include unfinished runs.
- Continue fetching results until `nextCursor` is `null`. An empty mode returns `results: []` and `nextCursor: null` with `200`, not an error. For ongoing polling, start at the first page and deduplicate by `runId`; cursors navigate older results and are not a snapshot of concurrent writes.
- Leaderboard has no pagination and returns at most 50 entries. It includes only extracted runs matching `season.scoringVersion` and `generatorVersion >= season.fromGenerator`. Highest score wins; ties favor earliest completion, then run ID and wallet address. Read the returned season rather than hardcoding it.
- Player lifetime totals span all seasons **within the requested mode**. `bestResult` follows the current leaderboard season. An unknown wallet returns zero totals and `bestResult: null`; a known wallet can also have no qualifying best result.
- Scores are banked values: `total = haul + killPoints`; `killPoints = expeditionKillPoints + renewableKillPoints`. Losses bank zero. Practice and Live are separate in both outer responses and each result's `mode`.
- Leaderboard/player aggregates can be cached server-side for up to **5 seconds**; `asOf` is their query timestamp. Responses have `Cache-Control: no-store`.
- `practiceAvailable` requires a healthy game host. `liveAvailable` additionally requires a fresh, qualified running wheel. Availability is not an admission reservation. An unknown wheel is not equivalent to a stopped wheel; `qualifying` means a new running observation is awaiting confirmation.
- The read API returns result data, not image URLs. The separately configured outbound Discord webhook attaches a PNG and labels its embed/image PRACTICE or LIVE.

Next-page request, after storing a non-null `nextCursor` in `CURSOR`:

```bash
curl --fail-with-body --get \
  -H "Authorization: Bearer $INTEGRATIONS_API_KEY" \
  --data-urlencode 'mode=practice' \
  --data-urlencode 'limit=10' \
  --data-urlencode "cursor=$CURSOR" \
  'https://squeek.win/api/integrations/v1/results'
```

## Errors and rate limits

Normal API errors have this JSON shape:

```json
{ "schemaVersion": 1, "error": { "code": "unauthorized" } }
```

| HTTP status | `error.code` | Handling |
| --- | --- | --- |
| `400` | `invalid_query` | Fix query/address/cursor; use the cursor's original mode. |
| `401` | `unauthorized` | Supply exactly one valid Bearer header with the current key. |
| `404` | `not_found` | Check route and API version. |
| `405` | `method_not_allowed` | Use GET. |
| `429` | `rate_limited` | Wait for the `Retry-After` header, in seconds. |
| `503` | `unavailable` | Retry with backoff; shutdown or infrastructure responses may have a different body. |

The authenticated budget is **120 requests/minute shared across all consumers of the key per server process**. Poll modestly (for example, every 10 seconds), reuse fetched data between bot commands, and honor `Retry-After`. The API key authorizes these read endpoints only; it does not authorize private `/api/account` routes or game mutations.

Production verification on **2026-10-29** covered all four endpoints, both modes, JSON Schema validation, pagination, score arithmetic and error responses. At that check there were no completed Live runs, so Live results/leaderboard correctly returned empty arrays; populated Live behavior was also covered by integration tests.
