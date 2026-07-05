# SIA — AI Shopping Concierge

> A conversational shopping assistant for Kapruka. Customers simply *chat* with Sia in plain
> language, and Sia discovers products, builds gifts, manages the cart, handles checkout, and
> tracks orders — all in one calm, guided conversation.

---

## Overview

**SIA** turns online shopping into a natural conversation instead of a search-and-click chore.

- 🛍️ **AI-powered shopping assistant** — talk to Sia the way you'd talk to a helpful store concierge.
- 💬 **Conversational commerce** — discover, compare, decide, and check out, all inside one chat.
- 🎁 **Gift expert** — describe the person and occasion; Sia recommends a thoughtful bundle.
- 🧠 **Personalized & context-aware** — Sia remembers the conversation, recognises returning shoppers, and tailors suggestions.
- 🔒 **Safe by design** — all prices and payment steps are verified on the server; nothing risky happens by accident.

The product runs in a calm **"Spotlight" shell**: a collapsible left sidebar (brand, *New chat*, shortcuts, language switch, identity row, cart) with the **Sia chat** beside it and a **live shopping canvas on the right** (products, cart, delivery, checkout, payment, tracking). As you talk, the canvas updates in real time. The rail auto-collapses to icons once a conversation is underway so the canvas gets full width.

---

# Customer Application (Frontend)

The shopper-facing two-pane web app. This is where customers meet Sia.

## 1. AI Shopping Assistant

### Features
- Natural, human-like conversations in plain language
- Product discovery through chat
- Personalized product recommendations
- Context-aware responses based on the whole conversation
- Multi-turn conversations with follow-up handling
- Live "mood" and typing cues so the assistant feels responsive

### Sub-Features
- Understands what the customer actually wants (intent, not just keywords)
- Remembers the context of the current conversation
- Handles vague or ambiguous requests by asking smart follow-ups
- Explains *why* it recommends something
- Greets returning customers by name (when they've introduced themselves)

---

## 2. Product Discovery

### Features
- Search the live Kapruka catalog using natural language
- Category exploration with starter prompts and quick chips
- A broad results grid with pagination
- Product comparison
- Similar / related product suggestions

### Sub-Features
- Price-based filtering (e.g. "under 6000")
- On-the-spot local filters in the result pane (e.g. "only red colour")
- Occasion-based suggestions
- Recipient-based recommendations
- Clean, image-forward product cards (titles sanitized for a polished look)

---

## 3. Gift Assistant

### Features
- Guided gift-building flow
- Gift recommendation engine that proposes a coordinated 2–3 item bundle
- Recipient and occasion profiling

### Sub-Features
- Asks for **occasion → recipient → budget → vibe**, then recommends
- Add a whole gift bundle to the cart in one action
- Personal gift message that rides all the way through to checkout
  - Add, edit, or remove the gift message at any time
  - The message is shown on the cart card so it's never lost
- Occasion-aware suggestions (birthdays, anniversaries, holidays, and more)

---

## 4. Shopping Cart

### Features
- Add products
- Remove products
- Update quantities
- Review the full cart on request

### Sub-Features
- AI-assisted cart management directly from chat ("add the first one")
- Add-to-cart straight from product cards without leaving the results grid
- A live cart pill in the top bar
- Cart summary with the order-level gift message and one-click Edit / Remove
- Cart survives a server restart (state is safely stored server-side)

---

## 5. Checkout Experience

### Features
- Guided, conversational checkout
- Delivery information collection
- Order review before paying
- Payment link generation

### Sub-Features
- Sia collects delivery city, date, and recipient details conversationally
- Live **delivery availability check** before payment
- A clear checkout review card with the amount and any warnings
- Secure payment link card returned only after the customer confirms
- Built-in safety: payment links and checkout previews expire, and any cart change triggers a fresh review

---

## 6. Customer Order Management

### Features
- Order tracking
- Order history
- Delivery status updates

### Sub-Features
- Rich delivery-path tracking card with a status pill
  (Delivered / Out for delivery / Shipped / Cancelled / …)
- Ordered, shipped, and delivered dates, amount, payment method, and city
- "Show all my past orders" lists previously tracked orders
- Real recipient greeting taken from the order (never invented)
- Collapsible view of the detailed delivery progress updates

---

## 7. Reorder & Repeat Shopping

### Features
- Quick reordering of past purchases
- Preview before reordering

### Sub-Features
- List orders that can be reordered
- Preview a reorder, then add it to the cart in one step
- Reduces effort for loyal, repeat customers

---

## 8. Personalization Engine

### Features
- Returning-customer recognition
- Preference awareness
- Use of saved shopping context

### Sub-Features
- Recognises a returning browser and welcomes the shopper back by saved name
- **Time-aware welcome greeting** — "Good morning / afternoon / evening" based on the
  shopper's local clock, with their name appended when known (e.g. "Good morning, Nimal"),
  localized across English, Sinhala, Tamil, and Singlish
- **Real identity row** in the sidebar — shows the saved name and the delivery city from the
  shopper's most recent order (falls back to a saved recipient's city, then a neutral guest line)
- Saved **recipients** (the people you shop for)
- Saved **occasions** with upcoming-occasion reminders
- **Wishlist** — save items, view them, and add the wishlist to the cart
- **Customer memory** — Sia can remember preferences, with the customer in control
  (view, forget, and consent management)
- **Recommendation feedback** — Sia learns from what the customer liked or passed on

---

## 9. Voice & Future AI Features *(Planned)*

### Planned Features
- Voice conversations with Sia
- Speech-to-text shopping
- Text-to-speech responses
- Fully hands-free shopping assistant

---

# Admin Application

A separate dashboard for the business to monitor and steer the platform. Built as its own
secure app with login.

## 1. Dashboard (Overview)

### Features
- Business and platform overview at a glance
- Live system monitoring
- Usage insights

### Sub-Features
- System health and readiness status
- Session and activity counts
- Operational snapshot of the platform

---

## 2. AI Monitoring

### Features
- AI usage tracking
- Conversation/usage analytics
- AI cost monitoring

### Sub-Features
- Token usage tracking
- Per-model cost estimates
- **Provider switching** — choose the active AI provider (Claude / MiniMax) at runtime
- **AI budget controls** — monthly/daily budgets, per-session and per-request limits, thresholds
- **Emergency AI off-switch** for instant cost protection
- *Note: cost figures are local estimates, not provider billing records*

---

## 3. Customer Management

### Features
- Customer insights
- Shopping-behaviour visibility

### Sub-Features
- Customer profiles and metadata
- Returning-customer identification
- Drill-down into individual customer detail and recent sessions
- Recent session IPs for diagnostics
- *Read-only view*

---

## 4. Order Management

### Features
- Order oversight
- Saved tracked-order visibility

### Sub-Features
- Review saved order snapshots
- Status visibility for customer-support workflows
- Paginated, sortable tables for easy navigation
- *Read-only view*

---

## 5. Retention

### Features
- Customer retention insights

### Sub-Features
- Visibility into returning customers and repeat engagement
- Supports decisions that improve loyalty

---

## 6. Runtime Monitoring

### Features
- Live service health
- Operational diagnostics

### Sub-Features
- Health and readiness checks
- Redis key counts by category
- AI provider configuration status (shown only as "configured" / "missing" — never the keys)

> **Admin scope:** The admin app is intentionally focused and safe. It does **not** support
> product editing, inventory/pricing changes, fulfillment actions, refunds, payment
> reconciliation, or staff role management.

---

# Behind the Scenes (Backend Services)

A short, non-technical view of what powers the experience.

## AI Engine
- Orchestrates the AI assistant and its tools
- Manages conversation context and "smart actions"
- Connects to Claude (default) or MiniMax, with optional polish from OpenAI
- Supports streaming replies for a responsive feel

## Commerce Engine
- Live product search against the real Kapruka catalog
- Server-verified cart, pricing, and stock checks
- Guided checkout and secure payment-link creation
- Order tracking and reorder support

## Analytics & Monitoring
- AI usage and cost tracking
- Performance and health monitoring
- Budget enforcement and cost optimization

---

# Key Business Benefits

## For Customers
- Faster product discovery — just describe what you want
- Personalized recommendations and remembered preferences
- A natural, low-effort shopping experience
- Better, more thoughtful gift selection
- Confidence at checkout with clear reviews and tracking

## For the Business
- More conversion opportunities through guided shopping
- Stronger customer engagement and retention
- Reduced support workload via self-service tracking and reorders
- Actionable insights into shoppers and AI spend
- Full cost control over AI usage

---

# Technology Highlights

- 🤖 AI-Powered Conversational Commerce
- 🎯 Context-Aware, Personalized Recommendations
- 🧩 Live Catalog Integration (real Kapruka products, prices & delivery)
- 🔐 Server-Side Safety (verified prices, gated checkout, expiring payment links)
- 📊 Admin Monitoring, Budgets & Analytics
- 🌐 Multi-Provider AI (Claude / MiniMax, optional OpenAI polish)
- 🗣️ Future-Ready for Voice & Multilingual Shopping

---

# Current Project Status

### ✅ Implemented
- "Spotlight" shell — collapsible left sidebar, chat + live shopping canvas, auto-hiding rail
- Time-aware personalized welcome (Good morning/afternoon/evening + name) and real identity row (name · delivery city)
- Conversational AI shopping assistant (multi-turn, context-aware)
- Live product discovery, filtering, and comparison
- Guided gift flow with bundle recommendations
- Gift message that carries through to checkout
- Full cart management (add / remove / quantity / review) from chat and product cards
- Guided checkout with live delivery checks and order review
- Secure payment-link generation with expiry safeguards
- Order tracking with rich delivery-path cards and order history
- Reorder of past purchases
- Returning-customer recognition and personalized welcome
- Saved recipients, occasions, and wishlist
- Customer memory with consent controls and recommendation feedback
- Admin dashboard: Overview, AI Monitoring, Customers, Orders, Retention, Runtime
- AI provider switching, budget controls, and emergency off-switch
- Streaming chat responses

### 🚧 In Progress
- Multilingual experience (English / Sinhala / Tamil / Singlish) — the personality and
  conversation foundation are in place; the language detection and mirroring layer is being added
- Broader, persistent analytics and audit history

### 🔮 Planned
- Voice conversations (speech-to-text and text-to-speech shopping)
- Deeper multilingual generation quality
- Expanded admin analytics and retention tooling

---

*SIA — shopping that feels like a conversation, not a search box.*
