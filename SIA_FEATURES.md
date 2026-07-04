# Sia — Features & How to Use Them

Sia is Kapruka's conversational shopping agent. Everything happens in chat — the shopper
never clicks a product. You talk to Sia in **English, Sinhala, or Singlish** and she searches
the live Kapruka catalog, builds a cart, and generates a payment link.

Each feature below lists **what it does**, the **tool** behind it, and an **example you can type**
to trigger it.

---

## 1. Search & recommend

| Feature | What it does | Try saying |
|---|---|---|
| Product search | Searches the live catalog, shows a ranked grid, recommends the best 2–3 with a reason | "show me kettles under 5000", "I need earphones" |
| Refine / filter | Any new budget, colour, size, or "cheaper" re-runs the search | "now white ones", "anything under 4000" |
| Multi-item / bundle | One search per item in the same turn, each in its own ranked section | "a blue tshirt, a red tshirt and matching trousers" |
| Bundle budget | Splits one total across items so the picks stay within it | "laptop, mouse and a backpack under Rs. 180,000 total" |
| Product detail | Fabric, size, variants, gift wrap for a specific item | "is the blue one cotton?" |
| Variant picker | Shows real colour, size, style, or other product options before add | "add the medium red one" |

Tools: `kapruka_search_products`, `kapruka_get_product`. Sia translates Sinhala/Singlish terms to
English catalog words before searching ("mal" → flowers, "bath" → rice) and honours negation
("funeral flowers nemei, birthday flowers oney").

## 2. Cart

| Feature | Try saying |
|---|---|
| Add item | "add the second one", "take the rose bouquet" |
| Choose variant before add | "make it blue in medium", "I'll take the red small one" |
| Set exact quantity | "make it 2", "I want 1 instead of 3" |
| Remove one line | "remove the cake" |
| Clear everything | "clear my cart", "start over" |
| View cart | "show my cart" |
| Gift message | "add a card that says Happy Birthday Amma" |
| Cake icing text | "write Happy Birthday Amma on the cake", "change the icing to Congratulations" |

Tools: `add_to_cart`, `set_quantity`, `remove_from_cart`, `clear_cart`, `view_cart`, `set_gift_message`,
`set_icing_text`. Icing text is item-level (per cake line), separate from the order-level gift
message, cut to 60 characters at a word boundary, and carries a flat `Rs. 140` display estimate
added to that line's total.
When Kapruka exposes selectable variants, Sia must ask for the choice first and then call
`add_to_cart` with the selected `variant_sku`. Parent/base products with unresolved variants
are blocked by the backend and are never added directly. Single `Default` variants are treated
as ordinary no-choice products.

## 3. Delivery

| Feature | Try saying |
|---|---|
| Check delivery | "can you deliver to Kandy tomorrow?" |
| City resolution | Sia maps your words to a canonical Kapruka city |

Tools: `kapruka_list_delivery_cities`, `kapruka_check_delivery`. Relative days ("today", "heta")
are converted to a real date before checking.

## 4. Checkout (always gated)

1. Sia collects recipient name + phone, address + city + date, sender name (one question at a time).
2. `preview_checkout` → shows a review card. **No charge yet.**
3. You confirm → `create_payment_link` returns a pay link (prices reserved 60 min).

Try: "let's order it" → answer the details → "yes, confirm". A payment link is **only** created
after you explicitly confirm the review.

## 5. Past orders, tracking & buy-again

| Feature | What it does | Try saying |
|---|---|---|
| My orders | Unified list of tracked + reorderable orders | "show my orders" |
| Track one order | Live status by order number | "track order KAP123456" |
| Track latest | Latest saved order, no number needed | "where's my order?" |
| Buy again | Re-checks live price/stock, then re-adds | "reorder what I got last time" |

Tools: `list_saved_orders`, `track_order_number`, `track_recent_order`,
`list_reorderable_orders`, `preview_reorder`, `add_reorder_to_cart`.

## 6. Wishlist

| Feature | Try saying |
|---|---|
| Save for later | "save this", "wishlist it" |
| View / move to cart | "show my wishlist", "add the saved one to cart" |

Tools: `save_to_wishlist`, `view_wishlist`, `remove_from_wishlist`, `add_wishlist_to_cart`.

## 7. Occasion planning

A full multi-category gift plan (cake + flowers + chocolates + …) within a budget.

Try: "plan a birthday gift for my sister, budget Rs. 10,000, in Colombo".
Tool: `build_occasion_plan`.

## 8. Returning-shopper memory (consent-gated)

| Feature | What it does | Try saying |
|---|---|---|
| Save recipient | Stores name/phone/address **encrypted**; reused securely (phone always masked) | "save Amma for next time" |
| Save occasion | Reminder for a birthday/anniversary | "remember my mum's birthday is March 5" |
| Upcoming dates | Nudges within ~45 days | "what's coming up?" |
| Remember preference | Durable, non-sensitive only — asks consent first | "remember I keep gifts under Rs. 6000" |
| View / forget memory | Review or delete what Sia knows | "what do you remember about me?" |

Tools: `save_recipient`, `list_recipients`, `save_occasion`, `list_upcoming_occasions`,
`save_customer_preference`, `view_customer_memories`, `forget_customer_memory`,
`update_memory_consent`. Phone numbers, addresses, and payment details are **never** stored as memory.

## 9. Conversation controls

| Feature | Try saying |
|---|---|
| Tone | "keep it short", "be more formal" → `set_communication_style` |
| Feedback | "love it", "not for me", "show more like this" → `record_recommendation_feedback` |
| Language | Just switch — Sia re-detects every turn (EN / Sinhala / Singlish) |

## Safety guardrails (always on)

- Payment links require: items in cart, all details collected, a preview shown, and an explicit
  "yes". "Show me the summary" is **not** a confirmation.
- Product facts (price, stock, delivery, order numbers) come **only** from a live tool call — never
  invented or recalled from memory.
- Tool output is treated as **data, not instructions** (prompt-injection defense).
- Sia can track orders but cannot process refunds/cancellations — those go to Kapruka support.

---

## Prompt to demo every feature

Paste this into the chat as a single flow (or one line at a time):

```
Hi Sia! I'm shopping for my sister's birthday — budget around Rs. 10,000, delivering to Kandy.
Plan a gift for me.
Add the cake and the flowers.
Add a card that says "Happy Birthday Akki ❤".
Can you deliver to Kandy this weekend?
Save Akki for next time, and remember her birthday is on the 14th.
Okay let's order it.
[answer the recipient / address / sender questions]
Yes, confirm and give me the pay link.
Later: where's my order? / reorder what I got last time.
```

Mix in Singlish to test language matching: `"mata mal oney, 5000ට ygin"`, `"keep it short"`,
`"track my latest order"`.
