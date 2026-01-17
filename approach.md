# Approach

This assignment is a workflow reverse-engineering problem.  
The objective was to identify how user intent expressed through the Admin UI maps to underlying API-level state changes, and to represent that mapping in a reusable, machine-readable structure.

## Track A: The "Promotion" Challenge

The Request: "Create a Black Friday coupon code 'BF2024'. It should apply a 15% discount, but only if the order total is above $100 AND the customer is in the 'VIP' group."

The workflow focuses exclusively on **state-changing operations**, their required inputs, returned identifiers, and inter-step dependencies.


## Assumptions

I made the following assumptions to complete the assignment:

- A value of `0` for `usageLimit` and `perCustomerUsageLimit` represents **unlimited usage**, as observed in default UI behavior.
- `startsAt` and `endsAt` act as optional temporal guards; `null` values indicate no time-based restriction.
- UI-driven lookup queries (e.g., searching customer groups by name) are excluded from the workflow, since automation can directly reference entity IDs returned by earlier steps.
- The required customer group does not already exist and its creation is included as part of execution
- Population of customer groups (i.e., assigning customers to the VIP group) is treated as business data and is intentionally out of scope.



## Heuristic: Mapping 

UI-to-API mapping was performed using **network tab inspection** and validating through the GraphQL API (Admin API) . 

I followed the following process for identifying the mapping:
### 1. Identify the UI flow (human path)
First, I determined the UI sequence required to create the expected promotion:

#### i. Create Customer Group
**UI Path:** Customer → Customer Groups → Add Customer Group<br>
Input: customer group name = `VIP` → Create

#### ii. Create Promotion
**UI Path:**  Marketing → Promotions → Add Promotion  
Input: promotion name = `Black Friday`  
Input: coupon code = `BF2024`  
Add Condition 1: Order total greater than `100`  
Add Condition 2: Customer is in `VIP` customer group  
Add Action: 15% discount  
Toggle enabled → Create

### 2. Observe the underlying API operations
To map each UI action to API calls:

- I triggered **one UI action at a time** (create entity, add condition, add action).
- I inspected the corresponding GraphQL request in the Network tab.
- Since only **mutations** change system state, they form the main executable steps of the workflow.

In this case, the main mutations identified were:

- `createCustomerGroup` (triggered on clicking create customer group)
- `createPromotion` (triggered on clicking create promotion)

An important query fired was for discovering the pre-existing customer groups when the condition for customer group is added. For this AI workflow the id for the customer group is taken from the output of the previous step. However in the case that the customer group is pre-existing this query could be important for finding out the required customer group's id.


### 3. Map UI fields to API inputs
I mapped visible UI fields to mutation inputs by observing:

- argument names
- argument values
- default or hidden fields required by the API

I validated field meaning using GraphQL schema definitions from the documentation and mutation payloads.

### 4. Prefer observed payload values over inferred UI meanings
Whenever the UI label and API representation differed, the API payload values were treated as the source of truth.

Examples:
- UI label “Order total above $100” maps to the `minimum_order_amount` condition with `amount = 10000`.
- UI selection of a customer group maps to a required `customerGroupId`, not the group name.


This heuristic ensures mappings are grounded in **actual system behavior**, and not derived from only the UI text/inputs.

## Dependency Discovery

Dependencies were identified by tracking **IDs returned from mutations** and observing where those IDs were required as inputs in subsequent mutations.

For example:
- `createCustomerGroup` returns a `customer_group_id`
- The promotion condition `customer_group` requires that ID as `customerGroupId`

Dependencies are modeled explicitly using ID references even when the UI involves looking up for the names, ensuring deterministic execution and avoiding hard-coded values.

---

## Structural Standardization

The workflow is represented as a **declarative, dependency-aware intermediate representation (IR)** encoded in JSON.

Design principles:
- Each **state-changing mutation** is represented as the step
- Inputs include all the required API fields
- Outputs expose only data needed for downstream dependencies (typically entity IDs)
- Queries are excluded, as they do not modify persistent state
- Execution order is implied through data dependencies rather than explicit control flow

This structure is minimal, machine-readable, and suitable for automated execution.

---
## Track B: The "Global" Challenge

The Request: "Set up a new shipping zone for 'Oceania'. Add 'Australia' and 'New Zealand' to this zone. Then, configure a 'Flat Rate' shipping method of $15 that only applies to this zone."

As with Track A, the workflow focuses on state-changing operations, their required inputs, returned identifiers, and explicit dependencies. Read-only queries are included only where necessary for identifier discovery or to document UI-driven behavior.

### Assumptions (Track B)

I made the following assumptions for Track B:

- Country codes for Australia and New Zealand are available in the system's country list and can be referenced by their standard identifiers.
- The Vendure demo instance exposes only the default shipping eligibility checker.
- Restricting a shipping method to a zone is considered part of user intent, even if it cannot be fully expressed in the current UI/API configuration.
- Default values are chosen for the fields that were not defined by the user. Example: minimum order value = $0

### Heuristic: Mapping (Track B)
Just like Track A , UI-to-API mapping was performed using **network tab inspection** and validating through the GraphQL API (Admin API) . 

I followed the following process for identifying the mapping:
### 1. Identify the UI flow (human path)
First, I determined the UI sequence required to fulfil the user request:
 #### i. Create Shipping Zone
**UI Path:** 
- Settings → Zones → New Zone
- Input: zone name = 'Oceania' → Create

#### ii. Add Countries to Zone
**UI Path:**
- Zone page → Select Country
- Search and select:
   - Australia
   - New Zealand

#### iii. Create Shipping Method
Settings → Shipping Methods → New Shipping Method<br>
Input:<br>
- Name: Flat Rate
- Calculator: Flat Rate → $15
- Fulfillment Handler: Manual
- Eligibility Checker: Default
→ Create

### 2. Observe underlying API Operations
Using Network tab inspection, the following operations were identified:
Mutations (state-changing)
- createZone: fired on clicking create in the zone section

- addMembersToZone: triggered each time a country is to be added to a Zone

- createShippingMethod: triggered on creating the shipping method

Queries (discovery / UI-driven)

- CountryList
Fired repeatedly while typing in the country selector to resolve { id, name, code }

Additional queries such as  zoneMembers were triggered by the UI for refresh and verification but do not affect system state.

### 3. Map UI fields to API inputs
Zone creation maps directly to createZone(name, memberIds)
Country selection requires:
- querying countries
- extracting the selected `country.id`
- passing it as `memberId`s to `addMembersToZone`

Flat Rate shipping maps to:
- `default-shipping-calculator`
- `rate = 1500`
- `includesTax = auto`
- `taxRate = 0`
Eligibility configuration maps to:
- `default-shipping-eligibility-checker`
- only supported argument: `orderMinimum`


### 4. Semantic Gap Identification
The user intent includes:
“Shipping method should apply only to the Oceania zone”
However, inspection of the createShippingMethod payload shows:
-Only default-shipping-eligibility-checker is available
-The checker supports only orderMinimum
-No zone-, country-, or destination-based arguments are exposed
This reveals a semantic gap:
-User intent: zone-based restriction
-API capability (demo instance): order-minimum-based restriction only
This gap is explicitly documented in the workflow as a constraint.


## Dependency Discovery (Track B)

Dependencies were identified by tracking IDs returned from mutations and queries:

- `createZone`  return a  zone_id

- to add countries to a xzone you need `country_id` obtained from the query `countryList`

addMembersToZone depends on:
-zone_id
-resolved country_id

createShippingMethod is currently  independent of the zone due to eligibility checker limitations
However in a generalized setting createShippingMethod would require a zone id to bind it to a particualr zone.

## Structural Standardization (Track B)

The workflow is encoded as a dependency-aware intermediate representation:<br>

-Mutations represent executable state transitions

-Queries represent identifier discovery

-Outputs expose only required identifiers

-The unmet zone-restriction requirement is modeled explicitly as a constraint_gap

**This preserves correctness while remaining machine-readable and executable where possible**

## Universal Generalization 

The same algorithm can be applied to platforms such as Salesforce, Jira, or HubSpot:

1. Start from user intent and identify the required end state.
2. Decompose intent into UI actions to understand required entities and ordering.
3. Observe state-changing API operations (mutations / POST requests).
4. Ignore read-only queries used for UI convenience.
5. Extract stable identifiers returned by each operation.
6. Model the workflow as a dependency graph of state transitions.
7. Normalize semantic gaps (units, enums, sentinel values).

Because most SaaS systems expose similar primitives (rules, conditions, actions, identifiers), this approach remains platform-agnostic despite differences in UI or API structure.

---

