# Earn Studio — Component Specification

## What Is This Component?

Earn Studio is a **visual mapping builder** for configuring how customers earn loyalty currency (points and tickets) from purchases. It provides an admin interface for merchants to define:

1. **Earn Factors** — Rules that determine how much currency a customer earns (e.g. "Spend ฿100 = 1 point" or "3x multiplier on shoes")
2. **Earn Conditions** — Qualifying criteria that gate when a factor applies (e.g. "only for Gold tier members" or "only for Nike products with ≥50 units purchased")
3. **The link between them** — Which earn factors use which condition groups, visualized as connection lines

The component renders as a two-column layout with SVG connection lines between them, similar to the [Hookdeck connection visualization](https://hookdeck.com/) pattern.

---

## Business Context

This component is part of the **Currency Rewards Engine** in the Supabase CRM platform. The engine supports:

- **Points** — fungible loyalty currency (all points are interchangeable)
- **Tickets** — non-fungible special currencies tracked per ticket type (raffle tickets, VIP passes, store credit, etc.)

Merchants use this component to configure the earning rules that the calculation engine (`calc_currency_for_transaction`) evaluates in real-time when a customer makes a purchase.

### Key Domain Concepts

| Concept | What It Is |
|---------|------------|
| **Earn Factor Group** | Container that bundles related earn factors. Defines shared properties: stackable (do multipliers combine?), window period, active status. |
| **Earn Factor** | Individual earning rule. Two types: **Rate** (converts spend to currency, e.g. ฿100 = 1 pt) and **Multiplier** (increases earnings, e.g. 3x points). |
| **Earn Conditions Group** | Reusable eligibility gate. Groups one or more conditions together. Can be shared across multiple earn factors. |
| **Earn Condition** | Single qualifying criterion: an entity type (product, tier, persona, etc.) + entity IDs + operator (OR/AND/EACH) + optional thresholds. |
| **Connection** | The `earn_conditions_group_id` FK on `earn_factor` that links a factor to its conditions. |

---

## Component Layout

```
┌─────────────────────┐     ┌─────────────────────┐
│  Earn Factor Groups  │     │ Earn Conditions      │
│  ─────────────────── │     │ Groups               │
│                      │     │ ─────────────────── │
│ ┌──────────────────┐ │     │ ┌──────────────────┐ │
│ │ Standard Earning │ │     │ │ Tier Perks    ●──┼─┤
│ │ Rule      [+] ✎ ▾│ │     │ │   [+] ✎ ▾       │ │
│ │ ┌──────────────┐ │ │     │ │ ┌──────────────┐ │ │
│ │ │★ Points Rate │─┼─┼──●──┼─│ │ Type │ Items  │ │ │
│ │ └──────────────┘ │ │     │ │ │ Tier │  32    │ │ │
│ │ ┌──────────────┐ │ │     │ │ │ Prod │  32    │ │ │
│ │ │◆ Points Mult │─┼─┼──●──┼─│ └──────────────┘ │ │
│ │ └──────────────┘ │ │     │ └──────────────────┘ │
│ └──────────────────┘ │     │                      │
│                      │     │ ┌──────────────────┐ │
│ ┌──────────────────┐ │     │ │ Product Picks ●──┼─┤
│ │ Tier Bonus Rule  │ │     │ └──────────────────┘ │
│ └──────────────────┘ │     │                      │
└─────────────────────┘     └─────────────────────┘
                  ●────SVG lines────●
```

### Duplication Rule

If a single Earn Conditions Group is linked by multiple Earn Factors, the right column shows **multiple instances** of that group card — one per connection. This avoids line crossings and makes the mapping visually clear (Hookdeck pattern).

---

## How The Component Calls APIs

All API calls go directly to Supabase via `fetch()` — no Supabase JS client library. The API layer is in `src/useApi.js`.

### Authentication

Every request includes two headers:

```
apikey: {supabaseAnonKey}          ← public anon key (hardcoded default)
Authorization: Bearer {authToken}   ← admin user JWT (bound from WeWeb auth context)
```

The `authToken` prop must be bound in WeWeb to the current admin user's JWT from the Supabase auth plugin. Without it, RPC calls will fail with 401. The anon key is safe to hardcode.

### Base URL

```
https://wkevmsedchftztoolkmi.supabase.co
```

Configurable via the `supabaseUrl` prop.

### API Call Patterns

The component uses two patterns:

1. **RPC calls** — `POST /rest/v1/rpc/{function_name}` with JSON body
2. **REST auto-CRUD** — `GET /rest/v1/{table_name}?query` for simple listing

---

## API Reference — Every Call The Component Makes

### On Mount (Initial Load)

When the component mounts and `authToken` is present, it calls `loadAll()` which fires three parallel requests:

#### 1. Load Earn Factor Groups

```
GET /rest/v1/earn_factor_group?select=id,name,stackable,window_start,window_end,active_status,created_at&order=created_at.desc
```

Returns an array of group rows. Then for **each group**, fetches its factors:

```
POST /rest/v1/rpc/bff_get_earn_factors_by_group
Body: { "p_earn_factor_group_id": "<group_uuid>" }
```

**Returns:** Array of factor objects, each with `id`, `earn_factor_type`, `earn_factor_amount`, `target_currency`, `target_entity_id`, `target_entity_name`, `public`, `active_status`, `window_start`, `window_end`, `window_end_ttl_days`, `has_time_conditions`, `earn_conditions_group_id`, `earn_conditions_group_name`, `conditions` (preview of linked conditions).

#### 2. Load Earn Condition Groups

```
POST /rest/v1/rpc/bff_get_all_earn_conditions_groups
Body: {}
```

**Returns:** Array of condition group objects with `id`, `name`, `conditions` (array of `{ id, filter_type, filter_ids, created_at }`), `conditions_count`.

Then for **each group**, fetches full details (including threshold info):

```
POST /rest/v1/rpc/bff_get_earn_conditions_group
Body: { "p_mode": "edit", "p_group_id": "<group_uuid>" }
```

**Returns:** Object with `id`, `name`, `merchant_id`, `conditions` array — each condition has `id`, `entity`, `entity_ids`, `threshold_unit`, `min_threshold`, `max_threshold`, `apply_to_excess_only`, `operator`.

#### 3. Load Entity Options (for dropdowns)

```
POST /rest/v1/rpc/get_all_entity_options
Body: {}
```

**Returns:** Array of `{ entity_type, id, name }` for all products, SKUs, brands, categories, stores, store attribute sets, tiers, and personas belonging to the merchant. Filtered client-side by selected entity type.

---

### Create Earn Factor Group (Modal)

When the user clicks "Create" above the left column and fills out the modal:

```
POST /rest/v1/rpc/bff_upsert_earn_factor_group
Body: {
  "p_group_data": {
    "name": "Standard Earning Rule",
    "stackable": true,
    "window_start": "2026-03-01T00:00:00.000Z",  // or null
    "window_end": "2026-06-30T00:00:00.000Z",     // or null
    "factors": []
  }
}
```

**Returns:** `{ success, code, title, description, group_id, group_created, factors_created, ... }`

After success → reloads all factor groups, emits `earn-factor-group-saved` trigger event.

---

### Add / Edit Earn Factor (Sidebar Panel)

When the user edits a factor in the sidebar and clicks Save, the component:

1. **Fetches current group state** to get all existing factors:
   ```
   POST /rest/v1/rpc/bff_get_earn_factor_group_details
   Body: { "p_mode": "edit", "p_earn_factor_group_id": "<group_uuid>" }
   ```

2. **Merges the edited factor** into the factors array (replaces by ID, or appends if new)

3. **Upserts the entire group** with the updated factors array:
   ```
   POST /rest/v1/rpc/bff_upsert_earn_factor_group
   Body: {
     "p_group_data": {
       "id": "<group_uuid>",
       "factors": [
         {
           "id": "<factor_uuid>",          // null for new
           "earn_factor_type": "rate",      // or "multiplier"
           "earn_factor_amount": 100,
           "target_currency": "points",     // or "ticket"
           "target_entity_id": null,        // ticket_type UUID if ticket
           "public": true,
           "window_start": "2026-03-01T00:00:00.000Z",
           "window_end": null,
           "window_end_ttl_days": 30,
           "active_status": true,
           "has_time_conditions": false,
           "earn_conditions_group_id": "<condition_group_uuid>"  // the connection
         },
         // ... other factors in the group
       ]
     }
   }
   ```

**Important:** The upsert function handles create/update/delete of factors atomically — factors not in the array get deleted, existing ones get updated, new ones get inserted.

After success → reloads factor groups, closes panel, updates connection lines, emits `earn-factor-saved` trigger event.

---

### Create / Edit Earn Condition Group (Sidebar Panel)

When the user edits a condition group in the sidebar and clicks Save:

```
POST /rest/v1/rpc/bff_upsert_earn_conditions_group
Body: {
  "p_config": {
    "id": "<group_uuid>",       // null for new
    "name": "Tier Perks",
    "conditions": [
      {
        "id": "<condition_uuid>",         // null for new
        "entity": "tier",                  // enum: product_product, product_sku, product_brand, product_category, store, store_attribute_set, tier, persona
        "entity_ids": ["<uuid1>", "<uuid2>"],
        "operator": "OR",                  // OR | AND | EACH
        "threshold_unit": "amount",        // null | amount | quantity_primary | quantity_secondary
        "min_threshold": 5000,             // or null
        "max_threshold": 50000,            // or null
        "apply_to_excess_only": false
      },
      {
        "entity": "product_category",
        "entity_ids": ["<shoes_cat_uuid>"],
        "operator": "OR",
        "threshold_unit": null,
        "min_threshold": null,
        "max_threshold": null,
        "apply_to_excess_only": false
      }
    ]
  }
}
```

**Returns:** `{ success, code, title, description, group: { id, name, action }, conditions: { inserted, updated, deleted, total } }`

Conditions not in the array get deleted. After success → reloads condition groups, closes panel, emits `earn-condition-group-saved` trigger event.

---

### Link Factor to Condition Group (Connect Popup)

When the user hovers a factor row and clicks the "+" button, then selects a condition group from the popup:

1. **Fetches current group state:**
   ```
   POST /rest/v1/rpc/bff_get_earn_factor_group_details
   Body: { "p_mode": "edit", "p_earn_factor_group_id": "<group_uuid>" }
   ```

2. **Updates the factor's `earn_conditions_group_id`** in the factors array

3. **Upserts the group:**
   ```
   POST /rest/v1/rpc/bff_upsert_earn_factor_group
   Body: {
     "p_group_data": {
       "id": "<group_uuid>",
       "factors": [
         { "id": "<factor_uuid>", "earn_conditions_group_id": "<selected_condition_group_uuid>", ... },
         // ... other factors unchanged
       ]
     }
   }
   ```

After success → reloads factor groups, updates SVG connection lines, emits `connection-changed` trigger event.

This is the same API call as editing a factor — the connection is stored as the `earn_conditions_group_id` field on the `earn_factor` row.

---

## Component Properties (WeWeb Config)

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `supabaseUrl` | Text | `https://wkevmsedchftztoolkmi.supabase.co` | Supabase project URL |
| `supabaseAnonKey` | Text | *(hardcoded CRM anon key)* | Public API key for `apikey` header |
| `authToken` | Text | *(empty — must bind)* | Admin user JWT from Supabase auth session |
| `leftColumnWidth` | Length | `420px` | Width of earn factor groups column |
| `rightColumnWidth` | Length | `420px` | Width of earn condition groups column |
| `connectionLineColor` | Color | `#C9CCCF` | Default color of SVG connection lines |
| `connectionLineActiveColor` | Color | `#005BD3` | Color when hovering a connection line |
| `configPanelWidth` | Length | `380px` | Width of sidebar config panels |

---

## Trigger Events (emitted to WeWeb)

| Event Name | When Fired | Payload |
|------------|-----------|---------|
| `data-loaded` | After initial data load completes | `{ factorGroupCount, conditionGroupCount }` |
| `earn-factor-group-saved` | After creating/updating a factor group | `{ groupId, groupName, action }` |
| `earn-factor-saved` | After creating/updating an earn factor | `{ factorId, factorType, groupId }` |
| `earn-condition-group-saved` | After creating/updating a condition group | `{ groupId, groupName, action }` |
| `connection-changed` | After linking/unlinking a factor to a condition group | `{ factorId, conditionGroupId, action }` |
| `error` | On any API failure | `{ message, code }` |

---

## Actions (callable from WeWeb workflows)

| Action | Description |
|--------|-------------|
| `refreshData` | Reloads all earn factor groups, factors, and condition groups from the database |
| `closePanel` | Closes any open sidebar config panel |

---

## File Structure

```
src/
├── wwElement.vue                        # Main orchestrator
│   - Renders two-column layout
│   - Manages all state (data, UI, panels)
│   - Computes displayedConditionGroups with duplication rule
│   - Handles all save/connect flows
│   - Draws SVG connection lines between factor rows and condition cards
│
├── useApi.js                            # API layer
│   - getHeaders() builds auth headers from props
│   - rpc() for POST /rest/v1/rpc/{fn} calls
│   - restGet() for GET /rest/v1/{table} calls
│   - Named methods for each endpoint
│
└── components/
    ├── EarnFactorGroupCard.vue          # Left column card
    │   - Collapsed: group name, factor count badge, "+ Add earn factor", edit, chevron
    │   - Expanded: list of factor rows with type icon, name, meta, multiplier badge
    │   - Hover on factor: shows "+" connect button on right edge
    │
    ├── EarnConditionGroupCard.vue       # Right column card
    │   - Collapsed: group name, condition count badge, linked factor count (plug icon), chevron
    │   - Expanded: conditions table (Type, Items count, Logic, Threshold type, Excess)
    │   - Connection dot on left edge for SVG line endpoints
    │
    ├── EarnFactorConfig.vue             # Right sidebar panel for factor editing
    │   - Fields: name, type (rate/multiplier), amount, target currency, target entity,
    │     window start/end, expiry days, public/private toggle, condition group dropdown
    │   - Save writes factor into its parent group via bff_upsert_earn_factor_group
    │
    ├── EarnConditionGroupConfig.vue     # Right sidebar panel for condition group editing
    │   - Group name field
    │   - Repeatable condition entries, each with:
    │     entity type dropdown, entity multi-select picker, operator toggle (OR/AND/EACH),
    │     threshold type, apply-to-excess toggle, min/max threshold inputs
    │   - Entity picker modal with search + checkbox selection
    │   - Save writes via bff_upsert_earn_conditions_group
    │
    ├── CreateGroupModal.vue             # Modal for creating new groups
    │   - Factor group: name, stackable toggle, window start/end
    │   - Condition group: name only
    │
    └── ConnectPopup.vue                 # Popup for linking factor → condition group
        - Lists all condition groups with search
        - Click to select → saves connection → line appears
```

---

## Data Flow Summary

```
                           ┌─────────────────────────────────┐
                           │       Supabase PostgreSQL        │
                           │                                  │
                           │  earn_factor_group               │
                           │  earn_factor                     │
                           │  earn_conditions_group           │
                           │  earn_conditions                 │
                           └──────────┬──────────────────────┘
                                      │
                                      │  fetch() with apikey + Bearer JWT
                                      │
                    ┌─────────────────┴─────────────────┐
                    │          useApi.js                  │
                    │  rpc()  →  POST /rest/v1/rpc/{fn}  │
                    │  restGet() → GET /rest/v1/{table}   │
                    └─────────────────┬─────────────────┘
                                      │
                    ┌─────────────────┴─────────────────┐
                    │         wwElement.vue               │
                    │                                     │
                    │  factorGroups[]                     │
                    │  factorsByGroup{}                   │
                    │  allConditionGroupsList[]           │
                    │  conditionGroupDetailsCache{}       │
                    │  allEntityOptions[]                 │
                    │                                     │
                    │  displayedConditionGroups (computed) │
                    │  connectionLines (computed from DOM) │
                    └──┬──────────┬──────────┬──────────┘
                       │          │          │
              ┌────────┘   ┌──────┘   ┌──────┘
              ▼            ▼          ▼
    EarnFactorGroupCard  SVG Lines  EarnConditionGroupCard
              │                      │
              ▼                      ▼
    EarnFactorConfig          EarnConditionGroupConfig
    (sidebar save → API)      (sidebar save → API)
```

---

## Database Tables Involved

| Table | Purpose |
|-------|---------|
| `earn_factor_group` | Group container: `id`, `name`, `stackable`, `window_start`, `window_end`, `active_status`, `merchant_id` |
| `earn_factor` | Individual rule: `id`, `earn_factor_type` (rate/multiplier), `earn_factor_amount`, `target_currency` (points/ticket), `target_entity_id`, `public`, `window_*`, `earn_factor_group_id` FK, `earn_conditions_group_id` FK |
| `earn_conditions_group` | Condition container: `id`, `name`, `merchant_id` |
| `earn_conditions` | Individual condition: `id`, `group_id` FK, `entity` (enum), `entity_ids` (uuid[]), `operator`, `threshold_unit`, `min_threshold`, `max_threshold`, `apply_to_excess_only`, `exclude` |

### Enums

- **`earn_factor_type`**: `rate`, `multiplier`
- **`currency`**: `points`, `ticket`
- **`earn_factor_entity_type`**: `product_product`, `product_sku`, `product_brand`, `product_category`, `store`, `store_attribute_set`, `tier`, `persona`, and more

---

## Save Behavior

There is **no page-level save**. Each entity saves independently:

| Entity | Save Trigger | API Called |
|--------|-------------|-----------|
| Earn Factor Group | "Create" modal → Save button | `bff_upsert_earn_factor_group` |
| Earn Factor | Sidebar config → Save button | `bff_upsert_earn_factor_group` (with full factors array) |
| Earn Condition Group + Conditions | Sidebar config → Save button | `bff_upsert_earn_conditions_group` |
| Factor ↔ Condition link | "+" popup → select group, OR sidebar dropdown | `bff_upsert_earn_factor_group` (updates `earn_conditions_group_id`) |

All upsert functions are atomic — they handle insert/update/delete within a single database transaction using `SECURITY DEFINER` functions scoped to the authenticated merchant.
