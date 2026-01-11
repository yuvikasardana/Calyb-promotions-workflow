# Archaeology Log

This document records non-obvious behaviors, defaults, and semantic gaps discovered while reverse-engineering the Vendure Admin UI and GraphQL API.

These findings were identified through Network tab inspection and schema validation.


### Monetary Values Use Minor Units

Prices and monetary thresholds entered in the UI (e.g. `$100`) are converted to **integer minor units** (`10000 cents`) in API payloads.

This conversion is not visible in the UI and must be handled explicitly when modeling workflows.


### Promotions Are Governed by Lifecycle State

Promotions have an explicit lifecycle controlled by the `enabled` flag.

- A promotion can exist in a disabled state.
- The Admin UI allows promotions to be created with `enabled = true`.
- `enabled`, `startsAt`, and `endsAt` together determine promotion if a promotion can apply or not.

A promotion with `enabled = true` and `startsAt / endsAt = null` is immediately and indefinitely active.


### Dates are Optional

The fields `startsAt` and `endsAt` are optional in `createPromotion`.

- `null` values indicate no time-based restriction.
- The UI does not require these fields to be set for a valid promotion.

This implies that time-based activation is enforced only when explicitly configured.


### Conditions and Actions Are Nested, Not Separate Mutations

Promotion conditions and actions are submitted as nested `ConfigurableOperationInput` objects within the `createPromotion` mutation.

the UI makes it look like you are “adding conditions” and “adding actions” as separate steps but the API never creates them independently


### UI Lookups Use Queries, Not Workflow Steps

When selecting a customer group in the promotion UI, the Admin UI performs lookup queries by group name.

These queries:
- do not modify system state
- exist purely for UI convenience
- are unnecessary for automation

The API ultimately requires a stable `customerGroupId` for the promotion to be created, not the group nameas suggested by the UI.


### Required Fields Not Exposed in the UI

Some API-required fields are not explicitly surfaced in the UI and are sent with default empty values, including:

- `customFields`
- `customerIds` (for customer group creation)

These fields must still be included in mutation inputs for successful execution.


### Localization Is Mandatory but not Necessary

Entities such as promotions require `translations` input, including language code and name.
This requirement supports localization but does not affect workflow logic or dependencies and is treated as static metadata.


## Summary

These discoveries informed the final workflow representation and ensured it reflects actual system behavior rather than UI assumptions.
