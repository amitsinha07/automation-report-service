# ONDC:TRV14 (Unreserved Entry Pass) — Validation Reference

This document lists **every validation that runs for ONDC:TRV14** when a Workbench
session is reported. It is written from the source code, not from the spec.

## 1. Where TRV14 is validated

TRV14 is **not** validated by this service (`automation-report-service`):

| Check | Result |
|---|---|
| `ONDC:TRV14:<version>` in `ENABLED_DOMAINS` (`src/utils/constants.ts`) | No |
| `src/validations/ONDC:TRV14/` folder | Does not exist |
| TRV14 section in `src/config/ReportingConfig.yaml` | Does not exist |
| TRV14 save-specs | Do not exist |

So `ReportService.generate()` sends every TRV14 session to **Pramaan** through
`checkPramaanReport()`. The only TRV14 code in this repo is the routing data that
service uses:

- `src/utils/constants.ts` (`FLOW_ID_MAP["ONDC:TRV14"]["2.0.0"]["unreserved-entry-pass"]`) maps Workbench flow ids to Pramaan test ids:

  | Workbench flow id | Pramaan test id |
  |---|---|
  | `purchase_journey_with_form` | EP_1 |
  | `purchase_journey_without_form` | EP_2 |
  | `purchase_journey_with_form_Multiple_Tickets` | EP_3 |
  | `purchase_journey_without_form_Multiple_Tickets` | EP_4 |
  | `technical_cancellation_with_form` | EP_5 |
  | `technical_cancellation` | EP_6 |
  | `User_Cancellation_FULL_With_Form` | EP_7 |
  | `User Cancellation (Full)` | EP_8 |

- `typeMapping["unreserved-entry-pass"] = "ENTRY_PASS"` sets the Pramaan test type.

All the checks below live in **`ONDC-Official/automation-report-pramaan`**, in
`sellerNPTestAPI/src/entryPassSellerNPTest/`, plus the shared helpers it calls.
The code was read at commit `cb89ac0`.

| File | Purpose | Test definitions |
|---|---|---|
| `testRunner.js` | Maps flow steps to validators | — |
| `context.js` | Context checks (all actions) | 15 + up to 7 from `commonBussiness.js` |
| `on_search.js` | on_search | 103 |
| `on_select.js` | on_select | 137 |
| `on_init.js` | on_init | 145 |
| `on_confirm.js` | on_confirm | 150 |
| `on_status.js` | on_status | 144 |
| `on_cancel.js` | on_cancel | 153 |
| `on_update.js`, `on_track.js` | **Never run for TRV14** (no EP flow uses them) | 132, 7 |
| `centralizedUtilities/responseVerification.js` | Sync ACK/NACK reply checks | 22 |
| `bussinessTests/commonBussiness.js` | Context checks against earlier calls | 7 |
| `bussinessTests/rideHailingBusiness.js` | `SETTLEMENT_AMOUNT` check (on_confirm) | 1 |

"Test definitions" counts the templates. Many run once per array element (each
item, fulfillment, stop, tag, list entry), so one report can contain several hundred
results.

### 1.1 What is validated

- **Only the seller's (BPP) replies are validated**: `on_search`, `on_select`,
  `on_init`, `on_confirm`, `on_status` and `on_cancel`.
- The buyer's requests (`search`, `select`, `init`, `confirm`, `status`, `cancel`)
  are **not** validated.
- Tests marked **(OPTIONAL)** in their title are still real assertions. Pramaan
  labels them optional in the report, but they fail if the condition is not met
  once they run. Most optional tests only run when the parent field is present.

## 2. Flows and which payload each step validates

Defined in `sellerNPTestAPI/src/config/flow.json` (all `type: ENTRY_PASS`, `version: 2.0.0`):

| Pramaan id | Name | Steps validated (in order) |
|---|---|---|
| EP_1 | Flow 1A: Purchase journey – single ticket **with form** | on_search ×3, on_select ×2, on_init, on_confirm, on_status |
| EP_2 | Flow 1B: Purchase journey – single ticket **without form** | on_search ×3, on_select ×1, on_init, on_confirm, on_status |
| EP_3 | Flow 2A: Purchase journey – multiple tickets **with form** | same as EP_1 |
| EP_4 | Flow 2B: Purchase journey – multiple tickets **without form** | same as EP_2 |
| EP_5 | Flow 3A: Technical cancellation **with form** | EP_1 steps + on_cancel ×2 |
| EP_6 | Flow 3B: Technical cancellation **without form** | EP_2 steps + on_cancel ×2 |
| EP_7 | Flow 4A: User cancellation **with form** | EP_1 steps without on_status + on_cancel ×2 |
| EP_8 | Flow 4B: User cancellation **without form** | EP_2 steps without on_status + on_cancel ×2 |

How `testRunner.js` picks the payload for each step:

| Step | Payload validated | Step label |
|---|---|---|
| `on_search_one/two/three` | 1st / 2nd / 3rd `on_search` in the logs | I / II / III |
| `on_select_one/two` | 1st / 2nd `on_select` | I / II |
| `on_init`, `on_confirm` | first matching log | — |
| `on_status` | next `on_status` in order | — |
| `on_cancel_technical` (both steps) | **first `on_cancel` both times** (`cancelIndex.on_cancel_technical = 0`) | "Technical" |

All EP flows use `on_cancel_technical`, including the user cancellation flows
EP_7 and EP_8.

`isSelfPickup` is `true` only when the flow id is `DEM_6`, so for every TRV14 flow
the expected fulfillment type is **`VISIT`**.

## 3. Common checks

### 3.1 Context checks (`context.js`) — every on_* call

| # | Check | Rule |
|---|---|---|
| C1 | `context` exists | must exist |
| C2 | `context.location` | object |
| C3 | `context.location.country.code` | string, **= `IND`** |
| C4 | `context.location.city.code` | exists, string |
| C5 | `context.domain` | **= `ONDC:TRV14`** |
| C6 | `context.timestamp` | matches `^20\d{2}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}\.\d{3}Z$` (milliseconds and `Z` required) |
| C7 | `context.bap_id` | exists |
| C8 | `context.transaction_id` | exists, valid UUID |
| C9 | `context.message_id` | exists, valid UUID |
| C10 | `context.version` | **= `2.0.0`** |
| C11 | `context.action` | equals the action being validated (`on_search`, `on_select`, …) |
| C12 | `context.bap_uri` | string matching `^(https?\|ftp)://…` |
| C13 | `context.ttl` | ISO 8601 duration (`P…`, e.g. `PT30S`) |
| C14 | `context.bpp_id` | exists |
| C15 | `context.bpp_uri` | string, valid URI as in C12 |

C14 and C15 are skipped only when the action is `search`, so they always run for
TRV14.

### 3.2 Context checks against earlier calls (`bussinessTests/commonBussiness.js`)

| # | Check | Rule |
|---|---|---|
| X1 | `context.message_id` unique | the same `message_id` appears at most once across all `on_*` logs |
| X2 | `context.timestamp` increasing | later than the previous log's request timestamp (only when this is not the first log) |
| X3 | `context.bap_id` | equals `bap_id` of the first log (not the first log) |
| X4 | `context.bap_uri` | equals `bap_uri` of the first log (not the first log) |
| X5 | `context.bpp_id` | equals `bpp_id` of the first log that has a BPP id and URI |
| X6 | `context.bpp_uri` | equals `bpp_uri` of that same log |

These checks only work when the validator passes `logs` to `contextTests()`:

| Action | `logs` passed? | Effect |
|---|---|---|
| on_search | yes | X1–X6 run |
| on_select | **no** | X1 always passes; X2–X6 never run |
| on_init | yes | X1–X6 run |
| on_confirm | yes | X1–X6 run |
| on_status | yes | X1–X6 run |
| on_cancel | **no** | X1 always passes; X2–X6 never run |

### 3.3 Sync reply checks (`centralizedUtilities/responseVerification.js`)

These run for **on_search, on_select, on_init and on_confirm only**. `on_status`
and `on_cancel` do not call `response_verification`, so their ACK/NACK reply is
never checked.

The service finds the matching request log (same `message_id`, action not
starting with `on_`) and checks its sync reply. Test ids look like
`ONDC:TRV14_bpp_sync_<action>_response_<n>_type|enum`:

| # | Check | Rule |
|---|---|---|
| S1 | sync reply exists | required |
| S2 | `context` | object |
| S3–S10 | `context.domain`, `action`, `bap_id`, `bap_uri`, `transaction_id`, `message_id`, `timestamp` | strings; `ttl` string (OPTIONAL) |
| S11 | `message` | object |
| S12 | `message.ack` | object |
| S13 | `message.ack.status` | string |
| S14 | `message.ack.status` | one of `ACK`, `NACK` |
| S15–S18 | `error` (when present) | object; `type` string (OPTIONAL); `code` string; `message` string |

For the 2nd and 3rd `on_search`: if no matching request exists, the call is treated
as **unsolicited** and an empty "Unsolicited Call" suite is returned (no failure).

## 4. on_search (`on_search.js`) — steps on_search_one, two and three

The same checks run for all three steps. The suite is named `on_search (I|II|III)`.
Context checks: C1–C15 and X1–X6. Sync reply checks: S1–S18.

### 4.1 Message and catalog

| # | Field | Rule |
|---|---|---|
| OS1 | `message` | exists |
| OS2 | `message.catalog` | object |
| OS3 | `message.catalog.tags` | array |
| OS4 | `catalog.tags[t]` (every tag) | has `display` (boolean), `descriptor` (object), `list` (array) |
| OS5 | `catalog.tags[t].descriptor.code` | string |
| OS6 | `catalog.tags[t].list` | non-empty array |
| OS7 | `catalog.tags[t].list[l]` | has `descriptor` (object) and `value` (string) |
| OS8 | `catalog.tags[t].list[l].descriptor.code` | string |
| OS9 | `catalog.tags[t].list[l].descriptor.code` | "equals itself" — compares the value to itself, **so it always passes** |
| OS10 | `catalog.tags[t].list[l].value` | non-empty string |
| OS11 | `message.catalog.descriptor` | object |
| OS12 | `catalog.descriptor.name` | string |
| OS13 | `catalog.descriptor.short_desc` | string (only when present) |
| OS14 | `catalog.descriptor.images[i]` | object, `url` string (only when images present) |

### 4.2 Providers

| # | Field | Rule |
|---|---|---|
| OS15 | `catalog.providers` | array |
| OS16 | `providers[p]` | object |
| OS17 | `providers[p].id` | string |
| OS18 | `providers[p].descriptor` | object |
| OS19 | `providers[p].descriptor.name` | string |
| OS20 | `providers[p].descriptor.short_desc` | string |
| OS21 | `providers[p].descriptor.long_desc` | string (only when present) |
| OS22 | `providers[p].descriptor.images` | array (OPTIONAL); each image an object with `url` string (OPTIONAL) |
| OS23 | `providers[p].categories` | array |
| OS24 | `categories[c]` | object |
| OS25 | `categories[c].id` | string (OPTIONAL) |
| OS26 | `categories[c].parent_category_id` | string (only when present) |
| OS27 | `categories[c].descriptor` | object |
| OS28 | `categories[c].descriptor.name` | string (OPTIONAL) |
| OS29 | `categories[c].descriptor.code` | when present: one of `CULTURE_HERITAGE`, `ECONOMY`, `PREMIUM_ECONOMY`, `BUSINESS`, `FIRST_CLASS`, `PASS`, `TICKET`, `SEATER`, `SEMI_SLEEPER`, `SLEEPER` |
| OS30 | `providers[p].time` | object |
| OS31 | `providers[p].time.range` | object |
| OS32 | `providers[p].time.range.start` | string |
| OS33 | `providers[p].time.range.end` | string |
| OS34 | `providers[p].locations` | array (OPTIONAL) |
| OS35 | `locations[l]` | object, `id` string (OPTIONAL) |
| OS36 | `locations[l].gps` | string matching `^-?\d{1,3}\.\d{6}, ?-?\d{1,3}\.\d{6}$` — **exactly 6 decimal places** (OPTIONAL) |
| OS37 | `providers[p].payments` | array |
| OS38 | `payments[z]` | object |
| OS39 | `payments[z].collected_by` | string |

### 4.3 Items (`providers[p].items[i]`)

`items` is checked only when present.

| # | Field | Condition | Rule |
|---|---|---|---|
| OS40 | `items` | — | array |
| OS41 | `items[i]` | — | object |
| OS42 | `items[i].id` | — | string |
| OS43 | `items[i].descriptor` | — | object |
| OS44 | `items[i].descriptor.name` | — | string (OPTIONAL) |
| OS45 | `items[i].descriptor.code` | — | one of `ABSTRACT`, `ENTRY_PASS`, `ADD_ON` |
| OS46 | `items[i].fulfillment_ids` | code ≠ `ADD_ON` | array |
| OS47 | `fulfillment_ids[f]` | code ≠ `ADD_ON` | string; the array contains it (always passes) |
| OS48 | `items[i].location_ids` | — | array (OPTIONAL); each entry a string |
| OS49 | `items[i].add_ons` | when present | array; each add-on is an object with `id` (string), `descriptor.name` (string), `quantity.available.count` (number), `price.value` and `price.currency` (strings) |
| OS50 | `items[i].price` | code = `ENTRY_PASS` | object; `currency` string (OPTIONAL); `value` string (OPTIONAL) |
| OS51 | `items[i].quantity` | code = `ENTRY_PASS` | object with `maximum.count` (number) and `minimum.count` (number) |

### 4.4 Fulfillments (`providers[p].fulfillments[f]`)

`fulfillments` is checked only when present.

| # | Field | Rule |
|---|---|---|
| OS52 | `fulfillments` | array |
| OS53 | `fulfillments[f]` | object |
| OS54 | `fulfillments[f].id` | string |
| OS55 | `fulfillments[f].type` | **= `VISIT`** |
| OS56 | `fulfillments[f].stops` | array |
| OS57 | `stops[s]` | object |
| OS58 | `stops[s].type` | one of `START`, `END`, `TRANSIT_STOP`, `INTERMEDIATE_STOP` |
| OS59 | `stops[s].instructions` | object (OPTIONAL) |
| OS60 | `stops[s].instructions.additional_desc` | object with `url` and `content_type` strings (OPTIONAL) |
| OS61 | `fulfillments[f].vehicle` | object (**required**) |
| OS62 | `fulfillments[f].vehicle.category` | one of `SITE`, `TWO_WHEELER`, `AUTO_RICKSHAW`, `CAB` |
| OS63 | `fulfillments[f].agent` | object (**required**) |
| OS64 | `agent.organization.contact.phone/email` | organization and contact objects, phone and email strings (all OPTIONAL) |

### 4.5 Catalog tags (BAP_TERMS / BPP_TERMS / PAGINATION)

When `catalog.tags` is present:

| # | Check | Rule |
|---|---|---|
| OS65 | `message.catalog.tags` | array |
| OS66 | tag `BAP_TERMS` | present |
| OS67 | tag `BPP_TERMS` | present |
| OS68 | tag `PAGINATION` | present |
| OS69 | for each of these tags | has `descriptor` (object), `display` (boolean), `list` (array); `descriptor.code` string and equal to the expected code; `display` boolean; `list` non-empty |
| OS70 | each required list entry | present; has `descriptor` (object) and `value` (string); `descriptor.code` string and equal to the expected code; `value` non-empty string |

Required list entries per tag:

| Tag | Required list `descriptor.code`s |
|---|---|
| `BAP_TERMS` | `BUYER_FINDER_FEES_TYPE`, `BUYER_FINDER_FEES_PERCENTAGE`, `STATIC_TERMS`, `SETTLEMENT_BASIS`, `SETTLEMENT_WINDOW` |
| `BPP_TERMS` | `BUYER_FINDER_FEES_TYPE`, `BUYER_FINDER_FEES_PERCENTAGE`, `STATIC_TERMS`, `MANDATORY_ARBITRATION`, `COURT_JURISDICTION`, `DELAY_INTEREST` |
| `PAGINATION` | `PAGINATION_ID`, `MAX_PAGE_NUMBER` |

## 5. on_select (`on_select.js`) — steps on_select_one (I) and on_select_two (II)

Context checks: C1–C15 only. X1 always passes and X2–X6 never run (see 3.2).
Sync reply checks: S1–S18.

### 5.1 Order and provider

| # | Field | Rule |
|---|---|---|
| SL1 | `message` | exists |
| SL2 | `message.order` | object |
| SL3 | `order.provider` | object |
| SL4 | `provider.id` | string |
| SL5 | `provider.descriptor` | object |
| SL6 | `provider.descriptor.name` | string |
| SL7 | `provider.descriptor.short_desc` | string (only when present) |
| SL8 | `provider.descriptor.images[n]` | `url` and `size_type` strings (OPTIONAL, when images present) |
| SL9 | `provider.locations` | array |
| SL10 | `locations[l].id` | string |
| SL11 | `locations[l].gps` | matches `^-?\d{1,3}\.\d{1,7}, ?-?\d{1,3}\.\d{1,7}$` (1–7 decimal places) |
| SL12 | `locations[l].descriptor` | object, with `name` and `short_desc` strings |
| SL13 | `locations[l].descriptor.additional_desc` | when present: `url` and `content_type` strings |
| SL14 | `locations[l].descriptor.images[n]` | when present: `url` and `size_type` strings |
| SL15 | `locations[l].rating` | string (OPTIONAL) |

### 5.2 Cancellation and replacement terms (required)

| # | Field | Rule |
|---|---|---|
| SL16 | `order.cancellation_terms` | **array (required)** |
| SL17 | `cancellation_terms[c]` | object |
| SL18 | `cancellation_terms[c].fulfillment_state.descriptor.code` | when `fulfillment_state` is present: `fulfillment_state` and `descriptor` objects; `code` one of `CONFIRMED`, `DRIVER_EN_ROUTE_TO_PICKUP`, `RIDE_CONFIRMED`, `RIDE_CANCELLED`, `PAYMENT_COLLECTED`, `RIDE_ENDED`, `DRIVER_AT_PICKUP`, `RIDE_STARTED`, `RIDE_UPDATED`, `RIDE_ASSIGNED`, `RIDE_ENROUTE_PICKUP`, `RIDE_ARRIVED_PICKUP` |
| SL19 | `cancellation_terms[c].cancel_by` | object with `label` and `duration` strings |
| SL20 | `cancellation_terms[c].cancellation_fee` | object |
| SL21 | `cancellation_terms[c].cancellation_eligible` | boolean |
| SL22 | `cancellation_terms[c].external_ref` | when present: object with `mimetype` and `url` strings |
| SL23 | `order.replacement_terms` | **array (required)** |
| SL24 | `replacement_terms[r]` | object; `external_ref` object with `mimetype` and `url` strings |

### 5.3 Quote

| # | Field | Rule |
|---|---|---|
| SL25 | `order.quote` | object |
| SL26 | `order.quote.price` | object |
| SL27 | `quote.price.value` = sum of breakup | `quote.price.value` (2 decimals) equals the sum of every `breakup[].price.value`, rounded exactly, down or up |
| SL28 | `quote.price.value` | string |
| SL29 | `quote.price.currency` | string |
| SL30 | breakup line `BASE_FARE` | **required** |
| SL31 | breakup line `TAX` | **required** |
| SL32 | breakup line `ADD_ONS` | OPTIONAL — the test still fails if it is missing |
| SL33 | each found breakup line | has `price` object with `value` and `currency` strings; `price.value` ≥ 0 (titled "positive value"); `title` equals the expected title |

### 5.4 Fulfillments

| # | Field | Rule |
|---|---|---|
| SL34 | `order.fulfillments` | array |
| SL35 | `fulfillments[f]` | object |
| SL36 | `fulfillments[f].id` | string (labelled OPTIONAL, asserted) |
| SL37 | `fulfillments[f].type` | **= `VISIT`** |
| SL38 | `fulfillments[f].stops` | array |
| SL39 | `stops[s]` | object |
| SL40 | `stops[s].instructions` | object, `additional_desc` object, `url` and `content_type` strings (OPTIONAL) |
| SL41 | `stops[s].type` | one of `START`, `END`, `INTERMEDIATE_STOP`, `TRANSIT_STOP` |
| SL42 | `stops[s].id` | when present: string; then `time` object and `time.timestamp` string are required |
| SL43 | `fulfillments[f].agent` | object (required); `organization.contact.phone/email` OPTIONAL |
| SL44 | `fulfillments[f].vehicle` | object (required) |
| SL45 | `fulfillments[f].vehicle.category` | one of `SITE`, `TWO_WHEELER`, `AUTO_RICKSHAW`, `CAB` |

### 5.5 Items

| # | Field | Condition | Rule |
|---|---|---|---|
| SL46 | `order.items` | — | array |
| SL47 | `items[i]` | — | object |
| SL48 | `items[i].id` | — | string |
| SL49 | `items[i].parent_item_id` | when present | string |
| SL50 | `items[i].descriptor` | — | object, `name` string |
| SL51 | `items[i].descriptor.code` | — | one of `ABSTRACT`, `ENTRY_PASS`, `ADD_ON` |
| SL52 | `items[i].descriptor.images[n]` | when present | `url` and `size_type` strings |
| SL53 | `items[i].price` | code = `ENTRY_PASS` | object with `currency` and `value` strings |
| SL54 | `items[i].quantity.selected.count` | code = `ENTRY_PASS` | `quantity` object, `selected` object, `count` number |
| SL55 | `items[i].time` | code = `ENTRY_PASS` | object, with `label` and `duration` strings (all OPTIONAL) |
| SL56 | `items[i].location_ids` | — | array (OPTIONAL); entries strings |
| SL57 | `items[i].tags` `INCLUSIONS` / `EXCLUSIONS` | — | OPTIONAL; when found: `descriptor` (object) + `list` (array); `descriptor.code` string and equal; `list` non-empty |
| SL58 | `items[i].fulfillment_ids` | code ≠ `ADD_ON` | non-empty array of strings |
| SL59 | `items[i].category_ids` | code ≠ `ADD_ON` | non-empty array of strings |
| SL60 | `items[i].add_ons` | when present | array; each has `id`, `descriptor.name`, `quantity.selected.count` (number), `price.value` and `price.currency` |

### 5.6 Form (`items[i].xinput`) — when xinput is present

| # | Field | Condition | Rule |
|---|---|---|---|
| SL61 | `xinput` | — | object |
| SL62 | `xinput.head` | when head present | object; `descriptor` object; `descriptor.name` string (OPTIONAL); `index` object with `min`, `cur`, `max` numbers (OPTIONAL); `headings` array (OPTIONAL) of strings |
| SL63 | `xinput.form` | — | object |
| SL64 | `xinput.form.id` | — | string |
| SL65 | `xinput.form.mime_type` | on_select **I** only | string |
| SL66 | `xinput.form.url` | on_select **I** only | string |
| SL67 | `xinput.form.resubmit` | on_select **I** only | boolean (OPTIONAL) |
| SL68 | `xinput.form.multiple_sumbissions` | on_select **I** only | boolean (OPTIONAL; note the field name is misspelled in the code) |
| SL69 | `xinput.required` | on_select **I** only | **must be `true`** (OPTIONAL) |
| SL70 | `xinput.form_response` | when present (typically on_select II) | object with `status` and `submission_id` strings |

The ride-hailing `FARE_POLICY`/`INFO` lists in `on_select.js` never run, because only
`INCLUSIONS` and `EXCLUSIONS` are looked up.

## 6. on_init (`on_init.js`)

Context checks: C1–C15 and X1–X6. Sync reply checks: S1–S18.

Includes **all on_select checks SL1–SL60** (provider, terms, quote, fulfillments,
items), with these differences:
- no `xinput` checks;
- no `parent_item_id` check;
- item tags depend on the item code (see 6.3).

on_init adds the checks below.

### 6.1 Order tags (required)

| # | Check | Rule |
|---|---|---|
| IN1 | `order.tags` | exists, non-empty array |
| IN2 | tag `BAP_TERMS` | present |
| IN3 | tag `BPP_TERMS` | present |
| IN4 | each tag | has `descriptor` (object) and `list` (array); `descriptor.code` string (OPTIONAL) and equal to the expected code; `display` boolean (OPTIONAL); `list` non-empty |
| IN5 | each required list entry | present; `descriptor` (object) + `value` (string) (OPTIONAL); `descriptor.code` string and equal (OPTIONAL); `value` non-empty (OPTIONAL) |

| Tag | Required list `descriptor.code`s |
|---|---|
| `BAP_TERMS` | `BUYER_FINDER_FEES_PERCENTAGE`, `BUYER_FINDER_FEES_TYPE`, `STATIC_TERMS`, `SETTLEMENT_BASIS`, `SETTLEMENT_WINDOW` |
| `BPP_TERMS` | `BUYER_FINDER_FEES_PERCENTAGE`, `BUYER_FINDER_FEES_TYPE`, `STATIC_TERMS`, `SETTLEMENT_BANK_CODE`, `SETTLEMENT_TYPE`, `MANDATORY_ARBITRATION`, `COURT_JURISDICTION`, `DELAY_INTEREST`, `SETTLEMENT_AMOUNT`, `SETTLEMENT_BANK_ACCOUNT_NUMBER` |

### 6.2 Billing (required)

| # | Field | Rule |
|---|---|---|
| IN6 | `order.billing` | object |
| IN7 | `billing.name` | string |
| IN8 | `billing.email` | string |
| IN9 | `billing.phone` | string |

### 6.3 Item tags — depend on `items[i].descriptor.code`

| Item code | Tags checked |
|---|---|
| `ENTRY_PASS` | `FARE_POLICY` (**required**) |
| `ABSTRACT` | `INCLUSIONS`, `EXCLUSIONS` (OPTIONAL) |
| `ADD_ON` | none |

For each found tag:
- `descriptor` (object) + `list` (array);
- `descriptor.code` string and equal to the expected code;
- `list` non-empty.

`FARE_POLICY` list must contain `MIN_AGE`, `MAX_AGE`, `GENDER`, `NATIONALITY` and
`ADDITIONAL_DETAILS_REQUIRED`. For each entry:
- `descriptor` (object) + `value` (string);
- `descriptor.code` equal to the expected code;
- `value` non-empty (OPTIONAL).

For `INCLUSIONS` and `EXCLUSIONS`, every list entry's `value` must be a non-empty
string (OPTIONAL).

### 6.4 Cancellation fee (inside cancellation_terms)

| # | Field | Rule |
|---|---|---|
| IN10 | `cancellation_fee.amount` | when present: object, with `currency` and `value` strings (OPTIONAL) |
| IN11 | `cancellation_fee.percentage` | string (OPTIONAL) |

### 6.5 Payments (required)

| # | Field | Rule |
|---|---|---|
| IN12 | `order.payments` | array |
| IN13 | `payments[z]` | object |
| IN14 | `payments[z].id` | string |
| IN15 | `payments[z].collected_by` | one of `BAP`, `BPP` |
| IN16 | `payments[z].type` | one of `PRE-ORDER`, `ON-FULFILLMENT`, `POST-FULFILLMENT` |
| IN17 | `payments[z].status` | if type is `PRE-ORDER`, `ON-ORDER`, `ON-FULFILLMENT` or `POST-FULFILLMENT`: **must be `NOT-PAID`**; otherwise `PAID` or `NOT-PAID` |

## 7. on_confirm (`on_confirm.js`)

Context checks: C1–C15 and X1–X6. Sync reply checks: S1–S18.

Includes all on_init checks (order tags 6.1, billing 6.2, item tags 6.3,
cancellation terms, quote, fulfillments, items, payments), with these differences:

| # | Difference | Rule |
|---|---|---|
| CF1 | `cancellation_fee.amount` checks | not run; only `cancellation_fee.percentage` (string, OPTIONAL) |
| CF2 | `provider.descriptor.short_desc` | not checked |
| CF3 | `order.created_at` | string (required) |
| CF4 | `order.updated_at` | string (required) |
| CF5 | `items[i].descriptor.short_desc`, `long_desc` | strings (required) |
| CF6 | `items[i].price` | when present: object with `currency` and `value` strings |
| CF7 | `items[i].time` | when present: object with `label` and `duration` strings (OPTIONAL) |
| CF8 | `payments[z].type` | one of `PRE-ORDER`, `ON-ORDER`, `ON-FULFILLMENT`, `POST-FULFILLMENT` |
| CF9 | `payments[z].status` | `PRE-ORDER`/`ON-ORDER` → **must be `PAID`**; `ON-FULFILLMENT`/`POST-FULFILLMENT` → **must be `NOT-PAID`**; otherwise `PAID` or `NOT-PAID` |
| CF10 | `payments[z].params` | object with `transaction_id`, `currency` and `account` strings |
| CF11 | `SETTLEMENT_AMOUNT` (`rideHailingBusiness.settlementTermsListTests`) | if a `payments[].tags[]` with code `SETTLEMENT_TERMS` has a `SETTLEMENT_AMOUNT` entry, its value must equal the amount worked out from the last on_select quote and the first on_init's `BUYER_FINDER_FEES` (exact, rounded down or up). TRV14 sends settlement terms in `order.tags` (`BPP_TERMS`), so this normally finds nothing and adds no test. |
| CF12 | "Driver not found" error block | `error` object, `error.code` = `90203`, `error.message` string. Runs only in `DRIVER_NOT_FOUND` mode, which no TRV14 flow uses. |

`order.id` and `order.status` are **not** checked in on_confirm.

## 8. on_status (`on_status.js`)

Context checks: C1–C15 and X1–X6. **No sync reply checks.**

Same checks as on_confirm (7), with these differences:

| # | Difference | Rule |
|---|---|---|
| ST1 | `order.id` | string (required) — the only action that checks it |
| ST2 | `order.created_at` / `updated_at` | **not** checked |
| ST3 | `items[i].descriptor.short_desc/long_desc` | not required |
| ST4 | `SETTLEMENT_AMOUNT` check (CF11) | not run |
| ST5 | Order tag list entries | `descriptor` + `value` and `descriptor.code` equality are required, not OPTIONAL; item-tag list entries are OPTIONAL |

Payments (`type`, `status` rule as in CF9, `params.*`), billing, quote and
fulfillments (`VISIT`, stops, agent, vehicle) are checked as in on_confirm.

on_status does **not** check `order.status` or any fulfillment state.

## 9. on_cancel (`on_cancel.js`) — both on_cancel_technical steps

Context checks: C1–C15 only (X1 always passes, X2–X6 never run). **No sync reply
checks.**

Both steps validate the **first** on_cancel payload in the logs, so the second
on_cancel is never checked (see 2).

Includes the on_confirm order checks — tags 6.1, item tags 6.3, cancellation and
replacement terms, quote, fulfillments, items, and payments with the CF9 status
rule and `params` — plus:

| # | Field | Rule |
|---|---|---|
| CN1 | `order.cancellation` | object |
| CN2 | `order.cancellation.cancelled_by` | string |
| CN3 | `order.cancellation.reason` | object |
| CN4 | `order.cancellation.reason.descriptor` | object |
| CN5 | `order.cancellation.reason.descriptor.code` | string (any value) |
| CN6 | `order.created_at` | string |
| CN7 | `order.updated_at` | string |
| CN8 | `stops[s].authorization` | object (when present) |
| CN9 | `stops[s].authorization.type` | one of `OTP`, `QR` (OPTIONAL) |
| CN10 | `stops[s].authorization.token`, `valid_to`, `status` | strings (OPTIONAL) |
| CN11 | `items[i].descriptor.short_desc/long_desc` | strings (OPTIONAL) |
| CN12 | `items[i].time` | when present: object with `duration` string |
| CN13 | Internal error | if the validator throws, a test "on_cancel request failed to be verified…" fails on purpose |

Differences from on_confirm:
- `order.billing` is **not** checked;
- `provider.descriptor.short_desc` is not checked;
- `SETTLEMENT_AMOUNT` (CF11) is commented out.

## 10. Files present but not used for TRV14

| File | Why unused | What it would check |
|---|---|---|
| `on_update.js` | No EP flow has an on_update step | Ride-hailing order fields: fulfillment `state.descriptor.code` equals the expected ride state; `vehicle.category` (`TWO_WHEELER`, `AUTO_RICKSHAW`, `CAB`) and `variant` (EV, SEDAN, …) and `energy_type`; `order.status` one of `ACTIVE`, `COMPLETED`; `updated_at` equals the previous request's `created_at`; `ROUTE_INFO` and `FARE_POLICY` ride tags; `SETTLEMENT_AMOUNT` |
| `on_track.js` | No EP flow has an on_track step | `message.tracking` object, `status` string, `location` object with `gps` and `updated_at` strings |

## 11. What is not validated for TRV14

These checks are missing from Pramaan's TRV14 module:

1. **Buyer requests**: `search`, `select`, `init`, `confirm`, `status` and `cancel` are not validated at all.
2. **Values across calls (apart from context ids and timestamp)**: none of these
   are compared with the earlier search, select, init or confirm:
   - provider id, item ids, quantities and prices;
   - quote and breakup amounts;
   - billing, payments and order id.
3. **Order status**: `order.status` is not checked in on_confirm, on_status or on_cancel (for example `ACTIVE` or `CANCELLED`).
4. **Fulfillment state**: no check of `fulfillments[].state` in any TRV14 action.
5. **Cancellation reason**: `cancellation.reason.descriptor.code` and `cancelled_by` only need to be strings; no allowed-values check.
6. **Two-step cancellation**: the soft cancel (quote) and the confirmed cancel are not told apart, and the second on_cancel is never checked.
7. **User cancellation (EP_7/EP_8)**: these reuse the technical cancellation checks unchanged.
8. **Ticket and form semantics**:
   - nothing checks that the selected quantity is within the on_search minimum and maximum;
   - nothing checks that `form_response.status` is a valid value;
   - nothing checks that the second on_select carries the `submission_id`;
   - nothing checks that multi-ticket flows (EP_3/EP_4) actually carry more than one ticket.
9. **Timestamps inside the order**: `created_at`/`updated_at` are only checked to be strings; there is no ordering or equality check.
10. **Sync replies**: the ACK/NACK reply is not checked for on_status and on_cancel.

## 12. Bugs and quirks in the TRV14 validators

| # | Location | Issue | Effect |
|---|---|---|---|
| B1 | `testRunner.js` `cancelIndex.on_cancel_technical = 0` | Both on_cancel steps read index 0 | Second on_cancel is never validated; the first is reported twice |
| B2 | `testRunner.js` lines for on_search / on_select fallback | `on_search({}, step, logs, constants)` and `on_select({}, step, logs, constants)` pass `logs` in the `isSelfPickup` position | When the expected log is missing, the fallback suite runs with wrong arguments |
| B3 | `on_select.js`, `on_cancel.js` call `contextTests` without `logs` | X1–X6 cannot run | message_id uniqueness, timestamp order and BAP/BPP consistency are never checked for on_select and on_cancel |
| B4 | `on_search.js` OS9 | `listItem.descriptor.code` compared with itself | Always passes |
| B5 | `on_search.js` OS47 / similar in other files | `fulfillment_ids` "should include" its own element | Always passes |
| B6 | Breakup "positive value" | Assertion is `>= 0` | Zero-value lines pass |
| B7 | GPS regex | on_search provider locations need **exactly 6** decimal places; order locations accept **1–7** | The same coordinate can pass in one call and fail in another |
| B8 | Ride-hailing leftovers | `vehicle.category` + `agent` required on a site visit; `cancellation_terms.fulfillment_state` enum uses ride states; `on_confirm` driver modes / error 90203; FARE_POLICY/INFO ride lists in on_select | Copied from the ride-hailing module; checks may not match the TRV14 spec |
| B9 | `xinput.form.multiple_sumbissions` | Misspelled field name | Checks a field that may not exist under the spec's spelling |
| B10 | "(OPTIONAL)" tests | Still hard assertions | Missing optional fields can show as failures (e.g. `ADD_ONS` breakup line) |
| B11 | on_init `payments[].status` rule | Any listed `type` forces `NOT-PAID`, unlike on_confirm/on_status/on_cancel | A PRE-ORDER payment marked `PAID` at on_init fails |
