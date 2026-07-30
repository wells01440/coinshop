# coinflow — System Design v0.2

Changes from v0.1: multi-user (Andy at the bench, Pamela on research and
Whatnot, a possible third person listing), phone-first access, coin-show
manifests, and external data sources (Numista API, CoinSnap, Nomisma,
Whatnot Seller API). One architectural consequence and it's a big one.

---

## 0. The architectural change: hub, not file

v0.1 had every client opening the SQLite file directly. Two writers on two
devices kills that. v0.2: **one FastAPI service owns the SQLite file** (WAL
mode, single writer process — SQLite is perfectly happy like this at any
scale you'll ever hit). Everything else is an HTTP client:

```
                    ┌──────────────────────────────┐
                    │  homelab (opti-1, Ansible)   │
   Pi bench kiosk ──►  FastAPI  ──  coinflow.db    ◄── CLI (Mac/anywhere)
   Pamela's phone ──►  (Tailscale-only, MagicDNS)  ◄── coinx (API client)
   show phone     ──►                              ◄── sync jobs (Numista,
                    └──────────────────────────────┘      Whatnot)
```

Tailscale is the entire security model: no public exposure, phones join the
tailnet, `https://coin.<tailnet>.ts.net`. This is one more Ansible role in
the infra repo you already run. Nothing new to buy.

## 1. Labels: QR, not barcode

The flip label carries a QR encoding the coin URL
(`https://coin.<tailnet>.ts.net/c/C-0147`) plus the human-readable C-number
and a Code128 strip for the bench/show scanners. Pamela's entire onboarding
is: point phone camera at flip → coin page opens. No app, no login friction
beyond the tailnet.

## 2. Roles and actors

No permissions system — this is a household, not a bank. But every mutation
records **who**: `event.actor`. Three named actors to start
(`andy`, `pamela`, `lister`), selected once per device/session. The audit
answers "who confirmed this attribution" and "who marked this sold," which
is SPECTRUM's accountability requirement done the cheap way.

## 3. The coin page (Pamela's surface)

One mobile-first page per coin, everything on it:

- Header: C-number, stage badge, **current location** (tray/box/show)
- Photo strip: selected first, then evidence log filterable by setup
  (the specular privy-mark hunt frames are right there with their notes)
- Physicals: weight, dia, thickness, die axis, material, color
- Attribution block: current value + **confirm field** — she enters/edits
  the RIC/RPC/Sear citation, hits confirm → writes `reference` row +
  `attribution_confirmed` event with her name → stage button unlocks
- References: catalog cites, Nomisma URIs, Numista link, comps with prices
- Lot/provenance: source chain, cost basis
- Action row (stage-aware): promote / hold / sell (price field) / move
- Notes: append-only, actor-stamped

Same page serves as the **on-air crib sheet** during a Whatnot stream: she
picks up the flip, scans, reads the writeup aloud, taps Sell with the
hammer price when it goes. Sold disposition happens in real time, not in a
post-show spreadsheet session.

## 4. External data sources

**Numista** — real REST API (v3): catalogue search, type detail, coin
identification by image, pricing, collection management; free tier with a
daily request quota. Integration: `coinflow numista attach <coin> <type-id>`
pulls the type record (issuer, dates, physical specs, obv/rev/edge
descriptions, cross-reference numbers) into `reference` rows and prefills
empty coin fields — never overwrites a human-entered value. Ancients
coverage on Numista is thinner than moderns; treat as one source, not truth.

**CoinSnap** — no public API; it's a phone-app identifier. Treat its output
as a *suggestion source*: a text field on the coin page ("CoinSnap says:")
that lands in notes/attribution-candidate, pending Pamela's confirmation.
Zero integration code; it's just structured skepticism.

**Nomisma/NUDS** — the field semantics (already baked into the schema) plus
`reference.kind='uri'` rows holding nomisma.org / OCRE / RPC-online URIs.
Export layer emits NUDS XML per coin for interop.

**Whatnot** — two channels:
1. *Seller API*: GraphQL with bulk export operations — you POST a mutation,
   poll until COMPLETED, download JSONL of orders. This is the reconciler.
2. *CSV fallback*: Seller Hub exports a per-livestream report and weekly
   orders reports (order ID, product name, subtotal, fees, refunds).

The reconcile job matches Whatnot orders against coins marked sold during
the stream (by C-number in the product name — **put the C-number in every
Whatnot listing title or description**, that's the join key) and attaches
order ID, net proceeds, and fees to the `sale` row. Refund rows in the
report flip a sale to `refunded` and bounce the coin back to `final`.

## 5. Coin show workflow (manifests)

A manifest is a named movement batch — SPECTRUM location & movement control
with a reconciliation gate on both ends:

```
coinflow show pack TNA-2026
  → scan each flip going in the box
  → each scan: location = 'show:TNA-2026', manifest row added
  → close: prints packing list (count + total ask price)

at the show (phone, cellular + Tailscale):
  → scan flip → coin page → Sell → price → sold, venue='show:TNA-2026'

coinflow show unpack TNA-2026
  → scan every returning flip → location back to home tray
  → diff MUST be empty: every coin is either scanned-back or sold.
    Anything else prints in red. That's the "know they are in hand"
    guarantee, both directions.
```

Offline fallback if the venue is a cell dead zone: the pack step exports the
manifest to the phone as a local list; sales queue on-device and post when
connectivity returns. (Build this only if it actually happens.)

## 6. Schema delta (v0.1 → v0.2)

```sql
CREATE TABLE actor (
    id    TEXT PRIMARY KEY,               -- 'andy','pamela','lister'
    name  TEXT NOT NULL
);

ALTER TABLE event ADD COLUMN actor TEXT REFERENCES actor(id);

CREATE TABLE sale (
    id          INTEGER PRIMARY KEY,
    coin_id     TEXT NOT NULL REFERENCES coin(id),
    venue       TEXT NOT NULL,            -- 'whatnot','show:TNA-2026','private'
    price_cents INTEGER NOT NULL,         -- hammer/agreed
    fees_cents  INTEGER,                  -- filled by reconcile
    net_cents   INTEGER,                  -- filled by reconcile
    order_ref   TEXT,                     -- Whatnot order ID / receipt no.
    sold_on     TEXT NOT NULL,
    sold_by     TEXT REFERENCES actor(id),
    status      TEXT NOT NULL DEFAULT 'pending'
                CHECK (status IN ('pending','reconciled','refunded'))
);

CREATE TABLE manifest (
    id        TEXT PRIMARY KEY,           -- 'show:TNA-2026'
    opened_at TEXT NOT NULL,
    closed_at TEXT,
    status    TEXT NOT NULL DEFAULT 'open'
              CHECK (status IN ('open','packed','reconciled'))
);

CREATE TABLE manifest_coin (
    manifest_id TEXT NOT NULL REFERENCES manifest(id),
    coin_id     TEXT NOT NULL REFERENCES coin(id),
    packed_at   TEXT NOT NULL,
    resolved    TEXT CHECK (resolved IN ('returned','sold')),
    PRIMARY KEY (manifest_id, coin_id)
);
```

Stage delta: `listed → sold` now requires a `sale` row; reconcile jobs may
later update it but never gate it (a show sale is done when the cash is).

## 7. Revised repo layout

```
coinflow/
├── schema/           # 001_init.sql, 002_multiuser.sql
├── src/coinflow/
│   ├── db.py  models.py  stages.py  naming.py
│   ├── api/          # FastAPI: routes/coins, photos, sales, manifests
│   ├── web/          # server-rendered mobile pages (htmx, no SPA)
│   ├── cli.py        # thin client over the API
│   ├── capture.py    # gphoto2 (runs on Pi, posts to API)
│   ├── labels.py     # QR + Code128 + C-number → Rollo
│   └── sync/
│       ├── numista.py   # type attach + prefill
│       └── whatnot.py   # order reconcile (API or CSV)
├── station/kiosk.py
├── ansible/          # role: coinflow service on opti-1
└── tests/
```

Web layer is server-rendered + htmx deliberately: one codebase, no JS build,
phone pages are just HTML, and the Pi kiosk can render the same pages in a
browser in kiosk mode — the 7" screen and Pamela's phone run the *same UI*.
DRY at the interface level.

## 8. Revised build order

1. Schema + FastAPI skeleton + Ansible role (service exists on tailnet)
2. Labels (QR+128) + `show pack/unpack` CLI — immediate value, backfills
   existing trays, proves the scan loop
3. Coin page (read) → Pamela can look up any coin from her phone
4. Coin page (write): confirm attribution, stage moves, Sell button
5. Pi capture station posting to the API
6. Numista attach, then Whatnot reconcile
7. NUDS export, offline show mode — only if ever actually needed
