# coinshop — Requirements (2026-07-29)

Supersedes design-v0.1/v0.2 where they conflict; read with
pamela-system.md (domain spec). Repo: wells01440/coinshop (public).
Naming: the project, package, and repo are **coinshop**. R-numbers are
stable IDs for issues/PRs. MUST/SHOULD/MAY per RFC 2119.

## 1. Domain & data

- R1. One SQLite file is the single source of truth; WAL mode; exactly
  one writer process (the API service; CLI direct-write allowed only in
  single-user mode). MUST.
- R2. Two-number system: lots (LOT-YYYY-NNN) carry sourcing, provenance,
  accounting; coins (C-NNNN) are individuated from lots with allocated
  cost basis (even split default, manual override). MUST.
- R3. No photograph enters the system without a C-number; naming.py owns
  the filename grammar and refuses nonconforming names. MUST.
- R4. Photo log is append-only; culling is a status flag
  (evidence | selected | technical_reject); DELETE is forbidden. MUST.
- R5. Stage machine: intake → processing → advanced → final → listed →
  sold (+ hold/returned/withdrawn/refund paths). Transitions occur only
  via gates that report exactly what is missing. MUST.
- R6. Every mutation writes an actor-stamped event row (SPECTRUM audit).
  MUST.
- R7. Money = integer cents; weight = integer milligrams. MUST.
- R8. Obverse/reverse/edge are first-class parts (NUDS): legend,
  description/translation, marks each. The reverse is never truncated
  or summarized away. MUST.
- R9. Migration 002 adds: struck TEXT, officina TEXT, special INT
  DEFAULT 0, significance TEXT, seller_attribution TEXT; date_from/to
  are reign years. MUST.
- R10. Attribution delta (seller_attribution vs attribution) and ROI
  (price vs cost) are derivable reports, not stored fields. SHOULD.

## 2. Pamela invariants (adoption constraints)

- R11. Her first scan MUST save time: scan → coin card (photos,
  attribution, location, story) in ≤1 s on the tailnet, before the
  system ever asks her for input. MUST.
- R12. Her flows never gain steps; data she doesn't care about never
  blocks her path (gates are bench-side concerns). MUST.
- R13. Writeup is RENDERED from structured fields in her exact card
  order (pamela doc §1); golden test = the Diva Helena card. Manual
  override allowed, never required. MUST.
- R14. Sticker color, tier band (low/mid/high/wicked), star (=AR), and
  heart (=special, "stop and look") are DERIVED and displayed; never
  stored redundantly; never nagged about. MUST.
- R15. Significance/story renders prominently on the crib-sheet view;
  it is the sales narrative. MUST.
- R16. Confirm-attribution action records reference + actor +
  attribution_confirmed event and unlocks stage promotion. MUST.

## 3. Interfaces

- R17. FastAPI service on the homelab, reachable ONLY over Tailscale;
  no public exposure. MUST.
- R18. /c/{id} mobile coin page: server-rendered HTML + htmx; same
  pages serve Pamela's phone, the shop iPhone 11, and the Pi 7" kiosk
  (browser kiosk mode). One UI codebase. MUST.
- R19. CLI (click): init, intake, split, measure, queue, promote, sell,
  show pack/unpack, label. In multi-user mode CLI becomes a thin client
  of the API. MUST (pack/unpack + label are next up).
- R20. Flip label: QR encoding the tailnet coin URL + human-readable
  C-number + Code128 strip; one label serves phone cameras and HID
  scanner guns (kiosk regex-extracts C-\d+ from typed input). Printed
  via Pillow→CUPS on the Rollo. MUST.
- R21. Show manifests: pack scans coins into location 'show:X' with a
  printed packing list; unpack diffs to EMPTY — every packed coin is
  scanned-back or sold; discrepancies render red. Sell at show = scan →
  SOLD at ask (2 taps), price editable only when different. MUST.
- R22. Bench capture (Pi + gphoto2 + D3300): scan → shoot per setup
  (xpol/axial/cpl_rake/specular…) → file lands pre-named → row inserted
  → thumbnail → only technical rejects flagged at bench. SHOULD (after
  labels/manifests).
- R23. Offline show mode (local manifest + queued sales). MAY — build
  only if a venue actually proves dead.

## 4. Integrations (glue, not platforms)

- R24. numistalib (Andy's PyPI pkg) for Numista: attach type → prefill
  EMPTY fields + reference rows; never overwrite human-entered values.
  SHOULD.
- R25. OCRE/RPC (Numishare APIs): resolve RIC/RPC citations to
  canonical URIs; store as reference kind='uri'; may prefill obv/rev
  type descriptions (same never-overwrite rule). SHOULD.
- R26. Wildwinds: link-out button only (no API). SHOULD.
- R27. CoinSnap: CSV import as attribution SUGGESTIONS, never
  authority. MAY.
- R28. Gemini: "Copy for Gemini" button packaging the card into a
  prompt (no API integration). SHOULD.
- R29. Whatnot: C-number in every listing title (convention, MUST);
  reconcile sales/fees/refunds via Seller API bulk export or
  livestream/weekly CSV; refunds flip sale status and return coin to
  final. SHOULD.

## 5. Engineering (house rules)

- R30. uv-only dependency management; ruff select=["ALL"] and mypy
  strict pass with zero violations; pytest green on CI (3.11 + 3.12);
  main stays green. MUST.
- R31. No secrets/data in the public repo: DBs, .env, keys, coin photos
  gitignored; env-var configuration; gitleaks pre-commit. MUST.
- R32. DRY: one function per concept; derived values are never stored;
  CLAUDE.md records approved architecture and invariants. MUST.
- R33. All surfaces (chat/Code/Copilot/Grok) coordinate through the
  repo: docs/ is the contract, decisions land in docs/decisions.md.
  SHOULD.

## 6. Open items

- O1. Pamela: colors for $2.50/$5/$7.50; ladder between royal ($80) and
  gray; lime-vs-green on the Severus fragment.
- O2. The $3→$450 coin: identify and seed as C-0001 (seller_attribution,
  attribution, significance, special=1) — first row and smoke test.
- O3. Balance RS-232/USB capability (auto-capture weight) — check model.
- O4. TagLens select-pattern reuse for `coinshop select` (one key per
  photo, self-emptying worklist) — design when photo selection lands.
