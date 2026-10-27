# Intigriti Challenge 1026 — Alternating Promotion Codes

> **CTF write-up.** This documents the intended behavior observed in Intigriti’s isolated Challenge 1026 storefront. It is not a report of a vulnerability in a production merchant.

## Challenge

Challenge 1026, “Come play our monthly CTF!”, presented a small online store with a €666.00 archive item, promotional codes, simulated store credit, and a challenge flag shown after a successful checkout. The challenge rules asked researchers to find a vulnerability on the challenge page and submit the flag, payloads, and short solve steps. Self-XSS and man-in-the-middle solutions were excluded.

All testing described here was confined to the challenge host:

`https://challenge-1026.challenges.intigriti.io`

## Finding

The cart rejected applying the same coupon twice in a row, but it accepted the same coupon again after a different coupon had been applied. Alternating two percentage discounts therefore reused each code and reduced the eligible item’s total to zero.

The relevant sequence was:

```text
NIGHT20 → NEWS30 → NIGHT20 → NEWS30
```

`NIGHT20` was shown for the Midnight Collection. `NEWS30` was revealed by the storefront’s newsletter form using a synthetic `.invalid` email address. Each coupon was accepted twice when alternated.

## Reproduction

1. Open the challenge storefront in a fresh session.
2. Add one **The Midnight Collection** item (the `midnight-vault` product, €666.00) to the cart.
3. Use the newsletter form with a synthetic address such as `ctf-1026@example.invalid` to reveal `NEWS30`.
4. Apply the following codes one at a time, in this exact order:

   ```text
   NIGHT20
   NEWS30
   NIGHT20
   NEWS30
   ```

5. Inspect the cart: both promotions show a count of two and the total is €0.00.
6. Complete the normal simulated challenge checkout. The confirmation returns the flag below.

## Result and arithmetic

| State | Discount added | Cart total |
| --- | ---: | ---: |
| One €666.00 item | — | €666.00 |
| First `NIGHT20` (20%) | €133.20 | €532.80 |
| First `NEWS30` (30%) | €199.80 | €333.00 |
| Second `NIGHT20` (20%) | €133.20 | €199.80 |
| Second `NEWS30` (30%) | €199.80 | €0.00 |

The simulated checkout completed with a €0.00 total and returned:

```text
INTIGRITI{094638e6-ba2f-4c50-8bb4-41ced0d5debd}
```

The challenge confirmation showed order reference `BLE-F7CD7B13C8`. This was a simulated CTF order, not a real purchase or payment.

## Why the sequence works

The important observation was not merely that multiple promotions could be present. Repeating one code immediately was rejected, but switching to another code and then returning to the first allowed reuse. Repeating that alternating pattern applied each discount twice.

The challenge behavior is consistent with duplicate-use validation that considers only the most recently applied promotion rather than checking the full set of promotions already applied. That is an inference from the observed sequence, not a claim based on confirmed server-side source code.

## Impact in the challenge

The sequence bypassed the storefront’s apparent one-use-per-code behavior, removed the entire price of the eligible item, and enabled the challenge checkout to return its flag. This describes impact only inside the deliberately vulnerable CTF application; it does not establish impact on any real store.

## Defensive takeaway

For a real commerce system, coupon eligibility and discount totals should be validated against canonical server-side cart state. Enforce per-order/per-customer usage rules across the complete applied-promotion set, apply the checks atomically, and cap the aggregate discount so the payable total cannot fall below the intended minimum. The checkout endpoint should recalculate and verify the final amount rather than trusting client-side presentation state.

## Research notes

- Used one eligible challenge item and only the two codes exposed by the challenge flow.
- Used a synthetic `.invalid` email address to reveal the newsletter code.
- Performed the sequence in Chrome Incognito on the in-scope challenge host.
- No personal email, real payment method, signed-in browser profile, off-scope host, or other participant’s data was involved.
- No real-world merchant vulnerability is alleged.

## Publication note

The challenge rules say to make a write-up public only after the challenge has ended **and** new submissions are no longer accepted. The posted challenge end time was 26 October 2026 at 23:59 UTC. Confirm on the live challenge page that submissions are closed before making this repository public.
