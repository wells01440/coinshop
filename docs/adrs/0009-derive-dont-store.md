# 0009. Derive, don't store

Date: 2026-07-29 · Status: accepted

Context: Pamela's physical system encodes meaning on purpose (sticker
color = price tier, star = silver, heart = special); storing those
digitally creates sync bugs between price and color, material and
star. Her writeup is a fixed field order, not freeform prose.

Decision: sticker color/tier, star, heart, ROI, and the attribution
delta are pure functions of stored fields; the writeup is RENDERED
from structured fields in her card order (golden test: Diva Helena).
Manual override exists but is never required.

Consequences: physical/digital cross-check for free; a price change
re-derives the sticker; one source of truth per fact. (R10, R13, R14)
