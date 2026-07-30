# 0005. Photos are append-only evidence; culling is a flag

Date: 2026-07-29 · Status: accepted

Context: for ancients, photography is evidence gathering — a raking
frame that looks bad may show the privy mark that makes a $4 coin a
$400 coin, recognized weeks later. Pamela's edge is reverse/marks
detail that others discard.

Decision: photo rows are never deleted; status is evidence, selected,
or technical_reject; only focus/shake failures are judged at the
bench. naming.py owns the filename grammar; no photo without a
C-number.

Consequences: disk is spent instead of evidence; selection is a
later, separate pass (TagLens interaction pattern, O4). (R3, R4)
