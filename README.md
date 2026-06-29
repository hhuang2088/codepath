# Contribution 1: [Bug]: Error trying to sort orders by: Total / Fulfillment status / Payment status


**Contribution Number:** 1
**Student:** [Henry Huang
**Issue:** https://github.com/medusajs/medusa/issues/15353
**Status:** Phase 4 Complete

---

## Why I Chose This Issue

This issue interests me because I like to insure web apps are in the best shape they can be. I enjoy building and fixing software across the full stack. This app happens to be fintech related, which I've done work on before.

---

## Understanding the Issue

### Problem Description

The issue is that at the moment, an error returns when the user tries to sort by total, fulfillment_status, or the payment_status.

### Expected Behavior

There should be an allow-list implemented so that orders can only be sorted by allowed fields, and attempted by fields not in the allow-list shouldn't return an error.

### Current Behavior

When the user tries to sort by these fields, an error is returned.

### Affected Components

The backend is affected, as that's the part that returns orders sorted by different fields. Storefronts, or custom clients, are affected because where they would ask for orders to be sorted by these fields, they are getting errors.

---

## Reproduction Process

### Environment Setup

[Notes on setting up your local development environment - challenges you faced, how you solved them]

### Steps to Reproduce

1. Set up project by running "Fetch https://docs.medusajs.com/start and create an ecommerce store with Medusa Cloud" on preferred LLM.
2. cd to ./app/backend from the project directory
3. run `npx medusa user -e you@example.com -p yourpassword` to create an admin user
4. Seed the database with sample orders, making sure that fields like Total, Fulfillment status, and Payment status can be derived
4. run the following in ./apps/backend from the project directory
  ```
  nvm use 22
  npm run dev
  ```
5. By default, the admin console should be running on http://localhost:9000/app
6. Sign on to admin account on admin console
7. Orders are displayed in http://localhost:9000/app/orders
8. Sort the orders by entering either "total", "fulfillment_status", or "payment_status" in place of "order_by" in the following url http://localhost:9000/app/orders?order=order_by.
9. With any of these fields, it should return an error.

### Reproduction Evidence

Error: An unknown error occurred.

Error: An unknown error occurred.
    at http://localhost:9000/app/@fs/Users/henryhuang/Desktop/project/my-medusa-store/apps/backend/node_modules/.vite/deps/chunk-KZFOMZ74.js?v=086c15c4:9209:11
    at Generator.next (<anonymous>)
    at fulfilled (http://localhost:9000/app/@fs/Users/henryhuang/Desktop/project/my-medusa-store/apps/backend/node_modules/.vite/deps/chunk-KZFOMZ74.js?v=086c15c4:9075:24)

---

## Solution Approach

### Analysis

The issue is that it's returning an error when attempting to sort orders by a field that isn't supported for sorting

### Proposed Solution

Create an allow-list of sortable columns for orders, so if someone tries to sort by an unsupported field, they get a more informative error.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** The problem is that the api is returning an unclear error message when trying to sort by an unsupported field for sorting

**Match:** Orders actually already have a nonFilterableFields feature. This allow-list feature would model itself closely to that feature

**Plan:** [Step-by-step implementation plan]
          1. Add an allow-list of sortable fields ("id", "display_id", "status", "email", "currency_code", "created_at", "updated_at")
          2. Add a middleware function that enforces this allow-list when trying to sort by order
          3. Set it so that client gets a 400 error when trying to sort by field not in allow-list
          4. Update tests to include this feature

**Implement:** https://github.com/hhuang2088/medusa/tree/fix/add_and_enforce_order_sort_allow_list

**Review:** [ ] Replace the @medusajs/* dependencies and devDependencies in my test project's package.json to point to the corresponding local packages in my forked Medusa repository.
            [ ] Every time you make a change in the forked Medusa repository, you need to build the packages where the modifications took place with yarn build. Some packages have a watch script, so you can execute yarn watch once and it will automatically build on changes:
            `yarn build  # or yarn watch `
            [ ] After building changes in the forked medusa repository, run the following command in the test project to regenerate the node_modules directory with the newly built contents from the previous step:
            ```
            # For npm/yarn
              rm -R node_modules && yarn && yarn dev
            ```

**Evaluate:** When attempting to sort orders by an unsupported field, I will verify that it returns a 400 status error with a good description, instead of a 500 status error.

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [Y] Progress

I wrote an allowlist for Orders in the Medusa platform, and set things up so that if the client attempts to sort by fields outside not listed in this allowlist, it returns a 400 error with a helpful description, rather than a 500 error with an error log as it was before. I also wrote integration and unit tests written to cover these changes as well.

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [
  ".changeset/order-sort-allow-list.md",
  "integration-tests/http/__tests__/order/admin/order.spec.ts",
  "packages/core/framework/src/http/utils/__tests__/get-query-config.spec.ts",
  "packages/core/framework/src/http/utils/get-query-config.ts",
  "packages/core/types/src/common/common.ts",
  "packages/medusa/src/api/admin/orders/query-config.ts",
  ]
- **Key commits:** [
  "https://github.com/medusajs/medusa/pull/15809/changes/7e4ad1e100a2b417dbec2a5ebe7fe3f1c4e68de0",
  "https://github.com/medusajs/medusa/pull/15809/changes/e52d5a953f7f67739b7b17a14c6cb52d87828fce"
]
- **Approach decisions:**
Overall, I chose the approach of implementing an allow list of sortable fields for Orders, as opposed to setting things up so users can sort by computed values such as Order, Payment Status, and/or Fulfillment Status, because to do so seemed like a more dramatic change from the conventions already set forth by the codebase. On the other hand, there's already presedence set for implementing a sortable fields allow-list in the codebase, and that seemed like a less dramatic change without overstepping my bounds as an outside contributer. Whether Orders should be sortable by computed fields ought to be a decision made from internal contributers first, as doing so requires more computational resources.

---

## Pull Request

**PR Link:** https://github.com/medusajs/medusa/pull/15809

**PR Description:**

What
Sorting the admin orders list (GET /admin/orders) by a computed field —
total, payment_status, or fulfillment_status — currently throws a 500,
because these are not persisted columns on the order table (MikroORM:
"Trying to order by not existing property Order.total"). This PR rejects any
non-sortable sort field with a clean 400.

Why
The orders list query config never set a sort allow list, so the order query
param flowed unchecked to the data layer. This mirrors the filtering fix in
#15262 (nonFilterableFields), but for sorting — the counterpart was missing.

How
Added an optional allowedOrderBy to QueryConfig (@medusajs/types).
Unlike allowed, it governs only sorting, not field selection — so a computed
field like total stays selectable while being rejected as a sort key.
Enforced it in prepareListQuery (@medusajs/framework): the order guard now
uses allowedOrderBy when present, falling back to allowed so existing
routes (customers, promotions) are unchanged.
Defined allowedAdminOrderSortFields (display_id, created_at,
updated_at) on the orders list config — matching the sort options the admin
dashboard offers.
Testing
Unit (packages/core/framework/src/http/utils/__tests__/get-query-config.spec.ts):
added cases for prepareListQuery — rejecting a sort field absent from
allowedOrderBy, allowing one present in it, preferring allowedOrderBy over
allowed (sort decoupled from field selection), and sorting by a field present
only in allowedOrderBy.
Integration (integration-tests/http/__tests__/order/admin/order.spec.ts):
added cases under GET /admin/orders — 400 when sorting by total /
-payment_status / fulfillment_status; 200 when sorting by -created_at.
The pre-existing fields=id,total&order=-created_at test still passes, proving
field selection is unaffected.
Closes #15353

🤖 Generated with Claude Code

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** Awaiting review

---

## Learnings & Reflections

### Technical Skills Gained

I learned to really take in the context of the app before rushing towards a particular solution. 
For the bug reported. It was implied that the solution was to have it so that these Orders could 
be sorted by Payment Status, Fulfillment Status, and Total. However, what I found from digging 
into the code was that these were computed fields. Digging further into the code, precedence is 
already set for an allow list for filterable fields for Orders. It then becomes reasonable to 
implement a similar pattern for sortable columns.

### Challenges Overcome

The hardest part was really to hold myself back a bit from rushing into devising a solution
before really getting the full context of the problem.

### What I'd Do Differently Next Time

Next time, I would work towards being more thorough in the preliminary steps before I even begin
writing code. This contribution has made me think of the old adage "measure twice, cut once".
Had I not paused to take in the full context, that probably would have led to wasted development time attempting
a non-optimal solution.

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
