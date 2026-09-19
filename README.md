# jevcache

**The decision ledger for Jev-class models.** A Jev decision is (approximately) a pure
function of `(model, schema, state)` — so it's memoizable. jevcache caches those
decisions locally: same decisions, fewer bills, and deterministic replay in CI. It's
backend-agnostic — point it at any local decision endpoint, a hosted Jev provider, or
a mock — and it never proxies or resells inference (your key stays on your machine).

- **Local-first & offline.** An embedded, zero-dependency store (in-memory map +
  append-only log). A point lookup is a map access — instant.
- **Private by construction.** State is redacted and canonicalized *before* it's
  hashed, so the fingerprint is over decision-relevant content only; PII (emails,
  phones, long digit runs) and volatile fields (ids, timestamps) never enter the key.
  For sensitive schemas, a per-schema `salt` keeps outsiders from enumerating decided states.
- **Deterministic CI.** Publish fixtures, `replay` them, catch model drift.


> **This is the distribution repo** — the `jevcache` CLI binary and docs.
> Install with `curl -fsSL jevcache.sh/install | sh`, or download a binary from [Releases](https://github.com/hyperspaceai/jevcache/releases).

## Install

```bash
curl -fsSL jevcache.sh/install | sh
```

Installs a single static binary (~2–3 MB, no runtime) for macOS/Linux, arm64 or x64.
The installer verifies a SHA-256 checksum before installing. `recall()` checks the local
ledger only (never a backend); `decide()` recalls, then on a miss routes to your configured
backend (`JEVCACHE_BACKEND=local|jev|mock`) and caches the result.

The CLI is written in Rust; its fingerprint is byte-identical to this repo's TypeScript
core, so a Rust ledger, the TS internals, and the hosted commons all share the same keys.


## CLI

```bash
jevcache demo                                                # try it in one command (no setup)
jevcache decide  --schema support.route --state ticket.json   # decide (cache on miss)
jevcache recall  --schema support.route --state ticket.json   # local only; exit 3 on miss
jevcache serve   [--host 0.0.0.0] [--port 9000]              # local decide/recall HTTP API
jevcache publish                                              # bundle your cache(s) to share
jevcache add     https://…/route.jevcache.json                # merge someone else's cache
jevcache replay  --schema eval.triage   --cases cases.jsonl   # CI determinism / drift
jevcache schemas                                              # list schemas
jevcache stats                                                # hit rate, spend saved
```

Schemas load from `./jevcache.schemas.json` or `~/.jevcache/schemas/*.json`. State is a
JSON file (or `-` for stdin; a bare string is a valid state). `recall` exits **3** on a
miss so CI can branch on it.

## Where it runs

- **A long-running app** — run `jevcache serve` as a sidecar and call `http://localhost:9000`.
- **Serverless / edge** (Vercel, Cloudflare, Lambda) — no daemon in an ephemeral function, so
  host jevcache once (`docker run`, Fly, Railway) with `JEVCACHE_SERVE_TOKEN`, and your
  functions call it over HTTPS. See `Dockerfile`.
- **Scripts / CI** — call the binary directly.

```bash
POST /decide  { schema, state }   → { answers, cached }   # caches on a miss
POST /recall  { schema, state }   → { hit, answers }       # never calls a backend
```

## Config (env)

| Var | Default | Purpose |
|---|---|---|
| `JEVCACHE_BACKEND` | `local` | `local` \| `jev` \| `mock` |
| `TYPESAFE_API_KEY` | — | your key for the `jev` backend (`POST /v1/systemone`; never stored) |
| `JEVCACHE_LOCAL_URL` | `http://127.0.0.1:8080/v1/decide` | local decision endpoint (`{state,questions}`→`{answers}`) |
| `JEVCACHE_DIR` | `~/.jevcache` | ledger + schemas location |
| `JEVCACHE_HOST` / `JEVCACHE_PORT` | `127.0.0.1` / `9000` | `serve` bind address |
| `JEVCACHE_SERVE_TOKEN` | — | bearer token required by `serve` when set (for a networked instance) |

## Share a cache

Sharing needs no server. `jevcache publish` bundles the caches already in your ledger —
one file per schema, holding fingerprints and answers only, never raw state. Host the file
anywhere; anyone merges it with `jevcache add` and their `recall()` starts hitting your
decisions.

```bash
# you: bundle every cache you've built
jevcache publish
# → bundled support.route.v3@3  1,284 decisions → ~/.jevcache/shared/support.route.v3.jevcache.json

jevcache publish --gist                 # upload as a public gist (needs the `gh` CLI), prints a URL
jevcache publish --schema support.route  # just one schema

# them: one command, then recall() hits your decisions — no backend, no account, no key
jevcache add https://gist.github.com/you/…/raw
jevcache recall --schema support.route.v3 --state ticket.json   # HIT
```

`add` also registers the schema locally, so `recall`/`decide` resolve it immediately. A
bundle is plain JSON — inspect it before you add it.

## Hosted global index (beta)

A hosted index is live at `https://jevcache.sh/api` (the default). Push a cache and recall
from it over the network — only fingerprints and answers cross the wire, never raw state.

```bash
jevcache publish --remote                                   # push your cache to the index (free)
jevcache use --schema support.route --state ticket.json     # recall from the index
```

Reads are open and keyless. Writes mint a free key (cached at `~/.jevcache/apikey`) so
decisions are attributable and no one can overwrite another publisher's cache.

| Var | Default | Purpose |
|---|---|---|
| `JEVCACHE_API_URL` | `https://jevcache.sh/api` | commons base URL — point at a different instance to override |
| `JEVCACHE_API_KEY` | — | write key (auto-minted on first `publish --remote`) |

Next: a decentralized index over the Hyperspace P2P network, and per-recall pricing so
publishers can charge for a cache.

## License

jevcache is **free to use**. This repository distributes the CLI binary and docs; the source is maintained privately. Not affiliated with TypeSafe (Jev is their model).
