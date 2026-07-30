# coinflow — System Design v0.1

A single-operator collections management system for ancient coins, modeled on
museum practice (SPECTRUM 5.1 procedures, NUDS field semantics), implemented as
a Python package over one SQLite file, with a Raspberry Pi bench station for
capture and a CLI for everything else.

Design values: append-only evidence, boring state machine, one primary key per
coin from the moment it enters the door, zero import steps, DRY (the SQLite
file is the single source of truth — coinx, the Pi, and the CLI all read it).

---

## 1. The two-number system (stolen from SPECTRUM)

Museums separate the **entry number** (assigned at the door, before you know
what you have) from the **accession number** (assigned when an object is
formally taken into the collection). This maps exactly onto the ancients
problem: you buy a *lot* — an auction group, a bag of uncleaned — and only
later individuate coins.

- **LOT-YYYY-NNN** — the purchase/deposit unit. Carries sourcing, provenance
  chain, total cost, seller, date. Accounting lives here.
- **C-NNNN** — the coin. Created when a coin is individuated from a lot.
  Cost basis is allocated from the lot (even split by default, manual
  override for the one good coin in the bag). The barcode label on the flip
  is this ID, printed at individuation, before the first photo.

Rule zero: **no photograph without a C-number.**

## 2. Stage model (state machine)

```
intake ──► processing ──► advanced ──► final ──► listed ──► sold
   │                                     │          │
   └────────► returned/rejected          └──► hold  └──► withdrawn
```

Stages are *gates with evidence requirements*, not to-do items. A transition
is legal only when its predicate passes:

| Transition            | Gate predicate                                             |
|-----------------------|------------------------------------------------------------|
| intake → processing   | lot linked, cost allocated, label printed                  |
| processing → advanced | weight_mg, dia_mm set; ≥1 photo per side (any setup)       |
| advanced → final      | attribution set (or explicitly `unattributed`), refs logged |
| final → listed        | ≥1 `selected` photo per side, price set, writeup present   |
| listed → sold         | sale record (venue, price, date, buyer ref)                |

Time between stages is unbounded by design. The DB holds the state; you never
have to remember where a tray was.

## 3. Schema (SQLite, DDL)

NUDS-informed: obverse/reverse/edge are first-class parts; die axis, mint,
authority, denomination, and reference IDs are explicit fields; provenance and
appraisal are structured. SPECTRUM-informed: locations are logged as
movements; every state change is an audit event.

```sql
PRAGMA foreign_keys = ON;

CREATE TABLE lot (
    id          TEXT PRIMARY KEY,          -- LOT-2026-001
    source      TEXT NOT NULL,             -- seller/auction house
    source_ref  TEXT,                      -- auction/lot/eBay item no.
    acquired_on TEXT NOT NULL,             -- ISO date
    cost_cents  INTEGER NOT NULL,
    fees_cents  INTEGER NOT NULL DEFAULT 0,
    provenance  TEXT,                      -- prior chain as known at entry
    notes       TEXT
);

CREATE TABLE coin (
    id            TEXT PRIMARY KEY,        -- C-0147
    lot_id        TEXT NOT NULL REFERENCES lot(id),
    stage         TEXT NOT NULL DEFAULT 'intake'
                  CHECK (stage IN ('intake','processing','advanced',
                                   'final','listed','sold','hold',
                                   'returned','withdrawn')),
    cost_cents    INTEGER,                 -- allocated basis
    weight_mg     INTEGER,                 -- 3-decimal g balance → mg int
    dia_mm        REAL,
    thick_mm      REAL,                    -- nominal
    die_axis      INTEGER,                 -- clock value 1–12 (NUDS)
    material      TEXT,                    -- AE/AR/AV/billon/…
    color         TEXT,                    -- patina description
    denomination  TEXT,
    authority     TEXT,                    -- issuer (e.g. Gallienus)
    mint          TEXT,
    date_from     INTEGER,                 -- year, negative = BC
    date_to       INTEGER,
    attribution   TEXT,                    -- 'unattributed' is a valid value
    grade         TEXT,
    location      TEXT,                    -- current tray/box/flip cabinet
    price_cents   INTEGER,
    writeup       TEXT,
    notes         TEXT,
    created_at    TEXT NOT NULL DEFAULT (datetime('now'))
);

-- NUDS treats obv/rev/edge as separately describable parts.
CREATE TABLE coin_part (
    coin_id     TEXT NOT NULL REFERENCES coin(id),
    part        TEXT NOT NULL CHECK (part IN ('obv','rev','edge')),
    legend      TEXT,
    type_desc   TEXT,                      -- iconography
    marks       TEXT,                      -- privy/countermark/mintmark/graffito
    PRIMARY KEY (coin_id, part)
);

-- Catalog references: RIC, RPC, Sear, SNG, die studies, comps.
CREATE TABLE reference (
    id        INTEGER PRIMARY KEY,
    coin_id   TEXT NOT NULL REFERENCES coin(id),
    kind      TEXT NOT NULL CHECK (kind IN ('catalog','die_study','comp','uri')),
    citation  TEXT NOT NULL,               -- 'RPC X 94332' / nomisma URI / sale URL
    price_cents INTEGER,                   -- for comps
    noted_on  TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Append-only photo log. Culling is a flag, never a delete.
CREATE TABLE photo (
    id         INTEGER PRIMARY KEY,
    coin_id    TEXT NOT NULL REFERENCES coin(id),
    session_ts TEXT NOT NULL,
    part       TEXT NOT NULL CHECK (part IN ('obv','rev','edge','oblique','detail')),
    setup      TEXT NOT NULL,              -- 'xpol','axial','cpl_rake','raking','specular'
    seq        INTEGER NOT NULL,
    path       TEXT NOT NULL,              -- NEF on disk; name = {coin}_{ts}_{setup}_{part}_{seq}
    status     TEXT NOT NULL DEFAULT 'evidence'
               CHECK (status IN ('evidence','selected','technical_reject')),
    note       TEXT                        -- 'privy mark visible at 4h'
);

-- SPECTRUM: location & movement control + full audit trail, append-only.
CREATE TABLE event (
    id        INTEGER PRIMARY KEY,
    coin_id   TEXT REFERENCES coin(id),
    lot_id    TEXT REFERENCES lot(id),
    at        TEXT NOT NULL DEFAULT (datetime('now')),
    kind      TEXT NOT NULL,               -- 'stage_change','moved','measured',
                                           -- 'photographed','labeled','priced','listed','sold'
    detail    TEXT                         -- JSON payload
);

CREATE INDEX idx_coin_stage ON coin(stage);
CREATE INDEX idx_photo_coin ON photo(coin_id, part, status);
CREATE INDEX idx_event_coin ON event(coin_id, at);
```

Money is integer cents. Weight is integer milligrams (your 3-decimal balance
maps losslessly). No floats where a ledger lives.

## 4. Repo layout

```
coinflow/
├── pyproject.toml            # ruff + mypy configured; zero-warning policy
├── schema/
│   └── 001_init.sql
├── src/coinflow/
│   ├── __init__.py
│   ├── db.py                 # connection, migrations, WAL mode
│   ├── models.py             # dataclasses mirroring tables
│   ├── stages.py             # transition table + gate predicates (pure funcs)
│   ├── cli.py                # click: intake, split, next, move, measure,
│   │                         #   attribute, price, select, promote, export
│   ├── capture.py            # gphoto2 wrapper: locked config, capture+download
│   ├── naming.py             # one function owns the filename grammar
│   ├── labels.py             # Rollo ZPL/EPL barcode label
│   └── export/
│       ├── nuds.py           # NUDS XML per coin (interop, future-proofing)
│       └── listing.py        # listing text from writeup + selected photos
├── station/
│   └── kiosk.py              # Pi 7" touch loop: scan → status → shoot → flag
└── tests/
```

## 5. Bench station flow (Pi + 7" + scanner + D3300)

Idle screen = work queue: `SELECT id, authority, stage FROM coin WHERE stage=?`.

1. **Scan** flip barcode → coin card: stage, measurements present/missing,
   photo count by setup, last session date, open notes.
2. **Shoot**: pick setup (xpol / axial / cpl_rake / specular hunt), each
   shutter fires via gphoto2, NEF lands pre-named, row inserted, thumbnail on
   screen. One key: `t` = technical reject (focus/shake only). Nothing else
   is judged at the bench.
3. **Measure**: type weight + dia when in `processing`; gates update live.
4. **Flag**: one-line note dictated to the photo row ('possible privy 4h').
5. Next scan switches context. Session ends whenever; state is in the DB.

Photo *selection* happens weeks later on the Mac, in coinx or
`coinflow select`, reading the same SQLite file. The specular evidence hunt
never fights the archival workflow because `setup` and `status` keep them in
separate lanes of the same append-only log.

## 6. What maps to what (SPECTRUM ↔ your stages)

| Your stage          | SPECTRUM procedure(s)                          |
|---------------------|-------------------------------------------------|
| Intake              | Object entry; Acquisition & accessioning        |
| Processing          | Condition checking & technical assessment       |
| Advanced processing | Cataloguing                                     |
| Final processing    | Valuation; (photo selection = Reproduction)     |
| Listing/sale        | Object exit; Deaccessioning & disposal          |
| (implicit, now explicit) | Location & movement control; Audit        |

The two you didn't have — location/movement and the audit event log — are the
ones museums consider non-negotiable, and they're precisely the "where is that
coin / what did I already do" failures in the current mess.

## 7. Build order

1. `schema/001_init.sql` + `db.py` + `models.py` (an evening)
2. `cli.py` intake/split/measure/next — usable immediately, pre-camera
3. `labels.py` Rollo output — flips get barcodes, backfill existing trays
4. `capture.py` + `station/kiosk.py` on the Pi
5. `export/nuds.py` last — interop layer, nothing depends on it
