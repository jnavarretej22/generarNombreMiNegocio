# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

Static single-page app + one Vercel serverless function. **No build step, no `package.json`, no test suite, no linter.**

- **Local dev:** `vercel dev` (Vercel CLI required). Serves `index.html` and exposes `/api/generate`.
- **Deploy:** `vercel` (preview) / `vercel --prod`.
- **Required env vars** (in `.env.local` for local dev, or in the Vercel project):
  - If `ACTIVAR_GEMINI = true` → `GEMINI_API_KEY` (and optional `GEMINI_MODEL`, default `gemini-2.5-flash-lite`).
  - If `ACTIVAR_GROQ = true` → `GROQ_API_KEY` (and optional `GROQ_MODEL`, default `meta-llama/llama-4-scout-17b-16e-instruct`).

## Architecture

### Two-file app
- `index.html` — entire frontend (HTML + CSS + JS inline), vanilla, no framework, no bundler. Only external assets are Google Fonts.
- `api/generate.js` — single Vercel serverless function. The frontend never calls a model provider directly; it always POSTs to `/api/generate`.

### Provider switch + automatic fallback (in `api/generate.js`)
Two top-level booleans gate the **primary** provider:

```js
const ACTIVAR_GEMINI = true;
const ACTIVAR_GROQ   = false;
const ENABLE_FALLBACK = true;
```

`buildProviderChain()` puts the primary (active + key present) first; if `ENABLE_FALLBACK` is true, the *other* provider is appended as a fallback when its key is set. The handler walks the chain; if a `ProviderError` has `retryable: true` (HTTP 408/429/500/502/503/504 or network error), it moves to the next provider, otherwise it aborts.

The successful response is augmented with two metadata fields:
- `_provider`: `'gemini'` or `'groq'` — which one actually served the request.
- `_fallback`: `true` if a non-primary provider answered (the frontend logs this to console).

`ProviderError` carries `{ status, provider, retryable }`. Use it (not plain `Error`) at the provider boundary — retryable defaults to `RETRYABLE_STATUS.has(status)`.

### Wire format: OpenAI-shape in, OpenAI-shape out
The frontend speaks the OpenAI chat-completions shape (`messages`, `temperature`, `max_tokens`, `response_format`). The function:
- For **Groq**, forwards the body as-is.
- For **Gemini**, translates `messages` → `system_instruction` + `contents`, uses `generationConfig.responseSchema` (the `NOMBRES_SCHEMA` constant) for structured output, then adapts the response back to the OpenAI shape (`{ choices: [{ message: { content } }] }`).

When adding a new provider, preserve this contract: input/output normalized to the OpenAI shape at the boundary.

### Two distinct generation modes — two distinct system prompts
The UI has tabs driving **separate system prompts** (JS constants in `index.html`):

- `SP_DESCRIBE` — "Describir" mode. User types a free-form description; the **Personaliza tu nombre ideal** collapsible (`customBanner`) appends optional structured constraints (`dSelectedTones`, `dSelectedLen`, `dSelectedLang`, prefix/suffix/contains, avoid, refs) onto the user prompt as `RESTRICCIÓN — ...` lines.
- `SP_FORM` — "Formulario guiado" mode. `buildFormPrompt()` validates three required fields (what / who / industry) and assembles a `BRIEFING DEL NEGOCIO:` block with optional fields. Throws user-facing errors when required fields are missing.

Both modes append their constraints **as plain text inside the user message**, not as structured params. Two parallel pill-handler blocks exist — one for form mode (`#industryRow / #toneRow / #lenRow / #langRow`), the other for describe mode (`#dtoneRow / #dlenRow / #dlangRow` with `d*`-prefixed state vars). Keep these symmetric when adding new constraints.

Both prompts contain a matching **EXCLUSIÓN OBLIGATORIA** clause for the repeat-suppression mechanism (see below).

### JSON contract — enforced in 4 places
Both system prompts force this exact shape:

```json
{ "nombres": [ { "nombre", "estilo", "concepto", "dominio" } ] }
```

- `estilo` is constrained to one of: **Neologismo | Descriptivo | Acrónimo | Compuesto | Abstracto**.
- `dominio` is rendered as a clickable "Verificar →" link to Namecheap's domain-search results.

The contract is mirrored in 4 places — keep them in sync when changing it:
1. `SP_DESCRIBE` (system prompt)
2. `SP_FORM` (system prompt)
3. The parse/validate code in `callGroq()` inside `index.html`
4. The `NOMBRES_SCHEMA` constant in `api/generate.js` (Gemini `responseSchema` — enforces structure at model level, with `estilo` as a closed enum)

`estiloClass(estilo)` maps the model's `estilo` string to a CSS modifier (`estilo-neologismo`, `estilo-descriptivo`, etc.) for per-style badge coloring. If you add an `estilo` value, update the enum in `NOMBRES_SCHEMA`, both system prompts, and the matching CSS rule.

### Repeat-suppression across regenerations
`seenNames` is a `Map<normalized, original>` (normalization = NFD-strip-diacritics + lowercase + remove non-alphanum via `normalizeName()`). Stored persistently in `localStorage` under `lumen_business_seen_names` as `{ [promptKey]: [originals[]] }` so exclusions survive page reloads.

On each `generate(prompt, sp)`:
1. If `prompt !== lastPrompt` → `loadSeenFor(prompt)` reloads the map for that prompt key.
2. `callGroq` appends a `NOMBRES YA SUGERIDOS: ...` block to the user prompt via `buildExclusionBlock()`.
3. After the response, `addSeenName()` dedupes by normalized key (so "Aether" and "AETHER" collapse to one).
4. `persistSeenFor(prompt)` writes the updated set back to localStorage.

Manual exclusion (the ✕ button on each card) calls `addSeenName()` + `persistSeenFor()` without regenerating — the veto applies on the next regeneration. `clearSeenFor()` resets the map AND removes the prompt's entry from persisted storage — triggered by the "⟲ Olvidar N descartados" button in the results header (visible only when the count of unseen-but-tracked names is positive).

### "Más como este" (seed-based search)
Each card has a ✨ button. It calls `buildSimilarPrompt(seed)` using the seed's `nombre`, `estilo` and `concepto`, switches the UI to Describir mode, sets `descPrompt.value` to the new prompt, and calls `generate()` with `SP_DESCRIBE`. Because the prompt is a new string, it gets its own `seenNames` context.

### Search history
Every successful generation calls `pushHistory(prompt, sp, nombres)`. Stored under `lumen_business_history` (capped at 30, deduped by `(prompt, mode)` — regenerations refresh the existing entry instead of stacking duplicates).

The drawer is **tabbed**: Favoritos / Historial. Each tab has its own footer. History items expose three actions:
- **Ver nombres**: load the cached `nombres` into the grid without an API call. Re-syncs `seenNames` from `seenStore` and folds in the historical names so the count stays consistent.
- **Volver a buscar**: re-runs the generation with the original prompt + the current accumulated exclusions.
- **Quitar**: remove the entry.

### Compare favorites modal
"Comparar" button in the favorites footer opens a 4-column table modal (Nombre / Estilo / Concepto / Dominio). "Copiar como texto" exports a bullet-list. Closes via ✕, footer button, click-outside, or Esc.

### Temperature control (Tono)
A pill bar above the Generar button (Conservador 0.6 / Equilibrado 1.0 / Original 1.3). `currentTemperature` is read from localStorage on init (`lumen_business_temperature`, default 1.0) and sent in every `/api/generate` request. The backend forwards it to Gemini (`generationConfig.temperature`) or Groq alike.

### Client-side state (localStorage)
- `lumen_business_favorites` — array of favorited name objects (`{nombre, estilo, concepto, dominio}`).
- `lumen_business_seen_names` — `{ [promptKey]: [originals[]] }`. Per-prompt exclusion lists. `promptKey` is `normalizeName(prompt).slice(0, 240)`.
- `lumen_business_history` — array of `{ id, prompt, mode, nombres, timestamp }`, capped at 30, deduped by `(prompt, mode)`.
- `lumen_business_temperature` — numeric value (0.6 / 1.0 / 1.3) from the tone pills.
- `lumen_theme` — `'dark'` or `'light'`.
- `STORAGE_KEY = 'groq_api_key_baby_names'` is declared but **never used** — historical artifact from copying the baby-names sibling. The hidden `#keyPanel` is similarly inert (the API key lives server-side).

### Vercel routing
`vercel.json` rewrites `/business.html → /index.html`. The sibling baby-names project's footer links to `business.html` (and this project's footer links back via `index.html`). If you rename either entry point, update both the rewrite and the cross-link.

## Gotchas

- Footer says "Impulsado por Groq + Llama 4 Scout" but the default switch is Gemini. Cosmetic only.
- `vercel.json` only has the `business.html` rewrite — Vercel auto-detects the static + `/api` layout.
- There is no `package.json`; the function uses only built-in `fetch` (Node 18+). Don't introduce `npm` packages without adding a `package.json` and reviewing deployment implications.
- `index.html` is large (~1500+ lines with inline CSS and JS). When editing, prefer `Edit` with enough surrounding context to disambiguate — many short patterns repeat.
- `regenBtn` reads `seenNames.size` (Map, not Set) — keep that in mind if you refactor the storage type.
- `mode` in history entries is `'describe' | 'form'` (not `'combine'` like the sibling project) — `pushHistory` uses `systemPrompt === SP_FORM` to decide.
- The two pill systems (`#toneRow` vs `#dtoneRow`, etc.) are still parallel duplicated structures — a known design debt. If you unify them, keep `selectedTones` vs `dSelectedTones` semantics consistent with how `buildFormPrompt()` and the describe-mode `extras` builder consume them.
