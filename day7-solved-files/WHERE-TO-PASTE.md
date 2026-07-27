# Day 7 — Solved Files & How To Run

Day 7 is the frontend day. Zero backend Java changes — everything
happens inside the `static-dashboard/` folder at the project root.

---

## What this folder ships

| File | Ticket(s) | What it does |
|------|-----------|--------------|
| `static-dashboard/dashboard.html`    | ADV098–100, ADV102, ADV104 | Page shell, KPI tiles, danger alert, SSE feed area |
| `static-dashboard/css/style.css`     | ADV099–103, ADV106          | Design tokens, dark theme, animations, responsive breakpoints, table styles |
| `static-dashboard/js/theme.js`       | ADV100                      | Dark/light toggle persisted to localStorage; no-FOUC IIFE |
| `static-dashboard/js/sse.js`         | ADV104–105                  | EventSource connection + prepend-and-animate + 50-entry DOM cap |
| `static-dashboard/trades.html`       | ADV106                      | Trade blotter page with sortable/resizable/frozen-header table |
| `static-dashboard/js/trades.js`      | ADV106                      | Click-to-sort, drag-to-resize, fetches from /api/v1/trades |

---

## Quick start — copy all solved files at once

```bash
# From the project root — one-shot overlay:
cp -R day7-solved-files/static-dashboard/ static-dashboard/
```

---

## Running the dashboard

You need two terminals.

### Terminal 1 — backend (optional but needed for real data)

```bash
cd backend
./mvnw spring-boot:run
# Runs on http://localhost:8081/api
```

If the backend is not running, both pages fall back to hardcoded demo
data so animations and sort/resize can still be demonstrated.

### Terminal 2 — static file server

```bash
cd static-dashboard
python3 -m http.server 5500
```

Then open:
- **Dashboard** → http://localhost:5500/dashboard.html
- **Trades table** → http://localhost:5500/trades.html

> **Why a server?** Loading via `file://` triggers CORS errors on `fetch`
> and `EventSource`. Always use the HTTP server.

---

## What to verify (end-of-day checklist)

### dashboard.html
- [ ] CSS Grid shell renders: header top, sidebar left, main beside it, footer pinned bottom.
- [ ] Theme toggle flips `data-theme` on `<html>` and persists across reloads — zero white flash on reload.
- [ ] Three demo trade cards slide in with the `translateX` + `fade-in` entrance animation.
- [ ] The orange danger alert pulses via the `pulse` keyframe.
- [ ] The `#sse-status` badge reads "Live" when the backend SSE stream is connected.
- [ ] Resizing below 720 px hides the sidebar; no horizontal scroll at 375 px.

### trades.html
- [ ] Table loads with rows (from backend or demo fallback).
- [ ] Clicking a column header sorts rows; clicking again reverses direction; ▲/▼ indicator updates.
- [ ] `aria-sort` attribute on the active `<th>` matches the sort direction (check in DevTools).
- [ ] Dragging a resize handle widens/narrows the column; drag continues outside the handle.
- [ ] Scrolling the table body keeps the header row pinned at the top.

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Blank feed / "Connecting…" stays | Backend not running — demo events fire anyway via setTimeout |
| CORS error on `fetch` | You opened the HTML via `file://` — use `python3 -m http.server 5500` |
| Theme flashes white on reload | Inline IIFE in `<head>` is missing or is after the `<link>` — check `dashboard.html` |
| Sort doesn't change order | Check `data-col` on `<th>` matches the property name on the trade object |
| Sticky header scrolls away | An ancestor has `overflow: hidden` — remove it; `overflow: auto` is fine |
| Drag stops when cursor leaves handle | Mouse listeners are on the handle, not `document` — fix in `trades.js` |
