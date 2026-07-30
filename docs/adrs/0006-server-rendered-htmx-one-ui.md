# 0006. Server-rendered htmx; one UI for phone and kiosk

Date: 2026-07-29 · Status: accepted

Context: surfaces are Pamela's phone, the shop iPhone 11, and the Pi
7-inch kiosk; a SPA means a JS build chain and two codebases;
adoption constraints R11-R12 demand speed, not richness.

Decision: HTML rendered by the API with htmx for writes; the kiosk is
the same pages fullscreen. A future native shell (SwiftUI scanner +
webview) wraps these pages rather than replacing them.

Consequences: no build step; pages are curl-debuggable; rich
interactions are out of scope by design.
