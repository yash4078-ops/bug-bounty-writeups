<div align="center">
🛒 Unauthenticated Zero-Value (₹0.00) Order Creation
via an Exposed Internal Test Product on a Production E-Commerce Platform

Severity: P2 · CVSS: 7.0 · Class: Business Logic Flaw / Insecure Design
Status: ✅ Fixed & Revalidated · Bounty Awarded

#BugBounty #BusinessLogic #EcommerceSecurity #ResponsibleDisclosure #AppSec

</div>
📌 TL;DR

A production e-commerce store had an internal, dev-only test product — literally named "Do not buy" — sitting live and publicly purchasable, priced at ₹0.00. No login, no discount code, nothing. Any guest could add it to cart, breeze through checkout, and walk away with a fully confirmed order: valid Order ID, confirmation email, the works.

A forgotten test SKU in production + no server-side price floor = free, fully-confirmed orders for anyone.

🧭 Table of Contents
Background
Discovery
Proof of Concept
Impact
Root Cause
Remediation
Timeline
Lessons for Bug Hunters
Lessons for Developers
🔎 Background

E-commerce checkout flows are usually guarded at the payment gateway — but a huge assumption quietly sits underneath that: "a product will always have a real price." When that assumption breaks (test data leaking into production, a pricing bug, a promo edge case), the payment step becomes irrelevant — because there's simply nothing to pay.

This is a classic Insecure Design issue (OWASP Top 10 — A04:2021): the vulnerability isn't a broken auth check or an injection flaw, it's a missing business rule.

🕵️ Discovery

Public storefronts often expose a predictive-search / autocomplete API for the search bar. These endpoints are meant to return live, sellable products — but they don't discriminate on how a product got created:

GET /search/suggest.json?q=<query>

Scanning through the JSON response, one entry stood out — an internal product clearly labeled as a dev/test item, with:

Price: ₹0.00
Publicly enumerable Product ID and Variant ID
No visibility restriction (should have been in "Draft" state, wasn't)
🧪 Proof of Concept

Step 1 — Discover the exposed test product via the public search API

GET https://<target>/search/suggest.json?q=test

Response includes the internal test product with price: "0.00" and a valid, guessable variant ID.

Step 2 — Add the ₹0 variant directly to cart (no auth required)

GET https://<target>/cart/<variant_id>:1

Item is added successfully to an unauthenticated guest session.

Step 3 — Proceed through guest checkout

Standard shipping details entered — no discount code, no coupon, no promo applied.

Subtotal: ₹0.00
Shipping: Free
Total:    ₹0.00

Step 4 — Complete checkout

Order is placed successfully:

✅ Valid Order ID generated
✅ "Thank You / Order Confirmed" page rendered
✅ Order confirmation email delivered — proving the backend order-lifecycle workflow (inventory, shipping, notifications) was fully triggered, not just a UI-level illusion

🛑 Ethical boundary respected: Testing was limited to a single order, performed responsibly, with no attempt to place bulk/automated zero-value orders or exploit this at scale.

💥 Impact

Because this bypassed payment entirely — not just discounted it — the blast radius went well beyond "someone got a free item":

💸 Operational cost: confirmed orders trigger real backend workflows — warehouse picklists, shipping labels, logistics — regardless of whether real money changed hands
📦 Inventory distortion: fake orders against SKUs sharing characteristics with live products can skew stock visibility for genuine customers
📊 Data integrity violation: zero-value orders pollute sales databases, analytics dashboards, and demand-forecasting models
🔓 Checkout logic bypass: the core e-commerce guarantee — "no payment, no order" — was broken with zero authentication and zero interaction complexity

Relevant CWEs:

CWE	Why it applies
CWE-840 — Business Logic Errors	Zero-value order creation bypassed intended payment logic
CWE-639 — Authorization Bypass via User-Controlled Key	Publicly guessable variant ID let guests add the item directly
CWE-200 — Exposure of Sensitive Information	Internal test product was discoverable via a public search API
🧩 Root Cause
An internal/dev-only test product was deployed to production instead of staging, and left in a publicly purchasable state instead of "Draft."
No server-side floor check on order total — the checkout flow trusted that "a listed product has a real price" instead of explicitly rejecting total_price == 0.00 outside of authorized promotions.
Public predictive-search API had no filtering to exclude internal/test-tagged products from results.
🛠️ Remediation
Immediate cleanup: cancel any zero-value orders already created; archive/unpublish the exposed test product from the live storefront.
Enforce a payment floor server-side: reject checkout completion whenever total_price == 0.00, unless an explicitly authorized discount/promotion flow set it that way.
Segregate environments properly: internal/testing products must never exist in a state reachable by the production storefront — use environment-scoped catalogs or a hard "internal-only" flag enforced at the API layer, not just UI visibility.
Filter internal-tagged products out of public APIs (search, autocomplete, sitemap, etc.) — visibility gaps in one surface (storefront UI) don't mean the data isn't reachable through another (API).
Add anomaly monitoring: alert on any confirmed order with total == 0 or unusual SKU patterns, so an escape like this gets caught within minutes, not weeks.
⏱️ Timeline
Date	Event
Report submitted	Full PoC (order confirmation + email evidence) submitted to the program
Triage	Report reviewed, target URL corrected to the actual affected storefront
Fix shipped	Test product removed/unpublished; direct cart-add to the exposed variant ID began returning "link no longer exists"
Revalidation requested	Program asked for a fresh revalidation PoC before closing
Revalidation submitted	Confirmed cart manipulation blocked; zero-value flow no longer reachable
Severity discussion	Researcher flagged a post-closure severity change (P1→P2) and asked for the CVSS rationale — kept the tone factual, cited the specific CWE/CVSS reasoning originally used
Resolution	Report closed, bounty awarded
🎓 Lessons for Bug Hunters
Public search/autocomplete APIs are an underrated recon surface. They often return raw catalog data — including things that were never meant to be customer-facing — well before you'd find them by browsing the UI.
A ₹0 price tag is worth chasing all the way to order confirmation. Don't stop at "added to cart" — the real proof of impact is a confirmed order with a real Order ID and a real confirmation email, showing the backend fully accepted the transaction.
If severity changes after closure, ask — once, factually. Reference the exact CWE/CVSS reasoning from your original report rather than just expressing frustration. It keeps the conversation professional and gives the triager something concrete to respond to.
Payout follow-ups should stay polite and spaced out, referencing the report/closure date each time — persistence without pressure gets better long-term relationships with programs than repeated escalation.
Always ask before publishing. Getting explicit written permission from the program before writing anything public — even a "high-level, non-technical" post — protects both the researcher and the organization, and most programs are receptive when asked respectfully.
👨‍💻 Lessons for Developers
Treat "Draft" / internal-only product states as a hard access-control boundary, not just a UI convenience — verify it's enforced at every API surface (search, cart, direct product URLs), not only the storefront listing page.
Never trust that "a product exists in the catalog" implies "it has a valid, non-zero price." Validate the actual order total server-side at the final checkout step, independent of how the cart got populated.
Staging and production catalogs should be structurally separated, not just hidden by naming convention ("do not buy") — naming is not access control.
<div align="center">

Found something similar in the wild? Report it responsibly through the appropriate bug bounty program. 🛡️

This writeup omits the target name and any account-identifying details in line with responsible disclosure practice — the underlying issue was fully remediated and independently revalidated before publication.

</div>
