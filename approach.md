# Approach

This assignment is a workflow reverse-engineering problem.  
The objective was to identify how user intent expressed through the Admin UI maps to underlying API-level state changes, and to represent that mapping in a reusable, machine-readable structure.
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
1. Find out the order of execution of the UI steps required to create the expected promotion. By discovering the vendure demo site the  sequence of UI actions was :<br>
i. **Create Customer Group** <br>
    Rightclick customer -> right click customer group -> input(customer group name = VIP) ->right click add customer group <br> 
ii. **Create Promotion** <br>
    rightclick marketing -> rightclick promotions-> input(promotion_name = Black Friday) -> input (coupon_code = BF2024) -> add condition1 = customer bill greater than input = 100
add condition 2 -> customer part of select = VIP customer group -> toggle enabled -> click create 

2. Trigger a **single UI action** (e.g., create promotion, add condition).
3. Inspect the corresponding **GraphQL mutation** in the Network tab. Since only mutations actually bring about a change in the system state ,   the lookup queries were excluded from consideration 
3. Identify the mutation name and its input payload.
    I identified two mutations in this case 
    i. createCustomerGroup - triggered on clicking add customer group
    ii. createPromotion - triggered on clicking create in the promotion section
4. Map visible UI fields to mutation inputs by observing:
   - argument names
   - argument values
   - default or hidden fields required by the API
5. Validate field meaning using the GraphQL input schema definitions in the documentation.
6. Prefer **observed payload values** over inferred meanings from UI labels.
  For example: 
     - UI label “Order total above $100” maps to the `minimum_order_amount` condition with `amount = 10000`.
    - UI selection of a customer group maps to a required `customerGroupId`, not the group name.

This heuristic ensures mappings are grounded in **actual system behavior**, and not derived from only the UI text/inputs.

## Dependency Discovery

Dependencies were identified by tracking **IDs returned from mutations** and observing where those IDs were required as inputs in subsequent mutations.

For example:
- `createCustomerGroup` returns a `customer_group_id`
- The promotion condition `customer_group` requires that ID as `customerGroupId`

Dependencies are modeled explicitly using ID references even when the UI invovles looking up for the names, ensuring deterministic execution and avoiding hard-coded values.

## Structural Standardization

The workflow is represented as a **declarative, dependency-aware intermediate representation (IR)** encoded in JSON.

Design principles:
- Each **state-changing mutation** is represented as the step
- Inputs include all the required API fields
- Outputs expose only data needed for downstream dependencies (typically entity IDs)
- Queries are excluded, as they do not modify persistent state
- Execution order is implied through data dependencies rather than explicit control flow

This structure is minimal, machine-readable, and suitable for automated execution.


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

The resulting workflow model is reusable, dependency-safe, and generalizable across SaaS platforms.
