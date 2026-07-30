# 0002. One SQLite file, one writer process

Date: 2026-07-29 · Status: accepted

Context: household-scale multi-user system; Postgres adds an ops tax
with zero benefit at this volume; naive file-sharing SQLite across
devices corrupts under two writers.

Decision: a single SQLite file in WAL mode, owned by exactly one
process — the FastAPI service. All other surfaces are HTTP clients.
CLI may write directly only in single-user bootstrap mode.

Consequences: trivial backup (one file); no migrations infra beyond
executescript; scaling ceiling irrelevant at inventory scale. (R1, R17)
