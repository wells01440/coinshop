# Pamela's System — the confidence contract (v2)

Sources: dictations 2026-07-29 (two sessions) + photographed flips and the
sticker sheet. This is the domain spec. coinshop MIRRORS this system; it
never replaces it. Her flows must never gain steps.

## 0. The prime directive: the story is the product

"People want to know the story." When research surfaces a hook — *late
Roman military issue, era of the sack of Rome, AD 410: a Legionnaire held
this* — it goes on the card even when identification doesn't need it.
The significance note is not decoration; it is the sales narrative and
the on-air script. Field: `coin.significance TEXT`. The Whatnot crib
sheet displays it big.

## 0.5 The business model: attribution arbitrage

Most flips in the wild are labeled from the obverse only — a guess plus
a date range ("Constantine family"). The reverse almost always says
more: which Victory, which campaign; even Victory's walking direction
can flag a posthumous issue struck by sons in the emperor's honor. And
privy/officina marks pin the mint. Pamela's edge: buy the mislabeled
cheap, re-attribute from the reverse, capture the delta ($3 -> $450,
one-of-ten known). At shows she only buys *labeled* coins when hunting
something specific — no time to attribute live.

System consequences:
- Record the SELLER'S claimed attribution at intake, separate from the
  confirmed one. Migration 002 adds `coin.seller_attribution TEXT`.
  claimed-vs-confirmed + cost-vs-price = the value-created report and
  the ready-made listing narrative ("sold as X; it is actually Y").
- The reverse is never second-class: writeup renderer and coin page
  give REV full weight (legend, translation, description, direction
  details); truncating the reverse is a bug.
- Future queue idea (park it): an "arbitrage watchlist" — coins whose
  seller attribution and confirmed attribution disagree get a badge;
  that list IS her best-stories inventory.

## 1. The card format (writeup generator spec)

Verified against real cards (Diva Helena, Theodosius I, Septimius
Severus). Fixed field order:

 1. Issue line — "RI - DIVA HELENA" / emperor / provincial-civic id
 2. Reign years — "Reign: 379–395 AD" (or commemorative range)
 3. Denomination/type — AE4, AR denarius
 4. Mint (+ officina: "Antioch mint, 1st workshop")
 5. Struck date if known — "(coin struck 388–392 AD)"
 6. Weight + diameter — "1.52 g, 10 mm"; condition flag here when
    physical ("Fragmented/broken")
 7. OBV: legend + description — "DN THEODO-SIVS PF AVG, right facing…"
 8. REV: legend + description + translation — "SALVS REIPVBLICAE
    ('The health/salvation of the Republic') winged Victory walking l…"
 9. Symbols / privy marks (chi-rho etc.) — before references
10. REF: — "RIC VIII Antioch 42", "RIC IV 32; RSC 690b"
11. Story/significance — '"Public Peace"', '"Victoria Augusti" design
    directly celebrates his early civil war victories over Pescennius
    Niger.'

**Golden test**: the Diva Helena card, rendered from structured fields,
must be byte-for-byte plausible as her handwriting's content. Seed these
three photographed coins as test fixtures.

Migration 002: `coin.struck TEXT`, `coin.officina TEXT`,
`coin.special INTEGER NOT NULL DEFAULT 0`, `coin.significance TEXT`;
`date_from/to` = reign; `writeup` becomes a render (manual override only).
Legend translations live in `coin_part.type_desc` alongside description.

## 2. The sticker system (physical layer, derived digitally)

Purpose per Pamela: **magnitude at a glance, across the room** — the
tier matters more than the exact number ("that's a $5 coin / that's a
$50 coin", and nobody gets taken by a know-it-all). Digital derivation
therefore surfaces BOTH: tier (low / mid / high / wicked-high) and the
exact color.

Ladder (dot = f(price)); observed in the wild: Theodosius AE4 = red
($15), Diva Helena = yellow ($35), Severus denarius fragment = green
(**confirm $40 lime vs $45 green** — photo reads lime-ish):

black ≤$2 · [$2.50/$5/$7.50 — colors UNCONFIRMED, purple is one] ·
pink $10 · red $15 · salmon $20 · brown $25 · tan $30 · yellow $35 ·
lime $40 · green $45 · forest $50 · aqua $55 · light blue $60 ·
blue $65 · dark blue $70 · navy $75 · royal $80 · gray = hand-written
(>$100/odd values)

Semantics (both are TEAM signals, legible to anyone at the booth):
- **Star = silver.** Exists because sellers on shows routinely don't
  know their own silver. With a star, Hannah or anyone can work the
  booth and never be wrong about metal on-air. Derivable: material=AR.
- **Heart (back of flip) = STOP AND LOOK, wherever it appears.** Not
  only "rare" — "might need to go off to be graded." A $1000 coin still
  just gets a heart; price is handled by gray/hand-written. Maps to
  `special=1`; the significance note is the explanation on scan.

Open questions for Pamela: (a) colors for $2.50/$5/$7.50; (b) ladder
past royal $80 — anything before gray?; (c) lime-vs-green on the
Severus.

## 3. Library and integration strategy (time-to-market)

Only custom code: the state machine (done), writeup renderer, coin page,
labels. Everything else is glue to tools she already trusts.

| Need | Use | Why |
|------|-----|-----|
| QR on labels | segno | pure Python, zero deps |
| Code128 strip | python-barcode | done problem |
| Label render/print | Pillow @203 dpi -> CUPS lp | Rollo mounts as a normal printer |
| Camera tether | python-gphoto2 | D3300 supported |
| Moderns/cross-refs | numistalib (Andy's) | caching + rate limits solved |
| RIC imperial | OCRE Numishare APIs | canonical URIs per RIC type, JSON reconciliation w/ ruler+mint+denomination |
| RPC provincial | RPC Online | her provincial/civic world |
| RSC etc. | citation text in `reference` | no lookup needed |
| Wildwinds | link-out button | no API; habit unchanged |
| CoinSnap | CSV import | suggestions, never authority |
| Gemini | "Copy for Gemini" button | packages card into a prompt for HER Gemini; zero API work |
| Whatnot | Seller API bulk export / CSV | reconcile by C-number in listing title |

## 4. Port instructions (Claude Code, runner-3)

- Merge coinflow-bootstrap into this repo; rename package -> coinshop.
- uv-only; `uv add fastapi uvicorn click`; update CLAUDE.md to record
  the approved architecture (FastAPI owns SQLite/WAL, htmx, click CLI).
- ruff ALL + mypy strict: rewrite dynamic SQL in cli.py measure(),
  fix DTZ-naive datetimes, tighten types. Fix, don't ignore.
- Keep CI green on 3.11 and 3.12.
- Build order: migration 002 -> writeup renderer (golden test = Diva
  Helena card) -> sticker/star/heart derivations on coin page ->
  labels.py -> show pack/unpack.
