# Contribution 2: 500 TypeError (reading 'calculated_amount') instead of 400 "do not have a price" when adding a variant unpriced in the cart's region
 #15932


**Contribution Number:** 2
**Student:** Henry Huang
**Issue:** https://github.com/medusajs/medusa/issues/15932
**Status:** Phase 3 Complete

---

## Why I Chose This Issue

This issue interests me because I like to insure web apps are in the best shape they can be. I enjoy building and fixing software across the full stack. This app happens to be fintech related, which I've done work on before.

---

## Understanding the Issue

### Problem Description

Currently, when POST /store/carts/{id}/line-items returns a 500 error when a variant for an item exists and is published but has no calculated price for the cart's region/currency.


### Expected Behavior

It should return a 400 error with message "Variants with IDs … do not have a price"

### Current Behavior

Instead a raw 500 (TypeError: Cannot read properties of undefined (reading 'calculated_amount') comes back

### Affected Components

[Which parts of the codebase are involved?]

---

## Reproduction Process

### Environment Setup

Node v22
PostgreSQL 16

### Steps to Reproduce

1. Create two regions with different currencies (e.g. eur and xaf).
2. Create a published product whose variant has a price only in eur.
3. POST /store/carts with the xaf region → 200.
4. POST /store/carts/{id}/line-items with that variant id → 500 TypeError (expected: 400 "do not have a price").

### Reproduction Evidence

TypeError: Cannot read properties of undefined (reading 'calculated_amount')
    at .../@medusajs/core-flows/dist/cart/workflows/get-variants-and-items-with-prices.js:106
    at Array.map (<anonymous>)
    at .../get-variants-and-items-with-prices.js:74
    at .../@medusajs/workflows-sdk/dist/utils/composer/helpers/transform.js:31

---
  Solution Approach

  Analysis

  prepareVariantsAndItemsWithPricesStep in
  packages/core/core-flows/src/cart/workflows/get-va
  riants-and-items-with-prices.ts performed
  validation and line-item construction in a single
  .map() pass, but deferred throwing until after the
  pass completed.

  For each item the step resolved a
  calculatedPriceSet and, when none was found for a
  non-custom-priced variant, pushed the variant id
  onto priceNotFound (line 109-111). It then
  continued in the same iteration to build the line
  item, where the price assignment branch guarded
  only on variant && !isCustomPrice — not on whether
  a price set had actually been found — and
  dereferenced calculatedPriceSet.calculated_amount
  (line 138-142).

  So the very item just flagged as unpriced
  immediately raised TypeError: Cannot read 
  properties of undefined (reading 
  'calculated_amount'). The MedusaError assembled
  from priceNotFound sat 20 lines below the .map()
  and was never reached. The result: POST 
  /store/carts/{id}/line-items returned an opaque
  500 for what is an ordinary business condition (a
  catalog gap — variant published but priced only in
  another region's currency), diagnosable only from
  server logs.

  The same structural flaw affected the sibling
  error path: prepareLineItemData throws "Variant 
  does not have a product" for a variant with no
  product, which would preempt the step's more
  specific "do not exist or belong to a product that
  is not published" message. That path is not
  reachable through today's callers —
  create-order.ts:319, add-line-items.ts:163,
  create-carts.ts:239 and add-to-cart.ts:180 all
  request variant fields derived from
  productVariantsFields, which includes product.* —
  so it is a latent footgun rather than a live bug.

  Proposed Solution

  Split the step into a validation pass and a build
  pass, so both MedusaErrors are thrown before any
  line item is constructed.

  This is chosen over the narrower alternative
  (adding && calculatedPriceSet to the assignment
  branch) because the collect-then-throw-later
  structure is what makes the bug possible: any
  future code added inside that map that touches
  calculatedPriceSet re-opens the same class of
  failure. The narrow guard is also applied, as
  defense in depth.

  Implementation Plan

  Using UMPIRE framework (adapted):

  Understand: When a published variant has no
  calculated price for the cart's region/currency,
  adding it as a line item must return HTTP 400
  type: invalid_data with "Variants with IDs <id> do
  not have a price" — the error the code already
  intends to raise. It currently returns a raw 500
  TypeError because the price set is dereferenced
  before that error is thrown.

  Match: The codebase already has the target shape.
  packages/core/core-flows/src/cart/steps/validate-v
  ariant-prices.ts is a dedicated validation-only
  step that accumulates into a priceNotFound array
  and throws a single MedusaError(INVALID_DATA, ...)
  with the byte-identical message — with no
  construction logic interleaved. The fix makes the
  inline logic in
  get-variants-and-items-with-prices.ts conform to
  that same separation. Error conventions follow
  CLAUDE.md §5.4 (MedusaError.Types.INVALID_DATA for
  invalid input/state). The unit-test harness —
  wrapping a step in a throwaway createWorkflow and
  running it against a bare Awilix container — is
  copied from
  packages/core/core-flows/src/product/steps/__tests
  __/process-product-options-for-import.spec.ts.

  Plan:

  ?? [] to return a resolved intermediate ({ item, 
  variant, calculatedPriceSet, isCustomPrice })
  instead of a finished line item. Validation
  collection and the variant.calculated_price
  assignment stay in this pass, unchanged.
  2. Move both throw new MedusaError(...) blocks
  above the build pass, preserving precedence
  (variant errors before price errors).
  3. Add a second .map() over the resolved
  intermediates that builds PrepareLineItemDataInput
  and calls prepareLineItemData, with
  calculatedPriceSet added to the price-assignment
  guard.
  4. Add
  packages/core/core-flows/src/cart/workflows/__test
  s__/get-variants-and-items-with-prices.spec.ts.
  5. Add a changeset (@medusajs/core-flows: patch).

  Implement: Branch
  fix/cart-line-item-validate-before-build, commit
  5d59ff0909 (+164/−24 across 3 files). Not yet
  pushed or opened as a PR — see Implementation
  Notes for why.

  Review:
  - [x] Follows CLAUDE.md formatting (no semicolons,
  double quotes, 2-space indent, ES5 trailing
  commas) — verified via the repo's Prettier config
  - [x] Uses MedusaError with INVALID_DATA per §5.4
  rather than a bare throw
  - [x] Changeset included, matching the format of
  existing entries (.changeset/all-dingos-admire.md)
  - [x] No public API, type signature, or step
  input/output shape changed
  - [x] Error precedence preserved
  (variant-not-found/unpublished still wins over
  missing-price)
  - [x] No emojis
  - [ ] Test location —  packages/core/core-flows/src/cart/workflows/__test
  s__/get-variants-and-items-with-prices.spec.ts

  Evaluate: New unit spec must fail on the pre-fix
  code with a raw TypeError and pass after; existing
  core-flows suites must stay green; the package
  must typecheck.

  ---
  Testing Strategy

  Unit Tests

  packages/core/core-flows/src/cart/workflows/__test
  s__/get-variants-and-items-with-prices.spec.ts —
  5/5 passing via npx jest src/cart:

  - [x] Test case 1: Happy path — a variant with a
  calculated price produces a line item carrying
  unit_price: 1000 and is_tax_inclusive: false
  (guards against regression in the reordering)
  - [x] Test case 2: Issue repro — a published
  variant with no calculated price rejects with
  type: invalid_data and message "Variants with IDs 
  variant_1 do not have a price", not a TypeError
  - [x] Test case 3: Multiple unpriced variants are
  aggregated into one message ("variant_1, 
  variant_3") while the priced variant_2 is skipped
  — proves the collect-all-then-throw behaviour
  survived the split
  - [x] Test case 4: Error precedence — a
  DRAFT-product variant that is also unpriced
  reports the variant error, not the price error
  - [x] Test case 5: An item carrying an explicit
  unit_price still bypasses the price requirement
  entirely (is_custom_price: true, unit_price: 500),
  so the fix doesn't over-reject custom-priced
  items

  Implementation detail worth recording: the
  workflow runtime serialises a thrown MedusaError
  into a plain object, so rejects.toThrow() reports
  "Received function did not throw" and silently
  fails to match. Assertions use
  rejects.toMatchObject({ type, message }) instead.
  This cost a debugging cycle and is not obvious
  from the existing specs.

  Integration Tests

  - [ ] Integration scenario 1: POST 
  /store/carts/{id}/line-items for a variant with no
  price returns 400 — not written here. PR #15939
  already covers exactly this in integration-tests/h
  ttp/__tests__/cart/store/cart.spec.ts (+48 lines),
  added after a maintainer asked for it.
  - [ ] Integration scenario 2: The issue's actual
  multi-region repro (variant priced in eur, cart in
  xaf) — not covered by either change. Both
  #15939's test and mine use a variant with no
  prices at all, which hits the same code path but
  is a weaker reproduction of the reported scenario.

  Manual Testing

  - npx jest src/cart → 2 suites, 14 tests passing
  (the new spec plus the pre-existing
  prepare-confirm-inventory-input.spec.ts)
  - npx tsc --noEmit -p tsconfig.json in
  packages/core/core-flows → exit 0
  - Read-through of prepare-line-item-data.ts
  confirming no residual 500 path: every
  calculated_price read there is optional-chained
  (lines 112, 119, 127), so an undefined price set
  variant, calculatedPriceSet, isCustomPrice })
  instead of a finished line item. Validation
  collection and the variant.calculated_price
  assignment stay in this pass, unchanged.
  2. Move both throw new MedusaError(...) blocks
  above the build pass, preserving precedence
  (variant errors before price errors).
  3. Add a second .map() over the resolved
  intermediates that builds PrepareLineItemDataInput
  and calls prepareLineItemData, with
  calculatedPriceSet added to the price-assignment
  guard.
  4. Add
  packages/core/core-flows/src/cart/workflows/__test
  s__/get-variants-and-items-with-prices.spec.ts.
  5. Add a changeset (@medusajs/core-flows: patch).

  - Its test sits in integration-tests/http/, which
  is where maintainer NicolasGorga explicitly asked
  for it (CHANGES_REQUESTED, Jul 6; author complied
  Jul 7). My unit-test placement is the exact thing
  that review pushed back on.
  - One changed line versus a ~60-line restructure,
  for the same user-visible outcome.
  
  ---
  Testing Strategy
  
  Unit Tests

  packages/core/core-flows/src/cart/workflows/__test
  s__/get-variants-and-items-with-prices.spec.ts —
  5/5 passing via npx jest src/cart:

  - [x] Test case 1: Happy path — a variant with a
  calculated price produces a line item carrying
  unit_price: 1000 and is_tax_inclusive: false
  (guards against regression in the reordering)
  - [x] Test case 2: Issue repro — a published
  variant with no calculated price rejects with
  type: invalid_data and message "Variants with IDs 
  variant_1 do not have a price", not a TypeError
  - [x] Test case 3: Multiple unpriced variants are
  aggregated into one message ("variant_1, 
  variant_3") while the priced variant_2 is skipped
  — proves the collect-all-then-throw behaviour
  survived the split
  - [x] Test case 4: Error precedence — a
  DRAFT-product variant that is also unpriced
  reports the variant error, not the price error
  - [x] Test case 5: An item carrying an explicit
  unit_price still bypasses the price requirement
  entirely (is_custom_price: true, unit_price: 500),
  so the fix doesn't over-reject custom-priced
  items

  Implementation detail worth recording: the
  workflow runtime serialises a thrown MedusaError
  into a plain object, so rejects.toThrow() reports
  "Received function did not throw" and silently
  fails to match. Assertions use
  rejects.toMatchObject({ type, message }) instead.
  This cost a debugging cycle and is not obvious
  from the existing specs.

  Integration Tests

  - [ ] Integration scenario 1: POST
  /store/carts/{id}/line-items for a variant with no
  price returns 400 — not written here. PR #15939
  already covers exactly this in integration-tests/h
  ttp/__tests__/cart/store/cart.spec.ts (+48 lines),
  added after a maintainer asked for it.
  - [ ] Integration scenario 2: The issue's actual
  multi-region repro (variant priced in eur, cart in
  xaf) — not covered by either change. Both
  #15939's test and mine use a variant with no
  prices at all, which hits the same code path but
  is a weaker reproduction of the reported scenario.

  Manual Testing

  None performed — no Medusa server or Postgres
  instance was started, so the 500→400 transition
  was not observed over HTTP. Verification was
  static and unit-level only:

  - npx jest src/cart → 2 suites, 14 tests passing
  (the new spec plus the pre-existing
  prepare-confirm-inventory-input.spec.ts)
  - npx tsc --noEmit -p tsconfig.json in
  packages/core/core-flows → exit 0
  - Read-through of prepare-line-item-data.ts
  confirming no residual 500 path: every
  calculated_price read there is optional-chained
  (lines 112, 119, 127), so an undefined price set
  can no longer produce a TypeError anywhere
  downstream

  The editor reports Cannot find name 'it'/'expect'
  on the new spec. This is pre-existing tsconfig
  noise for __tests__ in this package — the existing
  prepare-confirm-inventory-input.spec.ts shows the
  same diagnostics — and neither jest nor tsc
  --noEmit is affected.

  ---
  Implementation Notes

  Week [X] Progress

  (week number to fill in; work done 2026-07-20)

  Built: The validate-then-build split, the 5-case
  unit spec, and a changeset, committed to
  fix/cart-line-item-validate-before-build.

  Key finding — this is not an unclaimed issue. PR
  #15939 (zain-asif-dev, opened Jul 4) already fixes
  the reported bug with the one-line &&
  calculatedPriceSet guard plus an HTTP integration
  test, and is currently MERGEABLE. Comparing
  honestly, it is the better PR to land:

  - Its test sits in integration-tests/http/, which
  is where maintainer NicolasGorga explicitly asked
  for it (CHANGES_REQUESTED, Jul 6; author complied
  Jul 7). My unit-test placement is the exact thing
  that review pushed back on.
  - One changed line versus a ~60-line restructure,
  for the same user-visible outcome.

  Decision: do not open a competing PR. The branch
  is parked for a follow-up refactor once #15939
  merges, framed as structural hardening rather than
  a bugfix.

  Corrected claim: my initial justification for the
  restructure was that it also fixes the "Variant
  does not have a product" preemption. Checking all
  four call sites showed every one requests
  product.* fields, making that path unreachable
  today. The commit message states this is defensive
  so a reviewer isn't misled; the changeset was
  reworded from "return a 400 instead of a 500…" to
  "validate variants and prices before building cart
  line items" to avoid duplicating #15939's
  framing.

  Open items before this becomes a PR:
  1. Rebase after #15939 lands — both touch the same
  if (variant && !isCustomPrice) line, so a
  conflict is expected; resolution keeps the
  restructured version, which already subsumes their
  guard.
  2. Decide the test-location argument. The unit
  spec covers cases unreachable over HTTP (message
  aggregation, error precedence); either make that
  case to reviewers or drop the spec and ship the
  refactor alone.
  3. Timing risk: the issue was marked Stale on Jul
  20 with a 3-day close warning. Both it and #15939
  may be auto-closed before merge — worth a nudge on
  the PR.

---

## Implementation Notes

### Week [3] Progress

Write first pass at solution

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

- **Files modified:**
    - packages/core/core-flows/src/cart/workflows/ge
  t-variants-and-items-with-prices.ts (+33/−24) —
  split prepareVariantsAndItemsWithPricesStep into a
  validation pass and a build pass; added
  calculatedPriceSet to the price-assignment guard
    - packages/core/core-flows/src/cart/workflows/__
  tests__/get-variants-and-items-with-prices.spec.ts
  (+126, new) — 5-case regression spec
    - .changeset/olive-boxes-shave.md (+5, new) —
  @medusajs/core-flows: patch

- **Key commits:**
    - 5d59ff0909 — fix(core-flows): validate 
  variants and prices before building cart line 
  items (2026-07-20). Single commit; the branch is
  not split into steps.
- **Approach decisions:**

  - Restructure over the one-line guard. The issue
  proposes adding && calculatedPriceSet to the
  assignment branch, and PR #15939 implements
  exactly that. I chose the validate-then-build
  split instead because the collect-then-throw-later
  structure is the actual defect — the guard fixes
  today's dereference but leaves the map able to
  touch calculatedPriceSet before the error fires,
  so the same bug class returns with the next edit
  to that block. The narrow guard is applied too, as
  defense in depth. Cost of this decision: a
  ~60-line diff against a 1-line one for identical
  user-visible behaviour, which is a real argument
  against it and the main reason this branch is
  parked rather than submitted.

  - Conformed to an existing step rather than 
  inventing a shape. packages/core/core-flows/src/ca
  rt/steps/validate-variant-prices.ts is already a
  validation-only step that accumulates into
  priceNotFound and throws the byte-identical
  message with no construction interleaved. The
  restructure makes the inline logic match that
  separation, so the two now read the same way.

  - Preserved error precedence deliberately.
  Variant-not-found/unpublished still throws before
  missing-price. Reordering the passes could have
  silently flipped this, so it's pinned by test
  case 4.

  - Kept validation collection and 
  variant.calculated_price assignment in pass 1, 
  untouched. Only the build half moved. This keeps
  the diff reviewable as "code moved" rather than
  "code rewritten" and avoids touching the mutation
  that downstream steps depend on.

  - Changeset reworded from the bugfix framing.
  Originally "return a 400 instead of a 500…";
  changed to "validate variants and prices before
  building cart line items" so it doesn't duplicate
  #15939's changelog entry and read as a competing
  fix.

  - Unit tests over an HTTP integration test. Two of
  the five cases (message aggregation across
  multiple unpriced variants;
  variant-error-vs-price-error precedence) can't be
  provoked through a single HTTP request. This
  decision is knowingly against the maintainer steer
  recorded below and is the main open item.

---

## Pull Request

**PR Link:** https://github.com/medusajs/medusa/compare/develop...hhuang2088:medusa:fix/cart-line-item-validate-before-build?expand=1

**PR Description:**


   What
  
   Follow-up hardening for #15932. Restructures 
   prepareVariantsAndItemsWithPricesStep so variant
   and price validation completes before any line 
   item is built.

   This does not compete with #15939 — that PR is 
   the minimal fix for the reported 500 and should 
   land first. This is intended to rebase on top of
   it.
  
   Why
  
   prepareVariantsAndItemsWithPricesStep collected 
   validation failures into priceNotFound / 
   variantNotFoundOrPublished while building line 
   items in the same .map() pass, and only threw 
   once the pass finished. An item flagged as 
   unpriced went on to dereference 
   calculatedPriceSet.calculated_amount in the same
   iteration, raising a TypeError before the 
   intended MedusaError was ever reached.
  
   #15939 fixes the dereference. What remains is 
   the structure that allowed it: any future code 
   added inside that map that touches 
   calculatedPriceSet re-opens the same failure. 
   The step also already has a sibling of the 
   pattern in-tree — validateVariantPricesStep does
   validation only, with no construction 
   interleaved, and throws the identical message.
  
   Changes
  
   - First pass resolves each item's variant and 
   price set, collects validation failures, and 
   attaches variant.calculated_price (logic 
   unchanged, just no longer builds).
   - Both MedusaErrors now throw before the build 
   pass. Precedence is unchanged: variant errors 
   before price errors.
   - Second pass builds the line items, with 
   calculatedPriceSet in the assignment guard.
  
   No public API, type signature, or step 
   input/output shape changes.
  
   Secondary effect
  
   The reorder also stops prepareLineItemData's 
   generic "Variant does not have a product" from 
   preempting the step's specific "do not exist or 
   belong to a product that is not published". To 
   be clear about scope: that path is not reachable
   through any current caller — 
   create-order.ts:319, add-line-items.ts:163, 
   create-carts.ts:239 and add-to-cart.ts:180 all 
   request variant fields derived from 
   productVariantsFields, which includes product.*.
   It's defensive, not a bug being fixed.
  
   Testing
  
   Unit spec at packages/core/core-flows/src/cart/w
   orkflows/__tests__/get-variants-and-items-with-p
   rices.spec.ts, 5 cases: happy path, missing 
   price → invalid_data, multiple unpriced variants
   aggregated into one message, variant-error 
   precedence, and custom-price items still 
   bypassing the price requirement.
  
   Two of these (aggregation, precedence) can't be 
   provoked through a single HTTP request, which is
   why they're unit tests rather than an addition 
   to integration-tests/http/. Happy to move or 
   drop them if you'd rather keep everything in the
   HTTP suite — #15939 already covers the 400 
   end-to-end.
  
   npx jest src/cart → 14 passing. npx tsc --noEmit
   → clean.

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** Awaiting review

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
