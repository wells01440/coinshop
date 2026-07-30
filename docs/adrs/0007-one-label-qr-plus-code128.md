# 0007. One flip label: QR (tailnet URL) + Code128 + human C-number

Date: 2026-07-29 · Status: accepted

Context: phone cameras want URLs; HID scanner guns type whatever they
see; humans want a readable ID; two label formats would drift.

Decision: a single label carries all three; consumers receiving the
typed URL string regex-extract the C-number. Rendered with segno +
python-barcode + Pillow at 203 dpi, printed via CUPS to the Rollo.

Consequences: one template to maintain; reprints are free from data
(see ADR 0009). (R20)
