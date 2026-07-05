# Changelog

All notable changes to Kapruka Sia. Format loosely follows [Keep a Changelog]; dates are
Asia/Colombo.

## [Unreleased]

## [0.24.1] — 2026-07-06 — Accessibility & visual polish

Frontend-only pass. No backend, agent, or MCP behaviour changed — every diff is in
`frontend/` CSS/JS. Asset cache-busting versions bumped to `0.24.1` so browsers refetch.

### Added
- **Global keyboard focus ring.** A funnel-wide `:focus-visible` violet outline now covers
  links, buttons, inputs, and `[tabindex]` elements, so product CTAs, quick-replies, the
  sidebar, pager, and modal controls all show a visible focus indicator. Zero-specificity
  (`:where`) so components with their own focus cue keep overriding it.
- **Undo on cart removal.** Removing a cart line now surfaces an 8-second action toast with an
  **Undo** button that re-adds the item; the toast pauses its timer on hover/focus. Driven by a
  new `sia:cart:removed` event (`app.js`) and a `showActionToast` helper (`enhance.js`).
- **Localized product-card chrome.** The card's own static labels (Add / Compare / Delivery /
  Save, kickers, stock/pick badges, fallback reason chips) now render in EN / Sinhala / Singlish
  / Tamil via `PRODUCT_CARD_COPY`. Backend-supplied fields were already localized.
- **Variant overlay focus trap.** Tab now cycles within the size/colour sheet, focus lands on the
  close button on open, and closing (Esc or ×) restores focus to the trigger card.
- **Reduced-motion coverage.** `prefers-reduced-motion` now also stills the order-tracking route
  token / bob animations.

### Changed
- **WCAG AA contrast fixes.** `--faint` darkened on light surfaces and lightened on dark to clear
  AA; a new `--green-ink` token replaces fill-green for all green *text* (kickers, state pills,
  chips, order/tracking accents) where the brand green failed as small text.
- **De-cluttered product cards.** Dropped the featured pick's yellow gradient tint and gold
  top-bar, flattened the product-image placeholder (removed the radial-gradient tints), and
  restyled the delivery note from a boxed yellow badge to plain muted text.
- **Single-row card actions.** Add / Compare / Delivery / Save now sit on one 4-column row instead
  of a 3-column row with Save wrapping below.
- **Less duplicated "why this one".** Reason chips now drop chips that merely restate the pick
  identity or stock level shown elsewhere on the card, and the `best-for` line is hidden when a
  highlighted pick-reason is already present.
- **Softer order motion.** Shimmer, dash, bob, confetti, and route animations run **once** instead
  of looping forever; the live-status blink is slowed and gentled (2.4s ease, 0.5 min opacity).
  The order-row left accent bar became a full outlined border.
- **Bigger tap targets.** The cart remove button gained a 44px hit-slop (WCAG 2.5.5) without
  changing its 26px visual size; the cart total hides when the sidebar is collapsed.
- **Semantic tokens.** Magic z-index numbers replaced with `--z-raised/-toast/-tour/-overlay`;
  Noto Sans Sinhala/Tamil added to the display font stack.
- **Screen-reader hygiene.** Decorative `role="img"` placeholder divs marked `aria-hidden`; sidebar
  buttons given explicit `aria-label`s.

## [0.24.0] — 2026-07-05 — Demo session loader, delivery gate at preview, rate-limit surfaced, sidebar identity

### Added
- **Demo Session Loader** for live judging. A new gated `POST /api/session/load-demo` mints a
  fresh identity token mapped to the configured `DEMO_CUSTOMER_ID` and returns
  `DEMO_SESSION_ID`, so a judge on any browser can adopt the fully-populated seed shopper
  with one click. The endpoint 404s unless `ENABLE_DEMO_RESET=true` and both IDs are set —
  it can't be smuggled through `localStorage` because the identity cookie is HttpOnly and
  reads are gated by cookie → session ownership. A "Load Demo Session" button joins the
  sidebar (behind the same flag as the persona reset, so prod never shows it). See
  `DEMO_SEED_NOTES.md` for the seed/deploy steps. *(PR #65)*

- **Delivery gate at checkout preview, not just `create_order`.** Before this fix, an
  undeliverable city (e.g. a saved recipient stored as bare `"Colombo"`, which Kapruka
  rejects — it needs `"Colombo 03"`) passed the review and only failed at the final
  payment confirm. A shared `verify_delivery()` tri-state gate (`available` /
  `unavailable` / `unknown`) is now called at both preview entry points (`/api/checkout/preview`
  REST and the agent's `preview_checkout`) and reused by `create_order`. Preview blocks only
  on a definitive `unavailable` and fails open on `unknown` / rate-limit / transient — the
  create path still re-checks before charging, so a transient blip never double-charges. The
  agent is now nudged to resolve the exact city via `kapruka_list_delivery_cities` first.
  *(PR #65)*

- **Sidebar name + city now come from saved context.** Returning shoppers no longer show
  as "Guest" in a stale location. The sidebar name is filled from the most recent checkout
  *sender* name when the shopper hasn't introduced themselves in chat — a chat
  self-introduction always wins, so a typed `"I'm Sanjay"` is never silently overwritten by
  the remembered sender. The welcome city now prefers the most-recently-used saved
  recipient / address city, falling back to the latest order's city only when nothing is
  saved. *(PR #60)*

- **`show_last_results` local tool re-populates the canvas.** Blocks only render on the
  canvas when a tool runs this turn, so asking Sia *"show me those again"* previously
  re-listed the products in chat but left the canvas on the recipients/cart pane. The last
  `products` / `product_groups` block is now persisted per session; the no-arg
  `show_last_results` tool re-emits it through the existing frontend render path. Cleared
  after a successful `create_payment_link` so a paid order doesn't pop up mid-chat. *(PR #58)*

- **Voice mode stop path hardened.** Review of 0.22.0's voice countdown found three small
  bugs: `onerror` nulled recognition without flagging, so the always-following `onend`
  could restart on null (silent) and send a stale transcript; the keep-alive had no backoff
  so a browser that ends recognition instantly could spin `start() → onend` for the full 30s;
  the initial `start()` catch was silent. `onerror` now sets `aborted`, `onend` returns
  early when aborted, a 200ms minimum session is required before restart, and the start
  catch surfaces a user-visible message like `onerror`. *(PR #56 follow-up)*

### Fixed
- **Kapruka rate limits are now real failures, not successful empty results.** Kapruka
  returns rate limits as an HTTP-200 body (`isError=False`), e.g. *"Error: Rate limit
  exceeded. Wait a moment before retrying."* Without explicit detection this looked like a
  successful result: `tool_stats` never recorded a failure, the circuit breaker never
  tripped, and checkout told the shopper "please retry" — inviting a retry loop that dug
  deeper into the upstream limit.

  - `kapruka_client`: a new `KaprukaRateLimitError` is raised by the central `_call_once`
    detector the moment a rate-limit phrase appears in any tool's text content. The
    tenacity retry now uses `retry_if_not_exception_type(KaprukaRateLimitError)`, so one
    refused call stays one refused call — never amplified into three upstream calls.
  - `checkout_service`: `create_order` catches the rate limit distinctly. Instead of a
    generic "please retry" it returns `error="rate_limited"` with a wait message and a
    30-second per-session cooldown that blocks re-submits — both for `preview_checkout`
    and the delivery re-verify inside `create_order`.
  - `product_lookup`: `kapruka_get_product` is case-insensitive (verified live), so the id
    is lowercased up front. The old uppercase → lowercase double call is gone, and the
    variant-picker lookup and the cart re-read for the same product share one cached fetch.
  - `local_tools`: `create_payment_link` no longer auto-retries on `rate_limited`.
  - Tests: new `test_mcp_rate_limit.py`; variant / reorder mocks now use the real
    case-insensitive / canonical-id behavior. *(PRs #64 + #63 + #65)*

- **Rate-limit refusals don't poison the read cache.** A separate, follow-up guard in
  `_call_once` ensures a refusal isn't written into the 30-minute TTL Redis read cache —
  the next shopper would otherwise get a stale cache hit on the same id. *(PR #64)*

- **Null LLM content no longer crashes checkout (or pretends to be a greeting).**
  MiniMax's Anthropic-compatible endpoint returns `content=None` when the model emits
  garbled tool-artifact output. The orchestrator iterates `resp.content` in many places
  (`_text_of`, `_serialize_content_blocks`, tool extraction), so a null response raised
  `TypeError` and 500'd the stream — surfacing to the user as the *"Kapruka is slow to
  respond"* fallback and blocking checkout on the final confirmation. `content` is now
  coerced to `[]` at both acquisition points (`provider.create_message` for the JSON
  `handle_turn` and non-stream branch, and after the streaming `get_final_message`), a
  warning is logged when the coercion fires (so the upstream issue is visible), and an
  empty turn renders as the live `"What would you like to do next?"` placeholder — never
  as the cheerful *"what are you shopping for today?"* greeting (which silently turned a
  garbled-mid-checkout turn into a "successful" greeting). *(PR #62)*

- **`show_last_results` keeps checkout honest.** A 0.24 test pins that the tool is *not*
  re-emitted on a checkout turn — resurrecting search results during a payment-link flow
  would have hijacked the canvas. *(PR #58 tests)*

- **`remember_sender_name` is never destructive.** A pinned test fills the sidebar name when
  unset, and a chat self-introduction always wins over the remembered sender name —
  preventing a regression where checkout could silently overwrite a name the shopper typed
  in chat. *(PR #60 tests)*

- **Duplicate "LOW STOCK" badge removed from product card kicker.** Non-pick cards used to
  render `p.stockBadge` *and* the top-left stock overlay, so the same warning showed twice.
  The kicker now shows the Kapruka match hint only. *(PR #59)*

## [0.23.0] — 2026-07-05 — Tamil language support

### Added
- **Tamil is now a first-class language across Sia**, alongside English, Sinhala, and
  Singlish. Sia detects Tamil script each turn and replies in Tamil (the system prompt's
  mirror/re-detect rules, English catalog-term search bridging, and negation/confirmation
  word lists all now cover Tamil — e.g. `வேண்டாம்`/`இல்லை` negators, `சரி`/`ஆம்` confirmations).
- **UI Tamil mode.** A `தமிழ்` button joins the sidebar language pill; placeholders, the
  welcome-hero title/subtitle/greeting, and voice-panel copy are all localized to Tamil.
  `Noto Sans Tamil` is loaded so Tamil glyphs render natively, and script detection
  (`஀–௿`) auto-switches the UI when the shopper types Tamil.
- **Tamil voice input.** Browser speech recognition uses the `ta-LK` locale when Tamil mode
  is active, so spoken Tamil is transcribed and sent like the other languages.

## [0.22.0] — 2026-07-05 — Voice mode: 30s countdown + stop button

### Fixed
- **Voice mode closed itself after a few seconds.** Browser Speech Recognition ran with
  `continuous = false`, so `onend` fired on the first natural pause and tore the listening
  panel down before the shopper finished speaking. Recognition now runs with
  `continuous = true` and is restarted inside `onend` until a 30-second window elapses or the
  shopper stops it — so a pause no longer ends the session. Asset versions bumped to bust the
  cached `app.js`/`styles.css`.

### Added
- **30-second countdown + stop button in the listening panel.** The panel shows a live
  `m:ss` countdown that auto-stops (and sends the transcript) at zero, plus a red stop button
  that ends the recording and sends immediately. The mic button and hero mic still toggle;
  clicking either while listening now routes through the same stop-and-send path.

## [0.21.0] — 2026-07-04 — Admin IP bans, hero voice feedback, IP→country, demo persona reset

### Fixed
- **Voice input gave no visual feedback on the welcome hero.** The listening panel (blue
  wave animation + live transcript) lived inside the composer, which is hidden on the hero,
  and the `is-listening` state was toggled only on the hidden composer button — so clicking
  the hero mic recorded audio but showed nothing. The panel is now re-parented into the hero
  (mirroring the existing `#input` re-parenting), the listening/pulse state toggles on
  whichever mic button is visible, and asset versions were bumped to bust the cached
  `app.js`.

### Added
- **Manual IP ban/unban from the admin panel** (Runtime tab → *IP bans*). Banned addresses
  are written to a shared Redis key (`sia:banned_ips`) that the shopper backend reads in its
  security middleware and blocks with a 403 on every request — one chokepoint, all endpoints.
  IPs are validated and canonicalized with `ipaddress` on both sides (so an IPv6 ban matches
  regardless of formatting); the shopper read is cached in-process for 10s and **fails open**
  (never raises — serves the last-known-good set on a Redis blip), so an outage never turns
  the gate into a 500 on every request. Ban writes require a reachable Redis and return 503
  otherwise, rather than reporting a success that would never be enforced. Complements the
  existing per-IP slowapi rate limiting.
- **Admin Customers grid now shows each customer's last-IP country** (flag + name). The
  admin backend resolves the IP via the free `ip-api.com` service and caches the result in
  Redis (`sia:geoip:<ip>`, 30-day TTL), so each IP hits the API at most once — subsequent
  grid loads resolve from Redis. Private/loopback/reserved IPs (including the Docker gateway)
  are never sent to the API, and network errors are left uncached so they retry rather than
  poisoning the cache with a negative.
- **Demo-only "New Test Persona" reset in the nav bar** lets one browser walk through
  multiple shopper personas for QA/demo: it wipes the identity cookie (HttpOnly, so it
  clears via the backend), the current chat/cart/checkout intent, and all
  local/sessionStorage, then reloads as a brand-new customer. Gated behind
  `ENABLE_DEMO_RESET` (backend, default off → 404) and `SIA_DEMO_RESET_ENABLED` (frontend,
  default off → button hidden); enabled only in this repo's contest-demo compose stack.

### Security
- **Client IP is now derived by walking `X-Forwarded-For` right-to-left for the first public
  address, not the spoofable first hop.** `core/client_ip.py` previously preferred the
  client-supplied first XFF entry, which let a caller forge their apparent IP and evade the ban
  list and per-IP rate limit. Our proxies sit on private ranges and each appends its peer, so
  the rightmost *public* hop is the real client and a forged value (to its left) is ignored;
  it falls back to `X-Real-IP`, then the leftmost hop, then the socket peer. This also keeps
  multi-proxy deployments (an ingress in front of the frontend Nginx) from recording an
  internal `172.x` hop as the client IP.
- **Redacted personal data that had leaked into tracked files** — a real name, phone,
  address, and live Kapruka tracking codes were replaced with sample data across tests, QA
  transcripts, docs, and source comments. The LKR-parser regression fixtures keep a
  synthetic LKR-shaped code so the "tracking code is not a budget" guard still has a case to
  exercise. Working tree only; git history still contains the original values.

### Changed
- **README rewritten** around a full-system Mermaid architecture diagram (Redis + Postgres
  detailed), a Postgres ER diagram and Redis keyspace, a rubric-ordered capability table
  (voice input, occasion planning, buy-again, wishlist, consent-gated personalization), and a
  roadmap (Tanglish, deeper Sinhala/Singlish, server-side voice, multi-currency).
- `.gitignore` now excludes SIA demo-guide artifacts.

## [0.20.2] — 2026-07-04 — Occasion ticket cards, reorder receipt, reorder line totals

### Changed
- **"Upcoming occasions" now renders as boarding-pass style ticket cards** instead of a
  plain list row: a date stub (color-coded gold/pink/purple by urgency), a dashed tear
  line, an occasion-type icon (birthday, anniversary, holiday, corporate, custom), and
  the existing gold primary CTA. The section header gets a gradient bell badge and a
  divider so it reads as a section opener instead of bare text. Responsive down to
  mobile, where the ticket body stacks and the CTA goes full-width.
- **"Reorder review" now renders as a receipt** instead of a bare list: a gradient header
  badge, a "live-verified" band, one row per line with a product thumbnail, and a totalled
  footer behind a dashed tear line, matching the ticket motif above.
- **Sidebar "Recent"** now hides until a returning shopper's real order history has loaded,
  instead of showing static starter prompts in the meantime.

### Fixed
- **Reorder review showed each line's unit price instead of quantity × price**, so a 5×
  item at Rs. 390 read as "Rs. 390" instead of "Rs. 1,950", and the total was correct but
  looked inconsistent with the per-line prices above it. Both the chat reply (via a new
  `line_total`/`purchasable_total` computed server-side in `preview_reorder`) and the
  reorder-review UI now show quantity-aware line totals.

## [0.20.1] — 2026-07-04 — Search hijack, corrupted names, fake trackable orders, reorder wrong-item

### Fixed
- **A prior turn's search category could still hijack the current search when the model's
  own query came back empty or malformed.** The 0.20.0-era fix trusted any clean model `q`,
  but MiniMax intermittently emits an empty/garbled `q` (visible in logs as a "tool-artifact
  leak"), and the recovery path scanned the whole multi-turn window for a category anchor —
  so a stale "power bank" from one turn back could out-rank the current turn's "polo tshirt"
  and replay a cached result. `_apply_search_fallback` now prefers the *current* message's own
  product phrase over any prior-turn anchor when the model's query needs recovering, and
  `_recent_user_search_text` collapses each turn to one line so a multi-line textarea message
  is treated as a single unit.
- **A shopper's display name could be corrupted from a self-introduction sentence.**
  "Hi, I'm Kasun Perera. I want to send flowers" was stored and shown as "Kasun Perera. I
  Want" — name extraction ran past the sentence boundary into the next clause. It now stops
  at the first sentence-ending punctuation.
- **Tracking a payment-link reference (e.g. `ORD-20260703-K507`) instead of a real Kapruka
  order number saved it as a fake "trackable" order**, inflating the past-orders count and
  showing a fabricated status card for an order that was never paid for. A tracked order is
  now only persisted when Kapruka actually resolves a real order number — fixed in both the
  chat tool and the equivalent REST order-tracking endpoint.
- **Confirming a "buy again" reorder could add a different product than the one just
  reviewed**, if a stale wishlist/cart action was still live in the same session. The
  reorder-preview "Add to cart" button now adds the previewed order deterministically by its
  `preview_id`, bypassing the model entirely.
- **The past-orders summary could claim more "tracked orders" than were actually trackable**,
  because the count was derived from the total order-card count instead of the number that
  could actually be tracked live. The reply text now matches the same counts shown in the
  orders panel.
- **Saving the same gift recipient twice (once implicitly at checkout, once explicitly via
  "save her") created two near-duplicate address-book entries.** An explicit save now
  recognizes an existing match by phone number, or by name plus a shared city, and enriches
  that entry instead of writing a duplicate — while two different people who happen to share
  a first name are never merged.
- Confirming a reorder that fails to add anything (e.g. every line sold out) no longer reports
  success; it now tells the shopper the items are unavailable and offers to retry.

## [0.20.0] — 2026-07-03 — Sidebar shows real past orders, saved locations removed, idle results state

### Added
- **Sidebar "Recent" now shows a returning customer's own past orders** instead of static
  demo prompts. Each entry's title is condensed from its item names by the LLM on first
  view, then cached in Redis per order key (order number / order ref / customer order id)
  so repeat visits never re-call the model. Falls back to the existing heuristic
  item-title label if the AI budget is exhausted or the call fails, and to the static
  starter prompts for guests with no order history. New `GET /api/reorders/recent-summary`.
- **The idle results plane ("Results appear here") now shows a small animated sparkle**
  instead of a blank panel, so a shopper who hasn't searched yet doesn't read the canvas
  as broken. Respects `prefers-reduced-motion`.

### Removed
- **"Saved Locations" removed from the My Account sidebar section** — it opened a chat
  prompt with no backing feature.

## [0.19.4] — 2026-07-03 — Onboarding tour overlay blocked the page after Skip

### Fixed
- **Clicking "Skip" (or otherwise closing) the onboarding tour left an invisible click
  blocker over the middle of the screen**, so shoppers couldn't click any button or type in
  the chat input until they refreshed the page. `.sia-tour-popover` set
  `pointer-events: auto` unconditionally, overriding the parent `.sia-tour-root`'s
  `pointer-events: none` state, so the centered "Welcome" popover kept intercepting clicks
  even after the tour was dismissed and hidden via opacity. The popover now only gets
  `pointer-events: auto` while `.sia-tour-root` has `.is-active`.

## [0.19.3] — 2026-07-03 — Category search, cake relevance, and debug tracing

### Fixed
- **MiniMax empty search tool calls recover the shopper's product phrase instead of
  falling into the retry/tool-note state.** The v0.19.2 search-contamination fix correctly
  stopped raw multi-turn fallback queries like `"Colombo 03 confirm search for a scented
  candle"`, but it was too strict when MiniMax emitted `kapruka_search_products` with blank
  args for valid unanchored searches. Search fallback now preserves turn boundaries, strips
  stale delivery/checkout noise, recovers clean direct phrases such as `scented candle`, and
  includes Ayurvedic/herbal oil and balm anchors so flows like `Ayurvedic herbal → show all →
  Oils & balms → retry` call Kapruka with a real `q` instead of failing validation.
- **Broad category prompts (cakes, flowers, electronics, fashion, food, fruits, customized
  gifts, home/lifestyle) no longer stall on a clarifying question.** The system prompt had
  two sections in conflict — "read the situation and ask first" vs. "search immediately for
  named categories" — and several categories weren't even on the immediate-search list.
  Sia now searches first and asks about recipient/occasion/budget as a follow-up.
- **MiniMax tool-artifact leaks into a reply are now recovered instead of dead-ending the
  chat.** When the model leaks raw tool-call syntax into its final answer text, the backend
  now detects it (mirroring the frontend's existing detector) and substitutes a calm retry
  message, logging the original leaked text at `WARNING` for diagnosis. The detector
  requires either an unambiguous tool-call marker or two or more co-occurring weak signal
  words, so ordinary replies mentioning words like "Exception" no longer false-positive.
- **The "Retry" quick reply now resends the shopper's actual last query** instead of the
  literal word "Retry", which previously reached the backend with no memory of what failed.
- **Cake search no longer ranks baking tools/accessories ("Cake Turning Table") alongside
  edible cakes** unless the row also names a genuine edible-cake word (gateau, cupcake,
  fondant, etc.).
- **The upstream LLM read timeout was raised from 15s to 30s** to reduce false-positive
  `APITimeoutError`s on slower MiniMax responses; the existing 60s outer stream ceiling
  remains the real backstop.
- **An invalid `LOG_LEVEL` value no longer crashes startup** — it now falls back to `INFO`
  instead of raising inside `logging.setLevel`.

### Added
- **DEBUG-level request/response tracing across chat → LLM → MCP**, gated by a new
  `LOG_LEVEL` setting (default `INFO`). Traces the incoming/outgoing chat message, the
  LLM request/response summary (message count, stop reason, tool calls), every tool call's
  name and input, and the exact query/params sent to Kapruka search — all correlated by
  `request_id`. Checkout/recipient tool inputs (`preview_checkout`, `create_payment_link`,
  `save_recipient`, `set_gift_message`) are redacted before logging since the existing
  email/phone filter doesn't cover free-text names and addresses.

## [0.19.2] — 2026-07-02 — Browser QA follow-up

### Fixed
- **Product titles no longer show mangled HTML entities.** Some catalog rows carry a
  mojibake apostrophe (UTF-8 misread as Windows-1252) whose numeric character references
  lost their leading `&`, e.g. `Menn#226;n#8364;n#8482;s Gift Box` instead of
  `Men's Gift Box`. `clean_product_title` now decodes loose numeric entities and repairs
  the underlying mojibake before the rest of its cleanup runs.
- **Icing character counts Sia reports now match what was actually saved.** Long icing text
  is truncated to 60 chars server-side, but the LLM used to compute the character count it
  quoted back to the shopper from its own original wording instead of the truncated value.
  The `add_to_cart`/`set_icing_text` tool results now include an explicit
  `icing_text_length` field and instruct the model to use it.
- **A fresh search query no longer gets contaminated by an unrelated prior turn.** Right
  after a checkout-flow turn ("confirm", a delivery city), searching for something new could
  come back as a garbled query like `"Colombo 03 confirm search for a scented candle"` — the
  fallback used when the model's query was blank fell through to the raw, multi-turn
  concatenated text. It now only falls back to a recognized search-noise category anchor
  (unchanged, still fixes vague follow-ups like "any" → last topic); with no anchor match it
  leaves the query alone instead of guessing from stale text.
- **The "Sia is checking..." bubble now updates immediately on a stream error** instead of
  sitting unchanged until the connection finishes tearing down, which could look like a
  permanent hang on a slow upstream call.

## [0.19.1] — 2026-07-02 — Stale session history after refresh

### Fixed
- **Repeating a search after a browser refresh now shows the product/variant panel
  again.** A refresh reuses the same `session_id` (so the cart survives), but the visible
  chat always restarts empty. The LLM's conversation history for that `session_id` used to
  survive the refresh too, so a repeated question ("find me this — X" again) was answered
  from memory with no tool call and no UI block, leaving the results pane blank or stuck on
  a stale variant picker. `/api/session/start` now clears the session's message history
  (cart and checkout intent are untouched) every time it runs, matching the empty chat the
  frontend renders right after.

## [0.18.1] — 2026-07-01 — Variant QA follow-up

### Fixed
- **Selected variant confirmations use leaf SKU details.** The picker verifies the selected
  SKU before enabling add, so Comfy Crewneck `Colour 1 / Medium` shows and confirms
  `Rs. 1,990` with the clean `Comfy Crewneck T shirt Green Medium` name instead of the
  parent price or raw swatch URL.
- **Sticky checkout bar clears after cart removal.** Cart updates now push an explicit
  frontend event, and the asset version is bumped so browsers fetch the reset listener
  instead of keeping a stale cached `enhance.js`.

## [0.18.0] — 2026-07-01 — Dynamic product variant selection

### Added
- **Reusable Kapruka variant normalization.** `services/variants.py` turns
  `kapruka_get_product` `variants[]` payloads into product-agnostic option sets, including
  colour, size, style, type/subtype, fallback name parsing, and dirty colour swatches from
  hex codes or image URLs. Single `Default` variants still count as no shopper choice.
- **Variant picker UI blocks and card pickers.** Product cards preflight
  `/api/product/{product_id}/variants` before add, render an inline picker for real
  choices, disable out-of-stock combinations, show swatches where possible, and submit the
  selected variant SKU. Chat tool results can emit the same `variant_picker` block.

### Changed
- **Cart adds now guard unresolved parent products.** `/api/cart/add` accepts optional
  `variant_sku`; when a parent/base product still has selectable variants, the backend
  returns HTTP 409 with normalized options instead of adding the parent. Selected leaf SKUs
  add as the effective `product_id`, so cart rows show the clean title, stock, price, and
  image returned by `kapruka_get_product` for that SKU.
- **Sia's tool-use rules require variant confirmation.** Before adding products with
  selectable variants, Sia must show options, confirm the shopper's choice, and add the
  selected variant SKU rather than the base product.

## [0.17.0] — 2026-06-30 — Communication style recall, occasion planner, admin analytics

### Added
- **Communication style recall.** Sia honors a per-shopper tone — friendly, concise, or
  professional — via a new `set_communication_style` tool. The choice persists on the customer
  profile and is replayed into each turn's private-context preamble (never the cached system
  prompt), so the language/personality cache stays intact.
- **Occasion planner.** A new `build_occasion_plan` tool assembles a complete, budget-aware
  multi-category gift plan (cake, flowers, chocolates, hampers, keepsakes, …) from live Kapruka
  searches across 13 occasions, renders as a grouped plan block with an "Add whole plan" action,
  and degrades honestly when a category has no in-budget match.
- **Admin analytics dashboard.** A new Analytics tab and `/api/admin/analytics` endpoint surface
  the search → product-view → cart → checkout → payment funnel, top and zero-result searches,
  search latency (average + p95), and per-tool MCP health (calls, success/fail, average latency,
  last error) from the durable personalization event stream.
- **Backend-authoritative product value badges.** Normalized products now expose
  deterministic `reasonTags` plus a short `pickReason` for the highlighted recommendation.
  Badges are derived from catalog/search facts already available to the backend — rank,
  stock, active budget, explicit fast-delivery wording, lowest in-stock price, and upper
  price band — so the browser no longer has to invent product reasons from regexes.
- **Spotlight sidebar account shortcuts are documented.** The sidebar now includes My
  Account actions for past orders, saved locations, addresses/recipients, wishlist, and
  occasions, plus the existing Recent shortcuts and product tour replay.

### Changed
- **Product cards prefer backend reasons.** `renderProductReasons()` consumes
  `product.reasonTags` when present and keeps the legacy regex fallback for older payloads;
  `product.pickReason` renders as safe text under the Sia recommends card.
- **Render-only turns avoid a second model pass.** Docs now reflect the current runtime:
  Claude/MiniMax return the final answer directly after tool execution, with no optional
  second-model polish helper in the active architecture.
- **Returning cart recovery is a real welcome screen.** The saved-cart recovery UI is
  documented as a centered "Welcome back" hero with show-cart/start-fresh actions.
- **Tracking docs match the current card.** The UI shows the status pill, dates, amount,
  city, route rail, greeting message, and canonical path; raw Kapruka update lists and
  payment-method rows are not rendered separately.

### Fixed
- **Canonical Colombo district lookup is documented through recent history.** The agent
  normalizes shopper phrases like "Colombo 7" / "colombo-7" to Kapruka's zero-padded
  `Colombo 07` form before city lookup.

## [0.16.2] — 2026-06-28 — Budget enforcement for Sinhala phrasings; budget lines never add to cart

### Fixed
- **Budget is enforced for real Sinhala/Singlish phrasings, not just `under N`.** Over-budget
  items (e.g. an Rs. 8,800 saree on a "Rs. 4,000 budget") were still shown and could be added
  because `_budget_limit_from_text()` only matched `under|below|less than|yata|යට` and a
  trailing `\b` that never fires mid-word. It now also matches the `ට` particle between the
  number and the under-word ("4000 ට යටින්"), the suffixed forms `යටින්`/`අඩුවෙන්`
  (`යට`/`අඩු` as prefixes), `adu`/`yatin`, and a leading `budget N`. `normalize.products()`
  already drops over-budget rows when a ceiling is set, so populating it correctly fixes both
  the result grid and the over-budget add-guard at their single shared source.
- **A bare budget/filter line is a re-search, never a purchase.** A follow-up that only
  narrows a filter (budget, price, colour, size, quantity — e.g. "2000 ට යටින් විතරක්") could
  trigger `add_to_cart`. The system prompt (step 4) and the `add_to_cart` tool description now
  state that such a message re-runs the search and must not touch the cart.
- **Upstream timeouts raised so MiniMax-M3 turns finish instead of aborting.** Per-operation
  timeout `15s → 40s` and total `45s → 55s` (both under the 60s stream ceiling), removing the
  spurious mid-turn timeouts on slower 30–45s model responses. The timeout copy is now
  provider-neutral ("Sia is taking a little longer than usual") instead of blaming Kapruka,
  and the non-stream client abort was aligned (`30s → 55s`).
- **Reorder cards read "Order placed" instead of "Purchased."** A pay-link order is placed,
  not a confirmed purchase, so the orders-overview reorder card no longer overstates it.

### Changed
- **Prompt nudges for recipient fit and delivery reuse.** Sia is told not to recommend a
  product labelled for a different person (a "Baby"/"Men"/"Dad" item for a sister or mother),
  and to reuse the delivery city/date the shopper already gave rather than re-asking.

## [0.16.1] — 2026-06-26 — Cart totals include quoted delivery

### Fixed
- **Cart totals now include the latest quoted delivery fee everywhere the cart is shown.**
  When a shopper checked delivery and then reviewed the cart, the right-side cart review,
  chat cart snapshot, sidebar cart pill, and sticky checkout bar still displayed the item
  subtotal as "Total" (`Rs. 3,850`) even though the delivery quote was `Rs. 300`. The
  frontend now derives one shared cart display total from the current cart subtotal plus the
  latest matching delivery/checkout quote, rendering a visible Subtotal / Delivery fee /
  Total breakdown and a final `Rs. 4,150` in all cart surfaces. Checkout review totals
  remain server-authoritative.
- **Mobile sticky checkout bar stays inside the viewport.** At narrow widths the sticky bar
  now uses compact labels so the final amount and checkout controls remain readable while
  showing the delivery-inclusive total.

## [0.16.0] — 2026-06-25 — Personal sidebar: time-aware greeting, auto-hiding rail, real identity

### Added
- **Time-of-day welcome greeting.** The hero eyebrow now reads "Good morning",
  "Good afternoon", or "Good evening" based on the shopper's local clock instead
  of a fixed "Good evening", and appends their name when we know it (e.g. "Good
  morning, Nimal"). Localized across English, Sinhala, and Singlish via the new
  `partOfDay()` / `greetingEyebrow()` helpers in `frontend/app.js`.
- **Real sidebar identity row.** The user row at the foot of the sidebar shows the
  saved customer name and their delivery city instead of the hard-coded "Amaya ·
  Colombo · LKR" demo placeholder. The city is resolved server-side: the new
  `SessionStartResponse.welcome_city` field is populated from the shopper's most
  recent order (`order_history.latest_delivery_city`), so when several addresses
  have been used, the city from the **last order** wins. Falls back to a saved
  recipient's default city, then to a neutral "Guest · Sri Lanka · LKR" when the
  shopper is unknown. Rendered by `renderSideUser()`.

### Changed
- **Sidebar auto-collapses on the first action.** Sending a message (or firing a
  sidebar shortcut) now collapses the left rail to its icon strip so the results
  canvas gets the full width once a conversation is underway; "New chat" expands
  it again. Manual toggle and auto-collapse share `setSidebarCollapsed()` so the
  chevron and aria-label stay in sync however the state changed.

## [0.15.1] — 2026-06-25 — Stop the retry storm on slow MiniMax turns

### Fixed
- **Double-layered LLM retries no longer burn the timeout budget.** `create_message`
  wraps each model call in a tenacity retry loop bounded by the
  `upstream_total_timeout_seconds` (45s) wall-clock ceiling, but the underlying
  `AsyncAnthropic` client was *also* retrying internally (its default
  `max_retries=2`). On a slow provider the two layers compounded into up to ~6
  upstream HTTP attempts for a single logical call — observed in production as a
  burst of `Retrying request to /v1/messages` lines that drained the whole 45s
  budget and surfaced as an `APITimeoutError` / `upstream_unavailable` chat
  failure. Both Claude and MiniMax clients now pass `max_retries=0`, making
  tenacity the sole retry authority so attempt counts and backoff stay bounded
  and predictable.

## [0.15.0] — 2026-06-25 — "Spotlight" skin: sidebar shell + welcome hero

### Changed
- **New "Spotlight" shell.** The old top bar is replaced by a collapsible left
  sidebar holding the brand, *New chat*, Recent + Workspace shortcuts, the
  language switch, the user row + theme toggle, and the cart pill. A collapse
  control (`#sideToggle`, the `«`/`»` arrow) shrinks the rail to icons; it
  auto-collapses under 1080px. The conversation view is now chat-left /
  shopping-canvas-right, matching the Spotlight direction.
- **Welcome screen is now a centered hero.** `renderSuggestions` renders an
  eyebrow greeting, a large question, a primary search, smart-prompt pills, and
  a trust row instead of the store-strip + suggestion-grid. The hero's search
  routes through the same `send()` flow as the composer, and its copy is
  language-aware (EN / Sinhala / Singlish) — switching language on the welcome
  screen re-renders the hero in that language.
- **Spotlight palette + type.** Calmer royal-purple `#402970` / agent-violet
  `#7e47d6` / gold system, gold-gradient CTAs, softer rounded cards, a subtle
  radial aura behind the canvas, and Hanken Grotesk for display text. All in
  `sia-revamp.css`, which still loads after `styles.css` as an override layer.

### Notes
- **Backend wiring is unchanged.** No class, id, or `data-attribute` that
  `app.js` reads was renamed; chat, cart, delivery, checkout, tracking, and the
  orders overview behave exactly as in 0.14.4. The hero, sidebar toggle, and
  language re-render are the only `app.js` changes.
- **Wiring fully mapped.** Every `#id` `app.js` reads exists in the new shell.
  The two write-only nodes the old top bar lacked — `#resultsTitle` and
  `#statePill` — are now present as hidden placeholders so those writes land on
  real DOM nodes instead of `app.js`'s detached fallback elements.
- The mood-dot preview is retained in the DOM (`#moodDots`, `hidden`) for
  `app.js` compatibility but is not shown in this direction.

## [0.14.4] — 2026-06-24 — Reusable checkout details + flatter results plane

### Fixed
- **Delivery address and phone are remembered after checkout, so the next order doesn't
  re-ask for them.** Previously a completed checkout persisted the order's *line items* (for
  Buy Again) but never the *delivery contact* the shopper had just typed. A returning shopper
  whose saved recipient had only a city on file (e.g. "Nuwan Home → Colombo 03") was asked to
  re-enter the full street address and phone every time, even though they'd given them on a
  prior order. `create_payment_link` now calls `recipient_store.remember_checkout_delivery`
  after a successful order: it enriches the matching saved recipient in place (by `recipient_id`,
  then by phone, then by name) — filling in a missing phone and adding the address — or creates
  one when none matches. It is idempotent (repeat checkouts to the same person don't duplicate
  the address book) and best-effort (a memory failure can never break a created order). The next
  checkout resolves the full PII server-side from `recipient_id`, so Sia stops re-asking. +5 unit
  tests, including an end-to-end checkout → remember → reuse flow (271 pass).

### Changed
- **Past orders now show *when* they were bought.** Reorderable cards (item snapshots with no
  Kapruka tracking dates of their own) surface the purchase date from their checkout timestamp,
  and a tracked card backfills its date from the snapshot when tracking didn't carry one — so
  every order in "Your orders" shows a date instead of a bare "Saved".
- **The results pane is now a flat, floating plane.** Removed the oversized hero heading
  ("Start with a person…"), the "Live Kapruka results" eyebrow, the white results card / border /
  shadow, and the duplicate "Your orders" header + "Saved from this browser…" subtitle. The
  orders overview drops its outer card chrome too, so the only visible cards are the order rows
  themselves rather than a card-inside-a-card stack.

## [0.14.3] — 2026-06-22 — Relative-date recovery at checkout

### Fixed
- **Reordering with a saved address no longer dies at the delivery step.** After a shopper
  reordered a past order, added it to the cart, and gave a delivery address, the checkout asked
  for the delivery date and sender. When the shopper replied with a relative day, Sia called
  `kapruka_check_delivery` but sometimes omitted the date; the backend safety net
  (`_infer_delivery_date`) only recognized a literal English "tomorrow"/"today", so a misspelling
  ("tomarow"), a shorthand ("tmrw"), or the Singlish/Sinhala words the prompt itself promises to
  handle ("heta", "ada", "හෙට", "අද") all failed to resolve a date. With no date,
  `kapruka_check_delivery`'s required-field validation raised and the turn collapsed into a
  generic "I couldn't reach Kapruka cleanly just now" — the reorder never reached payment. The
  safety net now resolves all of these (plus "day after tomorrow" / "anidda" → +2 days), with word
  boundaries so unrelated words like "tomato", "Canada", or "custom" never trigger a false date.
  +2 unit tests (265 pass).

## [0.14.2] — 2026-06-22 — Reorder routing fix

### Fixed
- **"Reorder my past order" no longer falls into the trackable-only store.** The Buy Again
  button sends `Reorder my past order: <title>`, and any "buy again" / "order this again"
  phrasing carries the same intent. Because the message contained the substring "past order",
  `_is_saved_order_history_request` classified it as a request to *list saved tracked orders*
  and routed it to `list_saved_orders` (the trackable `order_history` store, which holds no line
  items). With no tracked order saved, Sia replied "I do not have saved past orders for this
  browser yet" even though a reorderable past checkout existed — so the Reorder button did
  nothing. Reorder is now detected as its own ACTION intent and handled by a direct path that
  uses the reorderable `order_items` store: it picks the named past order (substring/token match,
  falling back to the only order, or asking which one when ambiguous), re-checks live price and
  stock via `preview_reorder`, and returns a `reorder_preview` review card. The history
  classifier also now explicitly refuses reorder intent as a guard. The LIST intent ("show what
  I can buy again", "what can I reorder") is unchanged and still routes to
  `list_reorderable_orders`. +7 unit tests (263 pass).

## [0.14.1] — 2026-06-22 — Out-of-stock checkout handling

### Fixed
- **Out-of-stock at checkout now actionable, not a dead-end retry loop.** When Kapruka's
  `create_order` rejects a line because it isn't available at the requested quantity, it returns a
  plain string (`Error (product_out_of_stock): '…' is not available in the requested quantity.`)
  with no payment link. `checkout_service` previously bucketed this into the generic `no_pay_link`
  path and told the shopper "the order was prepared but no payment link came back, please retry" —
  so the agent re-confirmed the same quantity over and over, never succeeding. It now parses the
  upstream error code, returns `error="out_of_stock"` with the real reason plus a clear next step
  ("Lower the quantity or remove that item and I'll try again"), and keeps the cart intact. The
  `create_payment_link` tool passes the code and an explicit instruction to the agent (offer
  `set_quantity` / `remove_from_cart`, do not retry the same order), and the prompt documents the
  flow. Cart is never cleared on this failure. +2 unit tests (256 pass).

### Added
- **`set_quantity` and `clear_cart` agent tools.** `set_quantity` sets a line's EXACT quantity
  (fixes "make it 2" landing on 3 because `add_to_cart` increments) and enables decrements
  ("make it 1"); `clear_cart` empties the whole cart in one step. Resolves the MiniMax cart-edit
  gaps (S11 quantity, marathon silent-remove). Prompt updated to route quantity/clear intents to
  the new tools. +6 unit tests.

### Changed
- **Bounded model-call latency (B2).** Added `UPSTREAM_TOTAL_TIMEOUT_SECONDS` (default 45s),
  enforced with `asyncio.wait_for` around the whole retry sequence in `LLMProvider.create_message`,
  plus an explicit `httpx.Timeout` on the client. The per-operation `upstream_timeout_seconds` only
  bounded read time, so a slow-streaming MiniMax call could run 77s into a 60s gateway 504; the new
  ceiling fails fast with a calm error instead. `ENABLE_CHAT_STREAMING` confirmed on in compose.
- **Prompt hardening (provider portability):** re-detect the shopper's language every turn (don't
  latch to Singlish/Sinhala when they switch back to English); never assert a specific holiday/
  occasion calendar date that isn't given by the shopper or a tool; verify cart edits from the
  current turn's tool result before claiming them done.

### Fixed
- **Provider-portable price-filter validation.** `validate_tool_input` now coerces numeric-like
  string price filters (`"3000"`, `"Rs. 3000"`, `"3,000"`) to numbers and drops unusable/negative
  values instead of raising a tool error. MiniMax-M3 serializes `min_price`/`max_price` as strings,
  which previously failed every budget search (`ValueError: Invalid max_price`), forcing retry
  iterations and breaking add-to-cart on tight-budget conversations. Claude (numeric args) is
  unaffected. Added 3 unit tests; backend suite now 135 pass.

### QA
- **Full MiniMax regression/behavioral evaluation** (15-scenario battery + 26-turn long
  conversation, live against `MiniMax-M3`). Report:
  [`docs/qa/MINIMAX_QA_REPORT_2026-06-15.md`](docs/qa/MINIMAX_QA_REPORT_2026-06-15.md); MiniMax
  transcripts in `docs/qa/transcripts-minimax/`; long-conversation harness `docs/qa/long_convo.py`.
  MiniMax verified hallucination-resistant, multilingual, ~10× cheaper, and completing the full
  server-gated checkout. Open items documented: latency can breach the 60s gateway on heavy turns
  (1 × 504; recommend streaming + explicit total timeout), and weaker cart-edit / per-turn-language
  discipline vs Claude.

## [0.14.0] — 2026-06-21 — Unified orders surface (track + buy again)

### Added
- **Unified `orders_overview` surface.** The two past-order concepts — *trackable* orders
  (`order_history`, saved by Kapruka order number) and *reorderable* checkouts (`order_items`
  snapshots) — are merged into one list by the new `app/services/orders_overview.py`, keyed on
  `order_ref` where the same checkout produced both. `list_saved_orders` and
  `list_reorderable_orders` now return this same block, so "show my orders" and "reorder" render
  the identical surface. Each card carries `can_track` / `can_reorder` flags.
- **Orders List prototype port (frontend).** `renderOrdersOverview` renders the merged list in the
  `Orders List` prototype style: stacked thumbnails, mini animated route, status pill, ETA, an
  expandable 6-step stepper, and inline **Track live** + **Reorder** actions per card.
- **Order Tracking prototype port (frontend).** `trackingCard` restyled to the `Order Tracking`
  prototype: dark hero with status kicker/ETA/LIVE chip, moving-token route, stats grid, gift-card
  greeting, and confetti on delivered.

### Changed
- **Tool-routing guards (prompt).** Prompt §8 + Buy Again now forbid crossing tracking ↔ reorder
  ("reorder" never calls `list_saved_orders`/`track_order_number`, and vice versa). "Use the
  details from my past order" at checkout routes to saved recipients (`list_recipients`) — a
  tracked order does not expose a reusable address.
- `infer_sia_mood` now treats `orders_overview` / `buy_again` as `relieved`.

### Fixed
- **`Amount [object Object]` in the orders list.** `tracking_status_snapshot` stored Kapruka's raw
  `amount` (sometimes a nested object) verbatim; it now normalizes to a display string via
  `_normalize_amount`, and additionally persists `normalized_step` + delivery `city` so saved cards
  are meaningful without a second live lookup.

## [0.13.1] — 2026-06-21 — Multi-item search review & hardening

### Added
- **Dynamic, multi-signal coordination.** Companion items ("matching trousers") are now ranked
  against a `BundleContext` aggregated from the sibling items — colour still leads (neutral-first,
  tonal echoes), with category, formality register (casual t-shirts → casual trousers; dress shirt
  → formal trousers), and shared material as secondary signals. New `product_attributes` /
  `build_bundle_context` / `BundleContext` in `normalize`.
- **Centralised ranking config.** `RankingConfig` dataclass holds every relevance/coordination
  weight (was magic numbers) so behaviour is testable and tunable in one place.
- **Stronger per-item relevance.** `score_product_row` now also rewards a tight all-tokens-in-title
  match, penalises wrong category family and out-of-stock rows, and (when `max_price` is given)
  prefers in-budget items. `Product` carries `categoryName`/`inStock`; the clothing family taxonomy
  was expanded to include bottoms and outfit accessories (trousers/pants/chino/jeans/tie/belt/…).
- **Group intent metadata.** `product_groups` groups now carry `intent` (`companion`|`independent`)
  and a `thin` flag; the frontend shows a "Matching" chip on companion sections and a thin-result
  hint that still shows what came back.
- Sharper multi-item prompt guidance (bundle/set/outfit/hamper phrasing, one clean catalog query
  per item, budget → `max_price`, no glue words in `q`).
- `docs/MULTI_ITEM_SEARCH_REVIEW.md` review report; +18 unit tests.
- **Standalone SIA landing service.** New `landing/` Nginx container (`kapruka-sia-landing:0.13.1`,
  local port `4175`) serving a static concierge landing page. The Caddy HTTPS overlay now routes the
  base sslip host to the landing page, `chat.*` to the shopper UI, and `admin.*` to admin; backend
  CORS and `docs/DOCKER.md` updated for the new `CADDY_PUBLIC_HOST` / subdomain topology.

### Fixed
- **Plain multi-item sets no longer mis-tagged "matching".** Companion detection triggered on
  incidental colours in sibling result titles, so a desk set / grocery list had every bare item
  flagged as a coordinating item. It now requires an explicit "matching" cue or an explicit sibling
  query colour.
- **Catalog-hint pollution of formal-wear searches.** `normalize_catalog_query` matched romanized
  triggers as mid-word substrings, so the flower trigger `"mal"` fired inside "for-mal" and appended
  "flowers" to every formal search. Now whole-word or length-guarded prefix matching.
- Coordination anchors no longer polluted by an incidental colour in a mis-returned sibling result.

## [0.13.0] — 2026-06-21 — Multi-item search with per-item ranking & cross-item coordination

### Added
- **Multi-item search.** One request that names several items — "a blue tshirt and a red
  tshirt and matching trousers", "chocolates, a card, and flowers" — now returns a result
  section per item instead of collapsing to the last search. The model issues one focused
  `kapruka_search_products` per item; the orchestrator accumulates them as **ordered groups**
  (request order) rather than overwriting, and emits a new `product_groups` UI block. A
  single-item search still renders the original flat `products` block, so nothing regresses.
  Scales to ~10 items; the searches in a turn now run **concurrently** (`_prefetch_searches`
  + `asyncio.gather`) so a multi-item turn stays close to single-search latency instead of
  paying N serial MCP round trips. `MAX_TOOL_CALLS_PER_TURN` raised 12 → 16.
- **Per-item relevance ranking.** `normalize.products` now scores and orders each result set
  by query relevance (`score_product_row`): title hits outweigh summary/category hits, and a
  matched attribute (colour/size/material) is boosted hardest, so "blue tshirt" leads with
  blue shirts and "red tshirt" with red ones even though Kapruka returns the same rows for
  both. Single-item search benefits from the better default ordering too.
- **Cross-item colour coordination (any product type).** A companion item — flagged by a
  wording cue ("matching", "to go with", "coordinating", "pairs with") **or** named without
  its own colour while a sibling has one — is re-ranked so coordinating options lead: neutral/
  versatile colours first (they go with anything), then tonal echoes of the sibling items'
  colours (`normalize.coordination_score`, `_coordinate_groups`). It reads only colours, never
  categories, so it works for a matching bag, scarf, cushion, phone case, or trousers alike.
  An item that specifies its own colour is left as its per-item ranking ordered it.
- **Frontend grouped renderer.** New stacked, labelled `product_groups` view in the left pane:
  one ranked section per item in request order, each with a "Sia's pick" lead card, per-item
  count, and a "Show all N" inline expand. Mood/state/quick-reply helpers recognise the new
  block; the flattened union still backs cart-add and the local "only red" filter. +CSS.
- Prompt guidance for multi-item requests (one focused search per item, in order, with the
  app surfacing colour-coordinated options for "matching" items). +12 unit tests.

## [0.12.1] — 2026-06-18 — Returning-customer reliability

### Added
- True two-visit returning-customer audit with persisted browser identity, 63 live checks,
  transcript evidence, desktop/mobile screenshots, privacy controls, and ownership isolation.
- Deterministic returning-customer responses for exact name, budget, language, memory, saved
  recipients, occasions, wishlist, recently viewed products, Buy Again, and past-order requests.
- Encrypted, short-lived checkout detail snapshots so a reviewed saved-recipient checkout can be
  confirmed without exposing phone or address data to the model.

### Fixed
- Returning-customer feature requests now emit their structured UI cards instead of relying on the
  model to choose the corresponding tool.
- Broad past-order requests include both tracked Kapruka orders and checkout item snapshots.
- Saved-recipient checkout no longer collides with the recipient-list intent and can proceed from
  nickname through review and explicit payment-link confirmation without retyping PII.
- Exact saved-name and budget recall no longer varies with provider output or adds clarification
  turns.

### QA
- Backend: 233 passed; admin backend: 11 passed; frontend/admin frontend helper suites passed.
- Returning-customer Docker audit: 63 passed, 0 failed.
- Docker health/readiness and desktop/mobile browser checks passed with no console errors.

## [0.12.0] — 2026-06-17 — Sia UI prototype uplift

### Added
- Mood-aware shopper shell based on the supplied Sia prototypes: mood dots, persistent dark mode,
  language controls, listening mood, and browser voice-input fallback.
- Prototype-inspired product cards with gold `Sia's pick` treatment and compact reason chips.
- Route-style order tracking rail and saved-order progress bars derived from existing tracking data.
- Frontend guard that replaces leaked provider tool-call artifacts with a shopper-safe retry state.
- `docs/SIA_UI_PROTOTYPE_UPLIFT.md` with reviewed design files, implemented ideas, skipped work,
  testing, and known limitations.

### Changed
- Refined chat bubbles, Sia avatar styling, quick replies, composer, cart, gift-message,
  checkout-review, payment, empty, and retry states around the Kapruka violet/gold prototype
  direction while preserving current backend API contracts.

## [0.11.0] — 2026-06-15 — MiniMax provider integration + Sia trilingual & hardening

### Added
- Provider-selectable LLM layer with `LLM_PROVIDER=claude|minimax`. Claude remains the
  default and existing `ANTHROPIC_API_KEY` deployments continue to work; `CLAUDE_API_KEY`
  is now the preferred Claude key name.
- MiniMax support through MiniMax's Anthropic-compatible Messages API using
  `MINIMAX_API_KEY`, `MINIMAX_MODEL`, and `MINIMAX_BASE_URL`.
- Provider-aware usage persistence with response time, provider/model rollups, sanitized
  errors, and MiniMax/Claude key-status reporting without exposing secrets.
- Admin provider management: active provider setting, provider status cards, usage by
  provider/model, provider-aware usage logs, and provider metrics/status APIs.
- **Sia speaks the shopper's language.** New "Language — match the shopper" section in the
  system prompt: detect English / Sinhala (script) / Singlish (romanized) each turn and reply
  in the same one, mirror rather than upgrade Singlish to script, keep product
  names/prices/IDs/cities/dates in English, and translate product terms to English before
  searching the English-indexed catalog.

### Changed
- The existing MCP/local-tool orchestration, cart, checkout preview, payment-link gating,
  saved-order tracking, and streaming flow are reused for both providers.
- Claude and MiniMax both use Anthropic-compatible prompt-cache fields for the stable
  tools/system prefix and conversation tail.
- Prompt audit improvements: multilingual English/Sinhala/Singlish guidance, stricter
  product-match honesty, clearer live-tool-use requirements, safer tool argument rules,
  and better recommendation criteria for shopping and gifts.
- Docker, setup, deployment, backend, and architecture docs now describe provider
  selection and MiniMax configuration.
- Consolidated and hardened the system prompt's anti-hallucination rules into one explicit
  contract (prices, stock, delivery dates/fees, ids, promotions, discount codes, payment
  status, order status/numbers come only from a tool result this turn) plus an explicit
  tool-failure rule (never fabricate when a tool fails/returns nothing) and a "no invented
  discount codes" rule.
- Added a scope rule: Sia can track and explain orders but cannot process refunds,
  cancellations, or payment disputes — it points shoppers to the Contact/Help section on
  kapruka.com instead of promising an outcome or inventing specific support phone numbers or
  email addresses.
- Local Docker Compose now runs `CLAUDE_MODEL=claude-sonnet-4-6` (was `claude-haiku-4-5`) so
  the demo build matches production quality, notably for Sinhala/Singlish generation.

### Fixed
- Guided-gift flow no longer mistakes an occasion phrase for the recipient. "it's for a
  wedding" previously set `recipient = "a wedding"` and skipped the "who is it for?" step;
  occasion words and bare articles are now excluded from recipient extraction.
- **Guided-gift wizard no longer hijacks context-rich gift briefs.** Live QA showed it
  intercepted requests like "birthday gift for my best friend under 3000" and "present for
  mother, she likes gardening", then discarded the recipient/interest/budget and dead-ended on
  tight budgets (its bundle is flowers+cake+chocolate only). `_should_start_guided_flow` now
  routes recipient interest cues ("she loves…", "into…") and complete occasion+budget briefs to
  the Claude search loop, which uses that context directly. The wizard still serves genuinely
  bare "help me build a gift" asks. (+2 unit tests; 127 backend tests pass.)
- **Checkout confirmation gating.** A request to *see/show/review* the order summary is no
  longer treated as a go-ahead to pay. The system prompt now requires an unambiguous pay
  confirmation ("confirm", "yes pay", "place the order") before `create_payment_link`; "show me
  the summary" re-shows the review instead of generating a payment link.

### Security
- API keys remain environment-only and are reported to admins only as `configured` or
  `missing`.
- Error redaction now recognizes Claude-style and MiniMax-style secret prefixes.

### QA
- Added a reusable conversation regression suite (`docs/qa/REGRESSION_SUITE.md`, 100+
  scenarios), an executable live harness (`docs/qa/harness.py`) that drives the Docker
  `/api/chat` and records transcripts, and a full QA cycle report
  (`docs/qa/SIA_QA_REPORT_2026-06-15.md`) covering prompts, hallucination, human-likeness,
  cart ops, end-to-end payment links, and a Docker log review.

## [0.10.0] — 2026-06-14 — Streaming chat responses

### Added
- `POST /api/chat/stream` streams the final assistant answer with SSE events after the
  existing Claude tool-use loop finishes tool execution. Events include `start`,
  `tool_status`, `delta`, `blocks`, `done`, and `error`.
- The shopper frontend now uses fetch streaming when `SIA_CHAT_STREAMING=true`, updating
  the active bot bubble with `delta` text and attaching product/cart/tracking/checkout
  blocks only after the `blocks` event arrives.
- Docker/local streaming flags: `ENABLE_CHAT_STREAMING`, `CHAT_STREAM_TIMEOUT_SECONDS`,
  and `SIA_CHAT_STREAMING`. The included frontend Nginx disables buffering on
  `/api/chat/stream` so chunks reach the browser immediately.

### Changed
- `/api/chat` remains unchanged as the non-streaming fallback and rollback path.
- Docker image tags and runtime app versions are bumped to `0.10.0`.

## [0.9.3] — 2026-06-14 — Single-VM production hardening

### Added
- `docker-compose.prod.yml` production override for a single VM reached over
  `http://<PUBLIC_IP>:4173` (no DNS/TLS): overrides the CORS allow-list to the VM IP,
  switches `CLAUDE_MODEL` to a production model, replaces the dev admin credentials with a
  real username + PBKDF2 hash + session secret, and adds per-service memory/CPU limits and
  json-file log rotation.
- Root `.env` (git-ignored) for `docker compose` interpolation of deploy-specific values
  (`PUBLIC_IP`, `CLAUDE_MODEL`, `ADMIN_USERNAME`, `ADMIN_SESSION_SECRET`).

### Changed
- Docker image tags and runtime app versions are bumped to `0.9.3`.
- Reconciled stale version strings in `admin-backend/.env.example`,
  `admin-frontend/.env.example`, and `admin-frontend/admin.js` to the current version.

## [0.9.2] — 2026-06-12 — Admin table pagination and drill-in

### Added
- Admin customer, saved-order, and AI usage-log tables now load 20 rows at a time with
  next/previous pagination instead of pulling growing datasets all at once.
- Admin table headers now support allow-listed server-side sorting for customer metadata,
  saved orders, and AI usage logs.
- Customer/session drill-in now supports clicking customer IDs from tables and clicking
  recent session IDs in customer detail to reload the owning customer detail.

### Changed
- Customer detail now limits recent sessions and saved orders to the latest 20 rows to keep
  large customer histories fast and predictable.
- Docker image tags and runtime app versions are bumped to `0.9.2`.

## [0.9.1] — 2026-06-12 — Admin dashboard and session hardening

This release brings the separate admin services from the pre-`0.9.0` branch onto main
without dropping the situational-persona rebalance. It also adds session ownership checks
and proxy-aware client IP handling for the local Docker/Nginx topology.

### Added
- Separate `admin-backend/` FastAPI app with env-configured admin login, Redis-backed
  admin sessions, read-only Postgres customer/order queries, Redis key-prefix counts,
  shopper backend health/readiness diagnostics, and AI usage/settings APIs.
- Separate `admin-frontend/` static Nginx dashboard on port `4174`, with login, overview,
  customers, customer detail, saved orders, runtime diagnostics, and AI usage/cost views.
- Durable Anthropic usage tracking in Postgres for each Claude request, including
  input/output/cache tokens, estimated USD cost, status, model, session/customer context
  when available, and sanitized error details for failures or budget blocks.
- Editable admin AI budget settings, per-request output token limits, warning/critical
  thresholds, and an emergency AI disable switch.
- Admin customer views now show the latest recorded session IP and per-session IPs, derived
  from the same proxy-aware address used by throttling.

### Changed
- Shopper session-scoped APIs now authorize the submitted `session_id` against the
  browser's HttpOnly identity cookie before cart, checkout, reset, and recent-order
  operations can read or mutate state.
- `/api/session/start` no longer reassigns an existing session id to a different browser
  identity; a collision/takeover attempt receives a fresh server-generated session id.
- Backend/admin throttling keys use the forwarded client IP instead of the proxy container
  address, and Uvicorn containers trust the included Nginx proxy headers in Compose.
- The shopper backend now uses persisted AI budget settings before Anthropic calls and the
  configured per-request max output token limit instead of a hardcoded `1024`.
- Docker Compose now runs shopper frontend/backend, admin frontend/backend, Redis, and
  Postgres as the local deployment surface.

### Security
- Admin APIs are cookie-authenticated with HttpOnly `SameSite=Strict` sessions and return
  sanitized metadata only. Raw identity hashes, cookies, secrets, recipient phones,
  addresses, and raw tracking payloads are not exposed.
- API keys and secrets are never returned by AI admin APIs; key state is exposed only as
  `configured` or `missing`. Usage logs omit prompts/raw messages and sanitize session ids
  and errors.
- Admin login throttling now keys attempts by forwarded customer IP plus username.

## [0.9.0] — 2026-06-11 — Situational persona + everyday-shopper rebalance

Sia gains richer "thinking patterns" and is rebalanced so the everyday shopper buying for
themselves is the primary user, with gifting as one important mode. Trilingual
EN/Sinhala/Singlish support is scoped as the next step and is intentionally **not** in this
release; the prompt-cache fix below leaves a clean seam for the per-turn language hint.

### Added
- **"Read the situation, then have an opinion" persona** (`backend/app/agent/prompts.py`).
  Sia now reads the emotional/practical context before searching, recommends a specific pick
  with a one-line reason instead of handing over a menu, and offers one genuinely useful idea
  when the moment has weight (e.g. hand-deliver the breakup flowers yourself and add a note
  card). Anchored by concrete worked exemplars, which is the highest-leverage lever on the
  Haiku default model.
- **"Three hats, one voice" framing** — Concierge (talks, decides), Shopper (catalog search),
  Logistics (delivery/checkout) — giving the multi-agent *feel* on the existing single tool
  loop with no extra model calls or latency.
- **"You serve everyday shoppers too" section** — groceries, electronics, fashion, home and
  daily essentials get the same care as gifts, with the opinion shifting to value, stock,
  fit, and trust rather than occasion sentiment.

### Fixed
- **Prompt cache no longer busts for returning shoppers.** The saved customer name used to be
  concatenated into the cached system block (`orchestrator.py`), making the cached prefix
  per-shopper. It now rides a per-turn context line on the user message via
  `_turn_context_preamble`, so `system_prompt()` is a single shared cached prefix again.
- **Self-shoppers are no longer funneled into the gift wizard.** `guided_gifts` now bails when
  a message names a concrete everyday product (electronics/fashion/home/daily essentials) or
  is "for myself", so those turns reach the real Claude search loop instead of the
  flowers/cake/chocolate bundle flow. Short-word guards are word-boundary matched so "iron"
  and "tea" do not false-trigger inside "environment" / "instead".

### Changed
- Frontend starter prompts and store strip now lead with everyday self-shopping (earphones,
  groceries, electronics) while keeping a gift prompt and the Sinhala/Tanglish chip.
- Version bumped to `0.9.0`. Reconciled `backend/pyproject.toml` and the frontend entrypoint
  default — both left at `0.7.13` during the 0.8.0 release — so every versioned file matches
  again and `scripts/bump_version.py` works cleanly for the next release.

## [0.8.0] — 2026-06-09 — Kapruka brand re-skin of the UI

The single static UI is re-skinned to Kapruka's verified brand identity. No new UI,
no new endpoints, no new containers — the architecture is unchanged; only the look
changes. (An experimental React/Vite frontend was explored and then removed in favour
of re-theming the existing, working UI.)

### Changed
- **Brand re-skin of `frontend/styles.css`.** The off-brand blue primary (`#1267d9`) is
  replaced by Kapruka's deep royal **purple `#402970`**, the primary CTA (send button)
  becomes the signature **gold `#F8DA08`** (dark ink on gold for legibility), neutrals are
  warmed to a purple tint, and the background gradient shifts from a blue/cyan wash to a
  calm purple + gold wash. Typography is unchanged — the UI already uses **Inter**.
  All ~30 brand touch-points re-skin together (logo accent, eyebrows, chips, cart pill,
  tracking rails, focus rings, hovers).
- Version bumped to `0.8.0` across compose image tags, the backend FastAPI app, and the UI.

### Removed
- The experimental `frontend-redesign/` React app and its Docker service (port `4174`),
  the `app/api/data.py` read endpoints and the additive `ChatResponse.trace` field that
  existed only to feed it, and the related design docs. The backend is back to its lean,
  proven surface; the original UI on **`4173`** is the one and only frontend.

## [0.7.13] — 2026-06-07 — Remove hardcoded family-member (amma) bias

Several places singled out one family member ("amma" / mother), which biased
both Sia's guidance and the gift-search relevance toward mothers over fathers,
siblings, partners, friends, or anyone else. All of these are now
relationship-neutral. Recognition of the Sinhala term "amma" is **kept** — the
fix levels everyone *up* to the same treatment, it does not drop Sinhala
support.

### Fixed
- **Search relevance was biased toward mothers.** `_RECIPIENT_WORDS` in
  `normalize.py` is the set of recipient words stripped from a query before
  product-title matching, so "flowers for {recipient}" matches flowers
  regardless of who it's for. It only contained "amma"/"ammi"/"mother" (plus a
  couple of others), so "flowers for thatha / akka / malli / a friend" left the
  relationship token in the query and wrongly filtered out poetically-titled
  flower products — gift search literally worked better for mothers than for
  fathers or siblings. The list is now broad and relationship-balanced across
  English and Sinhala terms.
- **The guided-gift warm line was mother-only.** `_occasion_phrase()` returned a
  personalized Sinhala birthday line *only* when the recipient was "amma";
  everyone else got the generic English fallback. The warm line now applies to
  any recipient the shopper names.

### Changed
- **System prompt** no longer tells Sia to tune warmth "for gifts to amma /
  family"; it now says "family and loved ones" and explicitly instructs her to
  take the recipient as described (parent, partner, sibling, friend, colleague,
  …) without assuming or defaulting to any relationship.
- **Frontend** starter prompt and quick reply "Flowers for amma" → "Flowers for
  someone special"; the guided recipient suggestion chips "Amma / Friend /
  Colleague" → "Family / Friend / Colleague".

### Tests
- New `test_recipient_recognition_is_relationship_balanced` (16 relationships
  all keep the flower product) and `test_occasion_phrase_is_recipient_neutral`.

## [0.7.12] — 2026-06-07 — Warm, helpful-first onboarding greeting

### Fixed
- **The opening greeting was cold and transactional.** On load / new chat the
  first thing a shopper saw was "Welcome back, {name}. Tell me who this is
  for, the budget, and where it should go." (or the newcomer equivalent) — a
  hardcoded frontend line that demanded recipient + budget + city all at once,
  directly contradicting Sia's persona in the backend system prompt ("You are
  NOT just a search box… greet people, make small talk briefly"). Sia now
  greets warmly and offers to help first (find a gift, treat yourself, track an
  order), then guides the buy flow one focused question at a time only once the
  shopper says what they want. Returning shoppers are still greeted by name
  ("Welcome back, NAME"); newcomers get a neutral warm intro.

### Changed
- Extracted the opening line into a pure `welcomeGreeting(name)` helper in
  `frontend/app.js` (kept in sync with `system_prompt()` in
  `backend/app/agent/prompts.py`), with a jsdom regression test pinning that
  the greeting introduces Sia, offers help first, ends with an open question,
  and never re-introduces the old "who this is for / budget / where it should
  go" demands.

## [0.7.11] — 2026-06-07 — Review hardening: PII-safe order endpoints, tracking-card title fix, prompt cleanup

This release lands the action plan from the post-0.7.10 engineering review of
PRs #1, #3, #4, #5.

### Security
- **Order-tracking REST endpoints no longer leak recipient PII.**
  `GET /api/order/{order_number}` and `GET /api/order/recent` previously
  returned the *raw* Kapruka payload (`{"ok": true, "order": <raw>}`), which
  carries the recipient's **phone number and full street address**.
  `/api/order/{order_number}` is an unauthenticated lookup by order number,
  so anyone who knew or guessed an order id could read that PII. Both
  endpoints now return the same de-PII'd `tracking_timeline` envelope the
  chat path renders (status, canonical delivery path, first name + city,
  amount) — phone, street address, and full name never cross the boundary.
- **Full recipient name dropped from the tracking envelope.** The
  `recipient_hint` block carried the recipient's full name (`name`), which
  rode into the chat-rendered block and the public order endpoints even
  though nothing consumed it. It now carries only `first_name` + `city`.
  Added a regression test asserting phone / address / full name never
  appear in either endpoint's response.

### Fixed
- **Tracking card titled "Delivered to X" for orders that were not
  delivered.** The left-pane card always read "Delivered to {first name}"
  whenever a recipient name was present — including cancelled and
  in-transit orders, directly contradicting the status pill ("Cancelled",
  "Shipped") beside it. The title is now status-aware: "Delivered to X"
  only when the order is actually delivered, otherwise "Order for X".
- **Contradictory greeting-card instructions to the model.** The system
  prompt's hard rule told Sia to use the `greeting_message` "EXACTLY",
  while the per-tool-result `assistant_instruction` (added in 0.7.10) told
  her *not* to quote it. The system prompt now agrees with the tool
  instruction: mention that a card message is attached, never re-quote it.

### Changed
- De-duplicated the model-facing tracking summary. The local-track tool and
  the orchestrator's raw-MCP path now both build the model's view from one
  shared `build_model_tracking_payload()` helper, so the recipient-name
  guard and greeting-card rule can never drift apart. The greeting text is
  collapsed to a `has_card_message` flag instead of being sent verbatim and
  then forbidden.
- Dropped a redundant second `checkout_intents.invalidate()` call in
  `CartStore.add()`.

### Added
- `backend/scripts/bump_version.py` — single-command version bump across all
  ~11 files that duplicate the version string, replacing the per-release
  hand-editing that bloated every prior diff.

## [0.7.10] — 2026-06-07 — Drop intrusive tracking auto-restore on page load

### Fixed
- **Tracking card auto-restored on every page load.** The 0.7.9 release
  added an `init()` step that called `/api/order/recent` whenever a
  shopper had a saved trackable order, then re-rendered the full
  delivery-path card and dropped a "Welcome back. I reloaded the
  last tracking lookup for **VPAY827982BA** (Delivered)…" line into
  the chat thread. A shopper who only refreshed between sessions —
  or who had nothing to do with tracking — saw the tracking card
  pop up uninvited and the chat thread gain a second "welcome"
  bubble that looked like Sia was confused about who was talking to
  her.

  The auto-restore is now gone. The tracking card only appears when
  the shopper explicitly asks "track my order" or pastes a Kapruka
  order number — exactly the same chat-driven path the LLM has
  always used. The `init()` flow now only restores the topbar cart
  pill, which has never had the same "why is this here?" problem
  because it is a small persistent indicator, not a full left-pane
  card plus a chat bubble.

### Removed
- `Api.recentOrder()` — the page-init endpoint wrapper that fired
  the auto-restore.
- `tracking_timeline_envelope()` and its now-unused sub-helpers
  (`normalizeStep`, `stepLabel`, `toneForStep`, `canonicalPath`,
  `normalizeAmount`, and the `CLIENT_STEP_*` alias maps) — all of
  which only existed to feed the auto-restore path.
- The BUG-1 envelope frontend tests that exercised the removed
  helpers (the BUG-2 date / time / fallback tests are kept; the
  `trackingCard` smoke test continues to cover BUG 4 + BUG 6).

### Changed
- App / image / pyproject / Docker compose version bumped to
  `0.7.10`.
- `init()` keeps the cart-restore pattern but drops the tracking
  rehydration step. The chat thread always starts clean.

## [0.7.9] — 2026-06-07 — Tracking UI polish: auto-restore, local time, clean money/date

### Fixed
- **Tracking UI not loading on refresh.** The shopper refreshes the
  page after tracking an order and the left pane resets to the welcome
  state. The tracking card now auto-restores on init by calling
  `/api/order/recent` and converting the raw Kapruka payload into the
  same `tracking_timeline` envelope the chat tool path produces
  server-side. A small "Welcome back" message in chat explains the
  restore so the card is never mistaken for a hallucination.
- **Inconsistent timestamps.** "Thu Sep 22 03:39:35 EDT 2022" / "25 Sep
  2022 15:00:00 GMT" / "25 / SEPTEMBER / 2022" are now all rendered
  in the user's local timezone through one formatter, with a smart
  parser that understands every shape Kapruka returns and falls back
  to the raw string when it can't parse a value.
- **Raw amount JSON.** `{"value": "13780", "currency": "LKR"}` is now
  normalised server-side and client-side to a clean `Rs. 13,780`.
  The opaque `payment_method` code (e.g. "5306") is dropped from the
  envelope so the meta grid never shows two confusing money rows.
- **Weird `*"` greeting in chat.** The chat reply used to wrap the
  Kapruka greeting message in `*"…"` markdown that broke rendering.
  Both the local-track tool and the raw `kapruka_track_order`
  orchestrator path now instruct the model to mention that a card
  message was attached and let the left-pane card do the showing,
  rather than re-quoting the text.
- **Duplicate "Delivered" pill + status text.** The header used to
  read "Delivered Delivered VPAY827982BA" because the status pill
  and a separate status text node both carried the same word. The
  status text is gone — the pill holds the status, the order number
  chip is next to it, and the title carries the recipient greeting.

### Removed
- **Raw Kapruka updates list.** The collapsible "Raw Kapruka updates
  (N)" `<details>` block was redundant with the canonical six-step
  delivery rail, and the unedited Kapruka update strings made the
  card feel noisy. Shoppers who want the raw log can still see the
  full progress data via the chat tool path.

### Changed
- App / image / pyproject / Docker compose version bumped to `0.7.9`.

## [0.7.8] — 2026-06-07 — Rich delivery-path tracking UI + tracking-code parser fix

### Fixed
- **Tracking-code price leak.** A Kapruka tracking number like `VLKR00AB12CD` was
  being misread as `LKR 22` by the budget / price filters, so a shopper
  who typed a tracking number saw `Budget Rs. 22` and the guided gift
  builder thought they wanted a Rs. 22 gift. The regex now requires
  word boundaries around the currency / keyword tokens
  (`\b(?:rs\.?|lkr|under|below|budget|max|over|min|around|less than|more than)\b`),
  so the substring `LKR` inside `VLKR` no longer matches. Both the
  frontend `detectIntent` / `productFilterFromMessage` and the backend
  `_extract_budget` / `_extract_vibe` parsers are pinned by tests.
- **Recipient-name hallucination on tracking.** "Hope KAL loved it" no
  longer happens. The chat now sees a `recipient_first_name` field
  pulled from the real Kapruka recipient payload and an
  `assistant_instruction` that forbids inventing a name. When the
  payload has no recipient name, the chat is told to say "the recipient"
  / "they" instead.

### Added
- **Rich delivery-path tracking UI.** The left pane now shows the full
  canonical delivery path (`Order received → Confirmed → Being prepared →
  Shipped → Out for delivery → Delivered`) with a styled status pill,
  ordered / shipped / delivered dates, amount, payment method, city, an
  optional Kapruka-stored greeting card message, and a collapsible
  list of the raw Kapruka progress updates. Cancellation is a
  first-class state with a red tone. The follow-up panel below the
  tracking card offers "Show my cart", "Find a gift", and "Start a new
  chat" so the shopper can keep shopping without losing the canvas.
- New `summary` / `path` / `updates` / `recipient_hint` envelope on the
  `tracking_timeline` block. The legacy `status` / `steps` keys still
  work so the new UI can render older envelopes during the rollout.
- A non-PII `recipient_hint` (first name + city) is now carried in
  both the UI block and the model-side tool result, so the chat reply
  and the left-pane greeting card always agree on who the gift was for.
  Phone and street address stay on the live MCP response and are never
  echoed in the chat envelope.
- A Node-based smoke test for the frontend helpers
  (`frontend/tests/frontend_helpers.test.js`) that pins the budget /
  max-price regex guards and exercises the new `trackingCard` against
  a typical payload, including the no-recipient-name and cancellation
  cases.
- Backend regression tests in `tests/test_order_tracking.py` for the
  new tracking block shape, the canonical path, the recipient-name
  PII guard, and the non-PII tool-result content the model sees.
- Backend regression tests in `tests/test_guided_gifts.py` for the
  budget regex fix (including the `VLKR00AB12CD` regression case).

### Changed
- The orchestrator now feeds the model a non-PII tracking summary on a
  raw `kapruka_track_order` call as well as on the local
  `track_order_number` / `track_recent_order` tools. The model never
  receives the recipient phone or full address for a tracking lookup.
- App/image version bumped to `0.7.8`.

## [0.7.7] — 2026-06-07 — Gift message persists across cart adds and through checkout

### Fixed
- A shopper who says "Add gift message …" mid-shopping no longer has the
  message silently dropped from the order. The cart now carries an
  order-level `gift_message` field that survives every add, bundle add,
  remove, and preview/confirm round trip, and `create_order` falls back to
  the stored value when the agent forgets to repeat the message at
  checkout (the actual cause of the reported bug: Sia would confirm the
  message was added but the message was never persisted, so the second
  add + delivery checks + checkout review made the model lose it and the
  Kapruka order went out without it).

### Added
- New local agent tool `set_gift_message` (and an optional `gift_message`
  parameter on `add_to_cart`) so the agent can persist / edit / clear the
  order-level message.
- `CartView` now exposes `gift_message`; the cart card and the in-chat
  cart snapshot both render the message with one-click "Edit" and
  "Remove" actions. An empty cart now shows an "Add gift message" hint
  instead of being silent.
- New REST endpoint `POST /api/cart/gift-message` so non-chat clients
  (and the future cart card editor) can set or clear the message the
  same way the agent does.
- Agent system prompt now instructs Sia to persist gift messages via
  `set_gift_message` (or `add_to_cart`'s `gift_message` parameter) so
  they survive later adds and the final checkout.
- Quick replies after a cart add / bundle add now include "Add gift
  message" (or "Edit gift message" when one is set), matching the path
  Sia used in the bug report.
- Backend regression tests in `tests/test_redis_stores.py` pinning:
  - in-memory + Redis round-trip of the stored message,
  - add-to-cart convenience path (`gift_message=` parameter),
  - `clear()` resets the message,
  - editing the message after preview invalidates the active intent,
  - `create_order` falls back to the stored message and the upstream
    `kapruka_create_order` call carries it,
  - caller-supplied `gift_message` still wins over the stored value.

### Changed
- App/image version bumped to `0.7.7`.

## [0.7.6] — 2026-06-07 — Restore topbar cart pill on page refresh

### Fixed
- The topbar cart pill now restores from the server cart of record on page load.
  `frontend/app.js` `init()` now calls `GET /api/cart` with the session id from
  `sessionStorage` and pipes the result into `renderCart`, so a shopper who
  refreshes the tab still sees their saved cart in the topbar without having to
  ask "show my cart" first. The chat thread and result canvas still start clean
  on refresh by design; an empty server cart keeps the pill hidden.

### Added
- Backend regression tests pinning the `/api/cart` GET contract that the
  frontend's cart-pill restore flow relies on, covering both the in-memory
  fallback and the Redis-backed production path.

## [0.7.5] — 2026-06-07 — Saved past order listing

### Added
- Shoppers can ask "show all my past orders" in chat to see saved Kapruka order numbers
  previously tracked by the same browser/customer.
- New `list_saved_orders` local agent tool and `order_history` UI block list stored
  tracking snapshots without live-refreshing every order.
- Each saved order row offers a "Track this order" chat action, which performs a live
  `kapruka_track_order` lookup only for the selected order number.

### Changed
- App/image version bumped to `0.7.5`.

## [0.7.4] — 2026-06-07 — Guided gift discovery

### Added
- Guided gift discovery for broad requests such as "I'm looking for a gift": Sia now asks
  one question at a time for occasion, recipient, budget, and vibe.
- New `guided_gift_progress` and `bundle_recommendation` UI blocks. The left pane shows
  sticky progress dots and then a coordinated 2-3 item gift bundle.
- New `POST /api/cart/add-bundle` endpoint so the frontend can add a recommended bundle
  as one logical gift action while the backend still re-verifies each item before adding
  it to the cart.

### Changed
- App/image version bumped to `0.7.4`.

## [0.7.3] — 2026-06-07 — Saved recent order tracking

### Added
- Order tracking can now reuse the latest saved trackable Kapruka order number for the
  same browser/customer, so a returning shopper can ask to "track my order" without
  repeating the number after one successful lookup.
- Added `GET /api/order/recent?session_id=...` for API clients to track the latest saved
  order, returning `not_found` until an emailed Kapruka order number has been saved.
- Checkout payment-link creation now saves the `order_ref` only as a pending checkout
  reference; it is never used as a tracking number.

### Changed
- App/image version bumped to `0.7.3`.

### Fixed
- The agent now routes order tracking through local tools that persist successful
  order-number lookups, instead of calling `kapruka_track_order` directly and losing the
  number for future no-number tracking.

## [0.7.2] — 2026-06-06 — Returning-name display polish

### Changed
- App/image version bumped to `0.7.2`.

### Fixed
- Returning-customer welcome names now display with capitalized initials even when the
  shopper introduced themselves in lowercase, e.g. `kasun` displays as `Kasun`.

## [0.7.1] — 2026-06-06 — Sia mood signal and retry recovery

### Added
- Persistent Sia mood signal: `/api/chat` responses can include a `mood` field
  (`thinking`, `delighted`, `careful`, `puzzled`, `relieved`), and the frontend applies it
  to the header ✦, typing avatar, final bot avatar, and a compact mood chip.
- App/image version bumped to `0.7.1`.

### Fixed
- Repaired retry loops caused by stale Redis chat history containing orphaned Anthropic
  `tool_result` blocks without the required preceding `tool_use`. The backend now
  sanitizes history before Anthropic calls and logs the number of dropped invalid tool
  blocks without logging message content.

## [0.7.0] — 2026-06-05 — Redis state + cache and production Docker

### Added
- Async Redis state + cache layer (`app/services/redis_client.py`) with JSON/TTL helpers,
  a startup ping, and a safe in-memory fallback when Redis config is empty. Docker Compose
  can use split `REDIS_HOST`/`REDIS_PASSWORD` settings so passwords with special
  characters are encoded safely.
- Redis-backed implementations (with in-memory fallback) for:
  - session history: `sia:session:{session_id}:messages` (TTL 24h)
  - cart: `sia:cart:{session_id}` (TTL 24h) and product detail cache
    `sia:product:{product_id}` (TTL 30m)
  - checkout preview intents: `sia:checkout:intent:{intent_id}` and
    `sia:checkout:latest:{session_id}` (TTL 15m)
  - order idempotency: `sia:checkout:idempotency:{sha256}` (TTL 30m)
  - per-session create-order attempt counter: `sia:ratelimit:create_order:{session_id}`
    (TTL 1h)
  - daily token budget: `sia:budget:{yyyy-mm-dd}` (TTL 48h)
  - bootstrap catalog cache: `sia:bootstrap:v1` (TTL 30m)
  - MCP read cache: `sia:mcp:{tool}:{sha256(args)}` (TTL 30m)
- New `redis:7-alpine` Docker service. No host port exposed; only the backend service
  can reach it on the internal Docker network. Append-only, password-protected, with a
  named `redis-data` volume and a healthcheck using `redis-cli -a "$REDIS_PASSWORD"
  ping`.
- `/readyz` now returns a structured `components` map (`mcp`, `redis`) with
  `status`/`detail` per component, and reports `degraded` if any required dependency is
  unavailable. In `ENV=production` missing Redis config is reported as degraded on
  purpose.
- `POST /api/session/reset` now also clears the active checkout intent for the session,
  not only the chat history and cart.
- `PUBLIC_FRONTEND_ORIGIN` / `PUBLIC_API_ORIGIN` env hooks reserved for a future
  split-domain mode (the default same-domain Nginx path ignores them).
- `tests/test_redis_stores.py` covers the in-memory fallback, the Redis path (via a
  `FakeRedis` stub that matches the production `RedisLike` interface), JSON
  serialisation, key naming, and TTL application. Existing tests updated to the async
  store APIs.

### Changed
- Backend image is now `kapruka-sia-backend:0.7.0` and the FastAPI app version is
  `0.7.0`. Frontend image is `kapruka-sia-frontend:0.7.0`; `SIA_APP_VERSION` is bumped
  to `0.7.0`.
- Frontend Nginx `Content-Security-Policy` `connect-src` is now `'self'` only — the
  dev-only `http://127.0.0.1:8000` / `http://localhost:8000` entries are removed. The
  browser never talks to the backend host directly in the default same-domain topology.
- Backend CORS allowlist is now read from `CORS_ALLOW_ORIGINS`; Docker Compose defaults
  to the local frontend origins and production operators must override it with the real
  HTTPS origin. `*` is never used in production.
- Cart values stored in Redis include the product id, sanitized title, image URL /
  background string, URL, price, quantity, currency, and icing text — every field the
  UI needs to render the cart without a re-read.
- Docker runtime env files are split by service: `backend/.env` keeps backend secrets,
  `redis/.env` keeps the Redis service password, and `frontend/.env` keeps only public
  runtime values.
- `backend/.env.example` now documents only backend secrets. Ordinary production service
  settings live in `docker-compose.yml`, while `REDIS_URL` is documented only as an
  optional managed-Redis override.
- `README.md`, `backend/README.md`, `docs/DOCKER.md`, `docs/ARCHITECTURE.md`, and
  `docs/BACKEND_PLAN.md` document the Redis key map, the three-service Docker topology,
  the production env vars, the operational shape, and the backup note for `redis-data`.

### Fixed
- Fixed Redis-backed chat history serialization so Anthropic content blocks are stored as
  JSON objects, not SDK object strings. This resolves live `messages.N.content.0: Input
  should be an object` errors after session history is loaded from Redis.
- Docker Compose no longer depends on a root `.env` or a single shared backend env file;
  each service loads only the env file it needs.
- Redis healthcheck/command escaping now uses container-time `$${REDIS_PASSWORD}`
  expansion correctly, without the `${VAR:?error text}` pattern in the Compose file.
- API responses now include `X-Response-Time-Ms` and `X-Sia-Redis-Backend` headers, and
  backend logs include structured per-request timing plus MCP Redis-cache-hit messages.
- Docker Redis config now supports passwords containing `@` or other URL-special
  characters by passing password fields separately instead of interpolating a raw
  `REDIS_URL`.
- Product-card `Add` now calls `/api/cart/add` directly, appends a local Sia
  confirmation, and keeps the product result browser in place instead of depending on a
  Claude chat turn.
- Explicit cart fallback views now keep the topbar cart pill in sync even when the chat
  envelope is a retry/error response with an empty default cart snapshot.
- Static frontend assets are loaded with versioned URLs (`?v=0.7.0`) so browsers do not
  keep an old `app.js` bundle after a Docker redeploy.
- In production, carts and intents now survive a backend container restart as long as
  the `redis-data` volume is intact. The previous in-memory store lost all session/cart
  state on every container restart.

## [0.6.1] — 2026-06-05 — Browseable results and non-navigating add-to-cart

### Added
- Larger product result loading for search turns, with the backend aggregating up to three
  MCP search pages for the frontend result browser.
- Left-pane product pagination so shoppers can browse the loaded set without flooding chat.
- Local loaded-result filtering for requests such as "only red colour" or price refinements,
  using the products already loaded in the result pane.

### Changed
- Sia now receives a top-three product summary for chat while the UI receives the broader
  browseable product grid.
- Product `Add` actions update cart state and chat confirmation without switching the left
  pane away from product results.
- Cart review renders in the left pane only when the shopper explicitly asks to show/open
  the cart.
- App/image version bumped to `0.6.1`.

### Fixed
- Fixed the stale "Refreshing cart from the server" skeleton caused by "add another gift"
  and other cart-mutation phrasing that returned no UI blocks.

## [0.6.0] — 2026-06-05 — Dockerized backend and frontend apps

### Added
- Separate Docker images for the backend (`kapruka-sia-backend:0.6.0`) and frontend
  (`kapruka-sia-frontend:0.6.0`).
- `docker-compose.yml` for local two-container runs with health checks and restart policy.
- Frontend Nginx config that serves the static app and proxies `/api`, `/healthz`, and
  `/readyz` to the backend service.
- Runtime frontend config generation via `SIA_API_BASE` and `SIA_APP_VERSION`.
- `docs/DOCKER.md` with build, run, environment, health-check, and production notes.

### Changed
- Backend package metadata and FastAPI app version are now `0.6.0`.
- Frontend `app.js` now treats an intentionally empty `SIA_API_BASE` as same-origin API
  mode for the Docker/Nginx proxy.
- Root/backend/architecture docs now document Docker-first local startup and separate app
  images.

## [0.5.1] — 2026-06-05 — Result relevance, delivery consistency, and live-test docs

### Added
- Product relevance tests for the live failure shape where a "red jacket" search returned
  unrelated arrack cards.
- Delivery normalization/orchestrator tests for available, unavailable/fully-booked, and
  city-not-found responses.
- Browser-side safe debug logging for failed chat requests during development.

### Changed
- Product search blocks now pass through a query/category relevance gate before reaching the
  frontend. Broad flower, cake, grocery, and gift searches remain permissive, but concrete
  category intent such as clothing must match the returned row/category text.
- Delivery tool results sent back to Claude are now canonical summaries derived from the UI
  delivery block, so `available:false` cannot be softened into positive checkout copy.
- Frontend chat timeout increased to 30 seconds to avoid retry state during slower live MCP
  turns.
- `backend/README.md` now documents the current route set, passive `/readyz`, session reset,
  checkout preview intents, server-side price re-verification, and frontend path.

### Fixed
- Suppressed unrelated product cards when Kapruka MCP search returns loose matches that do
  not match the shopper's concrete query/category.
- Fixed delivery copy/structured-block mismatch where Sia could say "good news" while the
  `delivery_status` block showed `available:false`.
- Added safer frontend retry diagnostics without changing the shopper-facing retry message.

## [0.5.0] — 2026-06-05 — Gap review hardening and richer shopping states

### Added
- Server-side checkout preview intents. `preview_checkout` now creates an intent with
  `intent_id`, `expires_at`, and `cart_hash`; `create_payment_link` must include the latest
  valid intent after explicit shopper confirmation.
- REST `POST /api/checkout/preview` for non-chat clients and tests.
- Audit events for cart add/remove, checkout preview, confirmation, payment-link creation,
  and checkout failures.
- Separate per-session create-order attempt cap for local/dev, documented as Redis-backed
  per user/IP for production.
- MCP read cache for category/product/search/city reads plus a short circuit breaker after
  repeated Kapruka MCP failures.
- Frontend welcome store strip, checkout safety assurances, skeleton result cards, fallback
  Kapruka links, and checkout intent/expiry display.
- Backend tests for checkout intent enforcement and cart-change invalidation.
- Anthropic prompt caching on the agent turn: a `cache_control` breakpoint on the system
  block caches the stable tools+system prefix, and top-level `cache_control` caches the
  conversation tail. Verified ~32k input tokens served from cache per turn (`cache_read`).
- Per-turn token/cache logging (`tokens in/cache_write/cache_read/out`) for cost visibility.

### Changed
- Cart mutations invalidate any active checkout preview intent.
- Checkout is now backend-enforced instead of relying only on prompt instructions.
- Stacked welcome layout uses a shorter chat pane so the product canvas appears sooner.
- Security headers now include a CSP baseline for backend-served responses.
- Default `CLAUDE_MODEL` is now `claude-haiku-4-5` (low-tier for dev/testing); override with
  `CLAUDE_MODEL` for production (e.g. `claude-sonnet-4-6` / `claude-opus-4-8`).
- `config` treats empty env vars as unset (`env_ignore_empty`) so a blank exported
  `ANTHROPIC_API_KEY` can't shadow the value in `.env`.

### Fixed
- **Critical MCP connection lifecycle.** The session was opened with an `AsyncExitStack` in
  the FastAPI lifespan and used from request tasks, binding the anyio cancel scope to the
  wrong task — startup intermittently failed with `CancelledError`, and a failed connect
  left a half-open session that turned every later request into `ClosedResourceError`. The
  session is now owned by a single long-lived task (`_run`); request handlers only *use* it
  (safe cross-task), and `connect()` re-establishes it lazily.
- **Dropped-session auto-recovery.** If the transport streams close mid-flight (e.g. the
  server 429s the background writer), the client marks the session dropped and reconnects on
  the next request instead of failing permanently.
- **Connect back-off.** After a failed/refused connect, the client waits a cooldown before
  retrying so a flapping or rate-limiting upstream isn't hammered into a worse hole.
- **`/readyz` no longer storms the upstream.** It was forcing a reconnect on every poll; it
  is now a passive check of the existing session state (`is_connected`).
- **Search results vanished from the canvas while the chat described a product.** Two causes:
  (a) low-tier models (Haiku) sometimes call `kapruka_search_products` with flat args instead
  of the `{"params": {...}}` wrapper, so the call errored with "missing params object" and the
  search was dropped — the orchestrator now tolerantly wraps flat args; (b) an `empty_results`
  block rendered after a `products` block cleared the canvas — `empty_results` is now
  suppressed whenever products were found, and a non-empty result set is never overwritten by
  a later empty search.
- Removed stray `confirms` / `{status:ok}` files left by shell redirects.

## [0.4.0] — 2026-06-05 — Viewport, cart, and session reset polish

### Added
- **New chat** action that calls `POST /api/session/reset`, clears only the current
  session's conversation history and server cart, removes the browser session id, and
  rebuilds the welcome workspace.
- Compact chat cart snapshot for "Show my cart" requests, backed by `GET /api/cart` when
  the agent reply does not include a cart block.
- Product card actions for Add, Compare, and Delivery prompts.
- Cart feedback animations for count/total changes and cart row updates, with
  `prefers-reduced-motion` support.

### Changed
- Desktop viewport is now fixed to `100dvh`; chat and result panes scroll internally while
  the page itself stays still.
- Removed the top prompt/search summary pill to reduce visual noise and use more horizontal
  space.
- Product cards now use a larger image-forward treatment with price as a top-right signal,
  stock badge over the image, a warmer featured first card, and cleaner white cards after.
- Cart review now shows product thumbnails, title, quantity, unit price, line total, and a
  remove affordance.
- Chat bubbles and quick replies are smaller and denser with smoother enter transitions.
- Frontend session persistence moved out of `localStorage`; stored session ids are discarded
  on startup so refreshes begin clean.

### Fixed
- Fresh chats no longer inherit stale cart/history from older browser-local sessions.
- Chat auto-scroll now runs after DOM paint so long replies, quick replies, and cart cards
  stay visible.

## [0.3.0] — 2026-06-04 — Two-pane shopping workspace

### Added
- **Two-pane frontend**: persistent product/results canvas on the left and compact Sia chat
  on the right. Chat turns now update the result stage instead of burying every card inside
  the conversation.
- Result-stage views for starter suggestions, product grids, delivery checks, cart review,
  checkout review, payment links, tracking timelines, empty results, and retry states.
- Safe assistant-message formatting for `**bold**`, numbered lists, and bullet lists without
  allowing raw HTML injection.
- Optional OpenAI helper service (`backend/app/services/openai_helper.py`) for low-risk text
  polish. It is disabled by default and never receives checkout authority.
- Root `.gitignore` to keep `.env`, caches, virtualenvs, bytecode, and system files out of
  local commits.

### Changed
- Product title cleanup now runs in backend normalization and frontend rendering, removing
  stray backticks and ugly quote spacing while preserving valid quoted product names.
- Sia's prompt now asks for checkout/delivery details one focused question at a time and
  uses more personal gift-shopping language.
- Documentation now describes the current two-pane product, UI block routing, and
  Claude-primary/OpenAI-helper model strategy.

## [0.2.0] — 2026-06-04 — Conversational redesign

The product became a **pure chat agent**: the shopper does everything by talking to Sia —
search, cart, delivery, and payment-link creation — with no buttons to click. Product
results, cart, and the pay link render as cards *inside* the conversation.

### Added
- **Local agent tools** (`backend/app/agent/local_tools.py`) exposed to Claude alongside the
  Kapruka MCP tools: `add_to_cart`, `remove_from_cart`, `view_cart`, `create_payment_link`.
  The agent now acts on natural language ("add the first one", "show my cart", "send it to
  Nimal in Negombo on Friday").
- **Shared checkout service** (`backend/app/services/checkout_service.py`) — order creation
  with server-side price source-of-truth, delivery re-verification, and an idempotency key.
  Used by both the agent tool and the REST endpoint.
- **`GET /api/bootstrap`** — returns the real Kapruka categories (64, from
  `kapruka_list_categories`) plus colors/sizes/cities, so the frontend's typing chips map to
  the actual catalog instantly with no per-keystroke network call.
- **Personality system prompt** (`backend/app/agent/prompts.py`): Sia greets, has character,
  asks about the occasion, and does **not** auto-search on "hello". Collects order details
  conversationally and **confirms before** creating the payment link.
- Inline chat **blocks** in the chat envelope (`products` / `cart` / `pay_link`) rendered as
  cards in the thread.
- `CHANGELOG.md` (this file).

### Changed
- **Frontend rewritten as a single-column chat app** and **moved to `frontend/`** (served
  from there). No left rail, no separate panels — only the conversation.
- Typing-preview chips now cover every Kapruka category + colors/sizes/cities.
- Chat envelope: replaced `products`/`delivery`/`intent` top-level fields with ordered
  `blocks`; kept `cart` snapshot for the small header pill.
- `kapruka_create_order` is no longer exposed to the agent's MCP toolset — ordering goes
  exclusively through the gated local `create_payment_link` tool.
- New chat-focused stylesheet (`frontend/styles.css`) — fixed alignment of bubbles, product
  grid, cart/pay cards; messages and cards animate in.
- Hardened startup: MCP connect failures (incl. `CancelledError`) no longer crash the app;
  it lazy-connects on first request. `connect()` is now idempotent.
- Tightened the PII log redactor so it no longer mangles version/date numbers.

### Removed
- Intent chips inside chat bubbles.
- The "Kapruka MCP result / No results" heading and the "Ask me to check delivery" filler.
- The left sample-state rail and the demo thread/sample data.
- Old root `index.html` / `app.js` / `styles.css` (archived under `.archive/`).

### Fixed
- **"No results" bug** (e.g. searching "watches"): `kapruka_search_products` returns a plain
  string `"No products found…"` rather than a results object. It no longer shows a broken
  empty card UI — Sia explains it conversationally and suggests alternatives.

## [0.1.0] — 2026-06-04 — Initial backend + live integration

### Added
- FastAPI **backend-for-frontend** running a **Claude tool-use loop** over the live Kapruka
  MCP server (`mcp.kapruka.com/mcp`). Architecture in `docs/BACKEND_PLAN.md`.
- Discovery script + captured live tool schemas (`docs/MCP_REFERENCE.md`, `docs/MCP_TOOLS.json`).
- `/api/chat`, `/api/cart/*`, `/api/checkout`, `/api/order/{n}`, `/healthz`, `/readyz`.
- Server-side cart of record with **price re-verification** on add (`cart_store.py`).
- Normalize layer mapped to real MCP fields; security headers, CORS allowlist, rate limit,
  token-budget circuit breaker, PII-redacting structured logs.
- First version of the frontend wired to the backend (click-based; superseded in 0.2.0).

### Discovered (live MCP conventions)
- Every tool wraps arguments in a `params` object; `response_format` must be set to `json`.
- `kapruka_check_delivery` needs a **canonical** city ("Colombo 03", not "Colombo").
- `create_order` returns a pay link + `order_ref`; the trackable `order_number` is emailed
  only after payment.

### Fixed
- Empty `ANTHROPIC_API_KEY` env var was overriding the real key in `.env`
  (`env_ignore_empty=True`).

[Keep a Changelog]: https://keepachangelog.com/
