# Contribution 1: [Bug]: Error trying to sort orders by: Total / Fulfillment status / Payment status


**Contribution Number:** 1
**Student:** [Henry Huang
**Issue:** https://github.com/medusajs/medusa/issues/15353
**Status:** Phase 2 Complete

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

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

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
