# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A self-contained demo of an operations cockpit for a fictional Swiss contractor, Vanteo Construction SA. Everything is in a single `index.html` (~860 KB). It has two modes: a live 3D site map built with three.js, and a back-office covering sites, crews, timesheets, CRM, quotes, invoices, purchasing, equipment, SUVA safety and settings.

The UI is trilingual: French (`fr-CH`, the source language), English and Latin American Spanish (`es-419`). Source code, code comments and UI strings in the code stay in French, with inclusive forms such as `affecté·e` and `Conducteur·rice`. Translation happens at display time (see [Internationalization](#internationalization-window-i18n)). Every new UI string needs an English and a Spanish entry in the dictionary.

The project has no build step, package manager, linter, test suite, backend or git repository. All data lives in the browser's `localStorage`.

## Running and verifying

- Open `index.html` directly, or serve the folder statically (e.g. `python3 -m http.server`).
- You need network access: three.js 0.165.0 loads from jsDelivr through the `<script type="importmap">`, and the Outfit font loads from Google Fonts.
- The URL hash picks the view: `#map`, or `#bo/<moduleId>` (e.g. `#bo/invoices`).
- `?lang=fr|en|es` picks the language (also the FR/EN/ES switch in the top bar, stored in `vx-lang`).
- To find untranslated strings: set `localStorage['vx-i18n-debug'] = '1'`, reload in EN or ES, browse, then read `I18N.missing` in the console. Most entries are demo data (names, dates, amounts, task titles), which stays French by design.
- To reset to fresh demo data, use Back-office → Paramètres → Réinitialiser, or run `localStorage.removeItem('vanteo-demo-v1')` and reload.
- There are no automated tests. Verify a change in the browser and check the console. `APP.open` catches errors thrown by a back-office module's `render` and shows them inline in `#bo-content`.

## Navigating the file

- Find code by the `/* ==== … ==== */` banner comment at the top of each block, not by line number.
- The last `<script type="module">` is the 3D map. It is a minified bundle whose lines run to about 130k characters, so don't Read it whole. On those lines, `grep` with bounded `.{0,N}` context can hit ugrep's complexity limit. A short `python3 -c` slice around `s.find('needle')` works reliably.

## Architecture

### Load order

Each classic script is an IIFE that exposes one global. They load in this order:

1. Three `<style>` blocks:
   - shell and common components, with design tokens in `:root` (`--tq*` turquoise, `--ink*`, status colors);
   - back-office (`.bo-*`, tables, kanban, gantt…);
   - map (`.mx-*`, with its own `--m-*` variables on `#map-root`).
2. **`window.I18N`** (translation engine), then the dictionary between `<!-- I18N-DICT:START -->` and `<!-- I18N-DICT:END -->`, then `<script>I18N.start();</script>`.
3. **`window.VX`**: data layer. Holds the seed data, state, events, business actions, KPIs and `alerts()`, `dayPlan(emp)`, and the `VX.util` helpers.
4. **`window.UI`**: shared UI pieces (icons via the `UI.P` path map, logo, avatar, toast, modal, statusChip).
5. **`window.APP`**: application shell. Handles map/back-office mode switching, the module registry, global search (⌘K), the alerts bell, the language switch and `APP.focus`.
6. Between `<!-- BO-MODULES:START -->` and `<!-- BO-MODULES:END -->`: **`window.BO`** (shared back-office toolkit), then one script per back-office module.
7. `<script>APP.boot();</script>`
8. The minified 3D map ES module, which uses the globals above.

### VX is the single source of truth

- **Persistence.** `VX.S` is stored as JSON under the `localStorage` key `vanteo-demo-v1`. `VX.load()` re-seeds when the stored state is missing or `S.v !== 1`.
- **Seed data.** It is generated relative to today (`rel(days)`) with a deterministic PRNG (`rng(hash(...))`), so the demo always looks current.
- **Mutations.** Go through a `VX` action: `assign`, `unassign`, `moveTo`, `setTaskStatus`, `addTaskProgress`, `createTask`, `payInvoice`, `setQuoteStatus`, or the generic `upsert(coll, obj, evt)` and `remove(coll, id, evt)`. If you write to `VX.S` directly, follow it with `VX.save()` and `VX.emit('<evt>', …)`.
- **Events.** `emit` fires the named event, then a generic `'change'` event. This is what keeps the map and the back-office in sync, so a mutation that doesn't emit won't appear in the other view.
- **Events the map listens to.** `assign`, `unassign`, `move`, `task:new`, `site:new`, `sites`, `payment`, `victory`, `reset` and `change`. `victory` triggers the celebration effect when a task is finished, an invoice is fully paid or a quote is accepted. `notice` shows a toast.
- **`BO.ensure()` is the backfill/migration layer.** It lives in the BO base script and runs at load and on `reset`. It idempotently adds collections that `seed()` doesn't create (`ext`, `plan`, `toolbox`, `absences`, `maintenance`) and derived fields (site `contract`, `cfc`, `billedHist`; invoice `ht`, `reminders`; …).
  - Browsers keep their stored state, so a new field or collection added only to `seed()` never reaches existing users.
  - Backfill it in `ensure()` instead, with the `def()` pattern or a `== null` guard.
  - Don't bump `v`: that re-seeds and wipes the user's data.
- **Map preferences.** The map's own settings use separate keys: `vx-map-mute`, `vx-map-tray`, `vx-map-tts`.
- **Import/export.** Settings exports the whole of `VX.S` as JSON. Import writes it straight into `vanteo-demo-v1` and reloads the page.

### Back-office modules

- **Registration.** Each module ends with `APP.register({ id, label, icon, group, order, render(el, params), badge() })`.
  - `group` creates the nav sections and `order` sorts them.
  - `badge()` returns the nav counter.
  - `APP.open(id, { id, tab })` opens a module on a specific record or tab.
- **Rendering.** Modules render in full through `innerHTML`.
  - On every VX `'change'`, `BO.schedule()` debounces and calls `APP.refresh()`, which re-renders the current module and any open drawer.
  - The refresh is postponed while `BO.busy()` is true: a modal is open, a drag is in progress, or a text field is focused.
  - Keep view state (filters, sort, tab) in the module-level `st` object, not only in the DOM, and keep `render` idempotent.
- **Navigation.** It is declarative. One capture-phase click listener in BO handles these attributes:
  - `data-open="module:id:tab"`
  - `data-open-emp="<empId>"` (employee drawer)
  - `data-open-task="<taskId>"` (task drawer)
  - `data-map="type:id"` (jump to the 3D map)

  Use these attributes rather than wiring handlers on individual elements.
- **`APP.focus({ type, id })`** shows an object. The map's handlers in `APP.focusHandlers` try first (site, employee, task, equipment). Otherwise it opens the matching back-office module.
- **Reuse the BO toolkit:**
  - `BO.table` (config-driven sortable table)
  - `BO.formHtml`, `BO.formModal`, `BO.formRead` (forms built from field specs, with validation)
  - `BO.drawer`, `BO.closeDrawer`, `BO.confirm`
  - `BO.head`, `BO.kpi`, `BO.seg`, `BO.tabs`, `BO.opts*`
  - `BO.csv` with `BO.download`
  - `BO.nextNo(coll, prefix)` for `PREFIX-YYYY-NNN` document numbers
  - `BO.siteFin(id)` for site financials
  - `BO.printHtml` for printable previews
- **Icons.** A comment in the BO base marks ui.js (now the `window.UI` block) as not editable. Add new icons to the `XP` map in the BO base and render them with `BO.icon(name)`.
- **Escaping.** All markup is built from template strings. Escape every data value with `esc()` (`VX.util.esc`).

### 3D map

- The map's source is not in this repository; only the minified bundle is.
  - It mounts into `#map-root` and uses three.js addons (OrbitControls, GTAO post-processing…).
  - It talks to the rest of the app only through `VX`, `UI` and `APP`.
  - Make behavior changes on the VX/BO side where you can. If the bundle itself must change, keep edits minimal and targeted.
- Sites have a world-space `pos: { x, z }`, and new sites get one from `BO.findFreePos()`. The map manages each task's `anchor`.
- On `reset`, the map reloads the page if the set of sites has changed.
- The map's assistant panel takes typed commands, voice input (SpeechRecognition) and reads replies aloud (speechSynthesis). All of it runs locally, with no API calls. Its rule-based parser only knows French keywords; see the next section for how EN/ES input reaches it.
- The bundle carries a few small i18n hooks: `I18N.locale` / `I18N.dec` for number and date formatting, `I18N.t` around canvas `fillText` labels, `I18N.voice` / `I18N.speak` / `I18N.speechLang` for speech, `I18N.cmdFr` at the start of the command parser (`function Fo`) and of the new-project description parser (`function Ks`), and `I18N.tr` in the speech bubble (`function Ia`).

### Internationalization (`window.I18N`)

- **How it works.** A MutationObserver rewrites text nodes and the `placeholder`, `title`, `aria-label` and `alt` attributes, plus `value` on buttons and disabled or read-only inputs. It looks each French string up in the dictionary. FR does nothing, so the French UI is the code as written.
  - It skips `<textarea>`, editable `<input>` values, `[translate="no"]` and `.notranslate`.
  - Changing language reloads the page (`I18N.set`). Number and date formats follow `I18N.locale` (`fr-CH`, `en-CH`, `es-419`).
- **Dictionary.** It is a series of `I18N.add({ 'texte français': ['English', 'Español'] })` calls between the `I18N-DICT` markers. The key is the exact displayed French text, with whitespace collapsed and `'` normalized to `’`.
  - `{name}` is a variable part, translated in turn if it is itself a key. `{n}`, `{n2}` match numbers only.
  - A translation can be a function `(vals) => string`.
  - Without an exact or pattern match, the engine splits on ` · `, ` — `, ` – `, ` → `, ` | `, a trailing `:`, `Label : value` and parentheses, and translates each piece.
  - Avoid over-generic patterns such as `'{0} est {1}.'`: they also rewrite French demo data such as notes. Anchor patterns on distinctive literal words.
- **Text outside the DOM** needs an explicit call:
  - Message templates in `<textarea>`/`<input>` (`BO.compose` reminders and follow-ups): `I18N.t('Bonjour {contact},\n…', { contact, … })`. The key is the full French template. In FR it just fills the placeholders.
  - CSV cells use `I18N.tr`. Print previews run `I18N.translateTree` on the iframe. Canvas labels in the map use `I18N.t`.
- **Data stays French.** Demo data (names, task titles, notes, invoice labels) is never translated. Stored enum values (statuses, types, weather…) stay French in `VX.S`; only their displayed label is translated through the dictionary.
- **Assistant commands in EN/ES.** `I18N.commands('en' | 'es', { 'regex alternatives': 'mot-clé français' })` maps words (lowercased, accents and punctuation removed) to the French keywords the map's parser understands, before parsing. The longest keys apply first.
  - Rules run in sequence, so a replacement must not contain a word that a later, shorter rule rewrites (e.g. `present` → `presents` would break `retour au present`).
  - The same vocabulary feeds the new-project description parser (floors, pool, bridge…). Task vocabulary (`slab` → `dalle`, `formwork` → `coffrage`…) lets EN/ES commands match the French task titles.
- **Terminology.** chantier = site / obra, situation = progress claim / estimación, offre = quote / cotización, conducteur de travaux = construction manager / gerente de obra, chef de chantier = site manager / jefe de obra, dépôt = yard / patio, journal = site diary / bitácora de obra, HT = excl. VAT / sin IVA, TTC = incl. VAT / con IVA. Spanish uses Latin American vocabulary (costo, celular, usted).

### Swiss domain conventions

- **Amounts.** CHF, formatted with `chf`, `chf2` or `chfK`. Totals including VAT are rounded to 5 centimes with `round5`.
- **VAT.** `BO.tva()` returns `S.company.tvaRate` if set, otherwise `VX.TVA` (8.1 %).
- **Cost codes.** Cost lines use CFC codes (`VX.CFC`).
- **Invoices.** They include a QR-bill preview with a 27-digit reference (recursive mod 10, `BO.qrRef`). Progress invoices ("situations") use a 5 % retention (SIA 118).
- **Dates.** Stored as `YYYY-MM-DD` strings. Use the `VX.util` helpers (`today`, `addDays`, `diffDays`, `weekStart`, `fmtDate`) instead of raw `Date` arithmetic.
