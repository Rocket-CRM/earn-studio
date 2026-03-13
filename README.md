# Earn Studio — WeWeb Custom Element

Visual mapping component for Earn Factors ↔ Earn Condition Groups, built as a WeWeb custom element using the Polaris design system.

## Overview

This component provides a two-column layout that visualizes the relationship between:
- **Left column**: Earn Factor Groups containing individual Earn Factors (rate/multiplier rules)
- **Right column**: Earn Condition Groups containing qualifying criteria (entity-based conditions)
- **Connection lines**: SVG lines linking individual Earn Factors to their assigned Earn Condition Groups

## Development

### Prerequisites
- Node.js 18+
- npm

### Install
```bash
npm install
```

### Serve locally
```bash
npx weweb serve name=earn-studio type=wwobject --port=8083
```
Then add the custom element in the WeWeb editor developer popup.

### Build for release
```bash
npx weweb build name=earn-studio type=wwobject
```

## Architecture

```
src/
├── wwElement.vue                        # Main component - layout, state, API orchestration
├── useApi.js                            # Supabase RPC API helper
└── components/
    ├── EarnFactorGroupCard.vue          # Left column: collapsible group with factor rows
    ├── EarnConditionGroupCard.vue       # Right column: collapsible group with conditions table
    ├── EarnFactorConfig.vue             # Sidebar: add/edit earn factor
    ├── EarnConditionGroupConfig.vue     # Sidebar: add/edit condition group + conditions
    ├── CreateGroupModal.vue             # Modal: create new factor/condition group
    └── ConnectPopup.vue                 # Popup: link factor to condition group
```

## Supabase API Endpoints

All calls target `https://wkevmsedchftztoolkmi.supabase.co` via RPC:

| Function | Purpose |
|----------|---------|
| `bff_get_earn_factor_group_details` | Get group + factors |
| `bff_upsert_earn_factor_group` | Create/update group with factors |
| `bff_get_earn_factors_by_group` | List factors for a group |
| `bff_get_earn_conditions_group` | Get condition group + conditions |
| `bff_upsert_earn_conditions_group` | Create/update condition group |
| `bff_get_all_earn_conditions_groups` | List all condition groups |
| `api_get_earn_conditions_groups` | List with stats and pagination |
| `get_all_entity_options` | Entity picker data |
| `admin_delete_earn_factor` | Delete a factor |
| `admin_delete_earn_factor_group` | Delete a group |
| `admin_delete_earn_conditions_group` | Delete a condition group |

## Styling

Built on `polaris-weweb-styles` package using Polaris design tokens, mixins, and component patterns.

## Polaris Gap Tracking

### Polaris Styles NPM Package — Items to Add

- **What:** Sidebar config panel pattern (fixed right panel with header/body/footer)
- **Where used:** EarnFactorConfig, EarnConditionGroupConfig
- **Suggested addition:** `@mixin polaris-sidebar-panel`, `@mixin polaris-sidebar-panel-header/body/footer`

- **What:** Connection line / mapping visualization tokens
- **Where used:** SVG connection lines between columns
- **Suggested addition:** `--p-color-connection-line`, `--p-color-connection-line-active`, line weight tokens

- **What:** Expandable card with nested content pattern
- **Where used:** EarnFactorGroupCard, EarnConditionGroupCard
- **Suggested addition:** `@mixin polaris-expandable-card`, `@mixin polaris-expandable-card-header/body`

- **What:** Entity tag picker with multi-select inside bordered container
- **Where used:** EarnConditionGroupConfig entity selection
- **Suggested addition:** `@mixin polaris-tag-input`, `@mixin polaris-tag-input-container`

- **What:** Segmented toggle button group (OR/AND/EACH pattern)
- **Where used:** Operator toggle in condition config
- **Suggested addition:** `@mixin polaris-segmented-control-item--active` beyond existing segmented group

### Polaris Component Structure Guidelines — Items to Add

- **What:** Two-column mapping visualization pattern with connection lines
- **Where used:** Main layout connecting earn factors to condition groups
- **Suggested addition:** Document the pattern: left/right columns + SVG connection area, duplication rule for shared references

- **What:** Hover-to-link interaction pattern (+ button appears on hover)
- **Where used:** Earn factor rows showing "+" connection button on hover
- **Suggested addition:** Document interaction: hover reveals action affordance, click opens selection popup

- **What:** Sidebar config panel behavior (slide-in from right, independent save)
- **Where used:** Factor config and condition group config panels
- **Suggested addition:** Document panel lifecycle: open, edit, save, close; transition animation; overlay behavior

- **What:** Inline data table within expandable cards
- **Where used:** Conditions table inside expanded condition group cards
- **Suggested addition:** Document compact table variant for card-embedded data display

- **What:** Duplicate-block pattern to avoid crossing connection lines
- **Where used:** Right column duplicates condition groups that are shared by multiple factors
- **Suggested addition:** Document the Hookdeck-style deduplication pattern for visual clarity in mapping UIs
