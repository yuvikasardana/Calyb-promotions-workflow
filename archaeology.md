# Archaeology Log

This document records non-obvious behaviors, defaults, and semantic gaps discovered while reverse-engineering the Vendure Admin UI and GraphQL API.

These findings were identified through Network tab inspection and schema validation.

## Track A 

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


## Track B (Shipping)


### Country IDs must be discovered via the `countries` query
Countries cannot be added to a zone using name/code directly.  
The UI searches countries by firing `countries(options)` with filter `{ name contains, code contains }`, then uses the returned `id` in `addMembersToZone`.

 While most queries are ignored as they serve the purpose of UI refresh , adding Australia/New Zealand is inherently a discovery + mutation process which requires searching for the country's ids using queries.



###  Flat rate shipping uses default calculator with required args
The shipping method uses:
- `calculator.code = "default-shipping-calculator"`
- required args include:
  - `rate = "1500"` (represents $15)
  - `includesTax = "auto"`
  - `taxRate = "0"`

Rate must be converted to minor units and additional calculator args must be included.

### Shipping Method can be created without a name 
 The only required fields to be filled in shipping method are fulfillment handler, default shipping eligibility checker and shipping calculator. 
 If the name or code field is kept empty the API is simply called with an empty string input for both of these fields. 
 It is likely that these fields are for user convenience as a shipping method is recognized by the system via its id.
 
###  Only default shipping eligibility checker is available in demo instance
The shipping method checker is:

`default-shipping-eligibility-checker`

with only one argument observed:

- `orderMinimum`

No zone/destination-based eligibility arguments (e.g., zoneIds) are available.
 “Shipping method applies only to Oceania zone” cannot be implemented using only this demo instance configuration.

###  Zone-only shipping requirement becomes a constraint gap
User intent requires zone-based restriction, but the observed checker configuration does not support zone filtering.

**Impact:** this semantic gap must be documented explicitly instead of being guessed or hardcoded.
## Summary

These discoveries informed the final workflow representation and ensured it reflects actual system behavior rather than UI assumptions.
