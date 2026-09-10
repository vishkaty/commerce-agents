# Proposal: cart re-validation at checkout

## The gap

`checkout` renders the cart the backend returns and puts `checkout_handoff`'s URL on the
card. Between the card and the payment, a line can sell out or move in price; on a live
platform (Medusa v2) a cart line keeps its add-time price after the merchant reprices,
so the card and the hosted checkout charge a stale price. The contract says nothing
about re-validation, and if `checkout_handoff` raises `Unavailable` the executor reports
an outage, because `Unavailable` is not a `ValueError` that `run_presentation` relays.

## What this adds

1. `docs/backends.md`, `checkout_handoff`: "re-validate every line against the live
   catalog before returning a URL; a line that can no longer be fulfilled raises
   `Unavailable` naming it; a line whose price moved is refreshed and reported so the
   next checkout shows what will be charged."
2. `enrich_checkout` catches `Unavailable` from the handoff and raises
   `PresentationRefused` with its message, so the model hears the refusal, not an
   outage.
3. The host side: complete a paid cart only when the amount the provider collected
   matches the cart's total at completion; otherwise hold it for review.

## Reference implementation

Done over Medusa v2 in a lab adapter: `_revalidate` (sold out, short, repriced), a
`CartStale(Unavailable, ValueError)` so the reference card relays it today, and a host
that holds a mismatched payment. Verified live with a repricing between add and pay.
