# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

FutHub Store ERP — a single-page inventory/sales management app for a football jersey resale business (imports from China, sells in Belo Horizonte/MG, Brazil). All UI text and domain vocabulary are in Portuguese (pt-BR).

## Repository structure

The entire application is one file: **`index.html`** (~1700 lines — CSS in `<style>`, markup, and all JS in one `<script>` block at the bottom). There is no build step, no package manager, no bundler, and no test suite. `README.md` has no content beyond the project name.

Because everything lives in one file, always use `grep`/section markers (JS is divided by `// ── SECTION NAME ──` comments, e.g. `AUTH`, `ESTOQUE`, `VENDAS`, `FORNECEDORES`, `FINANCEIRO`) to jump to the relevant part rather than reading the whole file for small changes.

## Running locally

No server-side code — open `index.html` directly in a browser, or serve it statically (e.g. `python3 -m http.server`) if the browser blocks module/fetch behavior from `file://`. There is no lint or test command to run after changes; verify by loading the app in a browser and exercising the changed flow.

## Architecture

**Backend:** Supabase (Postgres + PostgREST + Auth). Two separate access paths are used:
- `supabase-js` (loaded via CDN) is used **only for Auth** — `signInWithPassword`, `getSession`, `signOut`.
- All data reads/writes go through a hand-rolled REST wrapper (`sbFetch` → `db.get/post/patch/delete`) that calls the PostgREST endpoints directly with the logged-in user's JWT as the bearer token. `supabase-js`'s query builder (`.from().select()`, etc.) is *not* used — keep new data code consistent with the `db.*` wrapper pattern.
- `SB_URL` and `SB_KEY` are hardcoded near the top of the `<script>` block. `SB_KEY` is the publishable/anon key (safe to expose client-side); do not replace it with a service-role key.

**Client-side cache:** a single global `C = { produtos, clientes, fornecedores, vendas, loaded }` object caches fetched tables in memory. Each entity has an `ensureX(force)` loader that skips refetching unless `force=true` or the cache is empty (`C.loaded.<entity>`). After any mutation (create/update/delete), call the relevant `ensureX(true)` followed by the matching `renderX()` to refresh both cache and DOM — this pattern is used everywhere and should be followed for new mutations.

**No server-side joins:** related data (e.g. `itens_venda` → `vendas`, `produtos` → `pacotes`) is fetched separately and joined in JavaScript against the in-memory cache, not via PostgREST embedding. See `ensureVendas()` for the pattern (fetches `vendas` and `itens_venda` separately, builds a `venda_id → itens[]` map, then computes totals/profit client-side).

**Navigation / rendering:** single-page app via plain DOM toggling, no router/framework.
- `go(id)` shows `#pg-<id>`, marks the matching sidebar `.nav-item` active, and calls the loader in the `LOADS` map (`{dash: loadDash, estoque: loadEst, ...}`).
- Each page has a `loadX()` (ensures cache is populated) and a separate `renderX()` (pure re-render from `C`, no fetch) — call `renderX()` alone when only re-filtering/re-sorting already-loaded data (see `renderEst()` on search/filter input).

**Tables (inferred from queries, no schema file in repo):** `produtos`, `clientes`, `fornecedores`, `vendas`, `itens_venda`, `pacotes`, `pacote_produtos`.

**Domain model:**
- *Produto* = one SKU/size line item with a cost buildup: `custo_usd × cotacao_brl (BRL/USD rate) + frete_unit_brl + imposto_brl = custo_total_brl`, compared against `preco_venda_brl` for margin.
- *Pacote* = a shipment grouped by tracking code (`rastreio`/`codigo_rastreio`), linked to 17TRACK (`https://t.17track.net/pt-BR#nums=<code>`) for tracking. Multiple produtos can share one pacote/tracking code; editing a produto's status can cascade to other produtos sharing the same tracking code (see the "exclusivos vs multi-pacote" logic in `saveEdit()`).
- *Venda* (sale) vs *Reserva* (reservation for out-of-stock/in-transit items): reservas don't deduct stock until confirmed via `confirmarReserva()`; vendas deduct stock immediately and restore it on `cancelarVenda()`.
- *Financeiro* page computes a "buffer" against Brazilian import tax risk (`canal vermelho` / customs red channel) — reserves a % of revenue against potential tax on in-transit inventory value (`calcBuf()`).

## Conventions to follow when editing

- Keep new JS inside the existing section-comment blocks (or add a new `// ── NAME ──` block) rather than introducing separate files/modules — the single-file structure is intentional here, not an oversight to "fix".
- Follow the existing `toast(msg, type)` pattern for user feedback and `confirm()` for destructive/consequential actions (delete, cancel, status changes with cascading effects).
- CSS custom properties in `:root` (gold/dark theme) drive all styling — reuse existing `--gold`, `--bk*`, `--tx*`, badge classes (`.b-ok`, `.b-warn`, `.b-danger`, `.b-info`, `.b-gold`), etc. instead of hardcoding new colors.
- Currency and date formatting go through the existing `R()` and `fmtD()` helpers — don't reimplement formatting inline.
