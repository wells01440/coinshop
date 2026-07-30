# 0003. Hub API, Tailscale-only, no public exposure

Date: 2026-07-29 · Status: accepted

Context: Pamela's phone, the shop iPhone, the Pi kiosk, and the show
floor all need access; the internet does not.

Decision: the API binds on the homelab, reachable only over the
tailnet; QR labels encode tailnet URLs; auth is tailnet membership
plus per-session actor selection — no user/password system.

Consequences: zero public attack surface; devices must run Tailscale;
cell coverage at venues is the availability story (R23 fallback).
