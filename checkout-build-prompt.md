# Build Prompt: Checkout Application (Retail Micro-Frontend Platform — Remote)

Use this as the instruction prompt for an AI coding agent (e.g. Claude Code) to scaffold and build the Checkout project.

---

## Context

You are building **Checkout**, the last of three independently deployable remotes in a teaching example of enterprise Angular micro-frontend architecture. The other projects are the **Shell** (host, owned by a separate platform team), **Catalog**, and **Account** — you do not own their internals, only your contract with them.

This project has two teaching goals, and they are not the same thing:

1. **Modern Angular (the "how"):** Angular 22, standalone components (no NgModules), Signals as the default reactivity model, zoneless change detection, `OnPush` as the default, modern control-flow syntax (`@if`/`@for`/`@switch`), lazy loading, and Native Federation as a **remote**.
2. **Enterprise architecture (the "why"):** Checkout is the highest-stakes remote on the platform — it is the one that handles money and the one Shell's nav depends on for live cart state. That raises the reliability and isolation bar above what Catalog or Account need: cart-state contract design, the receiving end of a cross-remote handoff (Catalog → Checkout), the hardest version of the "don't let a remote's auth assumptions leak" problem, and defining what "must never silently disappear" actually means in code, not just in a diagram.

The code should teach both. **Comment generously and explain the architectural reasoning, not just what the code does.** A comment like `// adds item to cart` is useless here. A comment like `// Cart count is exposed to Shell as a signal on the shared contract, not pushed via an event — Shell's nav needs to *read* current state on demand (e.g. on route change), not just react to changes that happened while it was mounted` is the target quality bar. Assume the reader is an experienced Angular developer who has *not* built a micro-frontend platform before.

---

## Data source: Northwind Traders on Supabase (write side)

Catalog reads Northwind's `products`/`categories` from Supabase. Checkout is the remote that **writes** — placing an order touches Northwind's `orders`/`order_details` shape. This is deliberately a harder problem than Catalog's read-only case, and the prompt wants you to treat it as one rather than paper over it:

- Model **Cart** as Checkout's own in-memory/session concept (a signal-based service owned entirely by Checkout) — carts are not a Northwind/Supabase concept and should not be forced into one.
- Model **order placement** as a write against Supabase's Northwind-shaped `orders`/`order_details` tables (or a Checkout-owned equivalent, adapted at the same anti-corruption boundary pattern Catalog uses for reads).
- **Do not use Supabase Auth here either**, for the same reason Catalog doesn't: Shell owns identity via the shared `AuthSessionService` contract. But this creates a real, worth-naming tension — Supabase's anon key is meant for public reads, not authenticated writes tied to a specific user's identity, and Shell's session isn't a Supabase session Supabase's RLS can natively check. **Do not silently work around this by just using a permissive public-insert RLS policy and calling it done.** Implement the simplest working version for the exercise (e.g., a public-insert policy scoped tightly to the `orders`/`order_details` tables only, nothing else), but write this decision up explicitly in `ARCHITECTURE.md` as a known simplification, and describe how a production system would actually solve it (e.g., a thin server-side function that verifies Shell's session token and performs the write with a privileged key, never exposed to the client). This gap — "the demo's shortcut vs. the production answer" — is itself part of what you're teaching.
- Do not let Supabase/Northwind's write-side schema (`orders`, `order_details`, foreign keys to `products`) leak into Checkout's component layer — adapt at the data-access boundary, same discipline as Catalog.

---

## What Checkout Owns

Checkout is a **bounded context**: everything about turning a selected product into a placed order.

- **Cart**: add/remove/update line items, computed totals — a Checkout-owned, signal-based cart service.
- **Checkout flow**: review cart → (stubbed) payment step → order confirmation. **Do not implement real payment processing.** Stub the payment step behind a clearly fake/mock gateway (e.g. a component that simulates success/failure) — this is an architecture exercise, not a PCI-compliance one, and pretending otherwise would be actively misleading.
- **Order placement**: writing the completed order to the Northwind-shaped Supabase tables via Checkout's own data-access layer.
- **The receiving end of Catalog's handoff**: implement the consumer side of the "product selected for purchase" event/contract that Catalog's `CONTRACT.md` defines — a product added from Catalog should land correctly in Checkout's cart.
- **The cart-state contract with Shell**: cart item count needs to show up in Shell's global nav. Decide deliberately — signal exposed on the shared contract that Shell reads, or an event Checkout emits on change — and implement it. Document why you chose the mechanism you chose.
- **Its own internal routing**: Checkout manages routes within its own bounded context (e.g. `/checkout/cart`, `/checkout/payment`, `/checkout/confirmation`); Shell only knows about the single mount point it delegates to Checkout.
- **Its own release cadence and its own reliability bar**: structure the project so it deploys independently, and treat "Checkout must never silently disappear" as a concrete requirement (see fallback behavior below), not a slogan.

## What Checkout Explicitly Does NOT Own

- Product data or browsing — that's Catalog's. Checkout receives a product reference (id, name, price snapshot) via the handoff contract; it does not re-fetch or re-own the product catalog.
- Identity/session management. Checkout consumes the same `AuthSessionService` contract every remote consumes — it does not implement its own login and does not manage its own token.
- Real payment processing, PCI-scope handling, or any real financial transaction. Stubbed only, as above.
- Global navigation, header, or layout chrome, or the cart *icon/badge UI itself* in the nav — Checkout exposes the cart count data; Shell decides how to render it.
- The Native Federation host configuration — Checkout only defines what it *exposes*, not how or when Shell chooses to load it.

---

## Requirements

### Angular / technical
- Angular 22, standalone components only, no NgModules.
- Zoneless change detection (no Zone.js dependency).
- `OnPush` as the default change detection strategy throughout.
- Signals for all local state (cart contents, checkout flow step) — no bespoke RxJS state management unless justified in a comment for why signals weren't a fit.
- Modern control-flow template syntax (`@if`, `@for`, `@switch`) — no `*ngIf`/`*ngFor`.
- Native Federation configured as a **remote**: expose Checkout's root routed component (or a small set of exposed entry points) in the federation manifest so Shell can lazy-load it without needing Checkout's internal file structure.
- Use `httpResource()`/`resource()` for the order-placement write and any supporting reads, instead of manual subscriptions.

### Architecture
- **Bounded context clarity**: the folder structure should make it obvious what is internal to Checkout (cart/payment/order feature code) vs. the exposed remote entry point vs. what is consumed from the shared contract library.
- **Consuming the runtime contract, not the host**: Checkout depends only on the versioned shared library (design tokens/base components, `AuthSessionService`) that Shell also depends on — never on Shell's own internal code, services, or state.
- **Implementing the Catalog handoff**: consume the exact event/contract shape Catalog's `CONTRACT.md` defines for "product selected for purchase." If that shape is ambiguous or missing something Checkout needs, note the gap explicitly in `CONTRACT.md` rather than quietly inventing extra fields Catalog doesn't know about.
- **Cart-state contract with Shell**: implement the mechanism decided above, and make it resilient to Shell mounting/unmounting Checkout at different times than expected (e.g., what does Shell read for cart count *before* Checkout has ever been loaded this session?).
- **Fallback behavior as a first-class requirement**: since Checkout is the remote that must never silently disappear, implement (don't just describe) what happens if Checkout fails to load — this lives partly in Shell's error boundary, but Checkout should expose whatever health/version signal Shell needs to detect a bad load quickly.
- **Write-side resilience**: order placement can fail (network, Supabase, the RLS/auth simplification above). Handle it as a first-class state in the checkout flow — a failed order must never look like a successful one to the customer, and retry/error UI is not optional.
- **Independent deployability**: prove that Checkout can be rebuilt and redeployed on its own, and that Shell picks up the new version at runtime without a Shell rebuild.
- **Version compatibility**: document how Checkout pins/verifies the version of the shared contract library it was built against.
- **Observability**: add lightweight structured logging for Checkout's own lifecycle events (remote mounted, cart mutations, handoff received from Catalog, order placement success/failure, load timing), using the same event shape/fields as Shell's, Account's, and Catalog's logging so a platform team can correlate across remotes — and so a failed checkout can actually be traced after the fact, which matters more here than anywhere else on the platform.

### Documentation deliverables
Alongside the code, produce:
1. `ARCHITECTURE.md` — explains Checkout's bounded context, the cart-state contract mechanism and why it was chosen, how the Catalog handoff is consumed, the Supabase write-side simplification and how production would actually solve it, and Checkout's failure/resilience strategy — written for another engineer joining the project.
2. `CONTRACT.md` — what Checkout exposes to Shell (its remote entry point/mount path, the cart-count contract mechanism and its exact shape/signal name), what it expects to receive from Catalog (the handoff event shape it consumes), and what it expects from the shared contract library.

---

## Output format

- Provide the full file tree for the Checkout project first, then the code, file by file, with inline comments per the standard above.
- After the code, provide `ARCHITECTURE.md` and `CONTRACT.md`.
- Include the exact RLS policy SQL used for the `orders`/`order_details` write path, alongside a clear, explicit callout of its limitations as a demo simplification.
- End with a short "how to run this locally, standalone and federated into Shell" section, including where to put the Supabase URL/anon key.

Do not silently skip the fallback-behavior, cart-state contract, or write-side resilience requirements to save time — they are the point of the exercise, not optional polish. Do not implement or simulate real payment processing under any framing.
