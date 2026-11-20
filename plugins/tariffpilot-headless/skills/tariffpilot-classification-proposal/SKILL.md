---
name: tariffpilot-classification-proposal
description: Check goods for export control with TariffPilot through the TariffPilot Manager MCP server, i.e. create a classification proposal for military goods, EU dual-use (Annex I of Regulation (EU) 2021/821) and the German national list, drive it to completion, and explain the identified list positions and applicable general licences. Use this skill whenever goods, software or technology should be checked for export control, for example "is this dual-use", "export control classification", "ECCN-like check", "military goods list", "Ausfuhrliste", and on German phrasing such as "Güterklassifizierung", "Exportkontrolle", "Dual-Use prüfen", "Rüstungsgut", "Ausfuhrlistennummer", "AL-Nummer", "Klassifizierungsvorschlag", "Allgemeingenehmigung", or a product description plus "in TariffPilot". Also use it to continue or read an existing classification proposal. Use it even for a single quick product, because a proposal only finishes if the assistant keeps it running.
---

# Classification proposals (export control) with TariffPilot

A classification proposal is not answered by one tool call. TariffPilot processes it in stages (product analysis, military goods, EU dual-use, German dual-use, general licences), a stage takes from under a minute to several minutes, and **the proposal pauses after each stage until it is started again**. If you create a proposal and stop, it stays unfinished forever. The core of this skill is the run loop below: keep calling the start tool until the proposal is finished.


## Tools

All tools belong to the TariffPilot Manager MCP server (`https://app-cloud.tariffpilot.com/mcp`). Every tool except `list_tenants` takes `tenant_id`; a JSON request body goes into the argument `body`.

1. `list_tenants`: the tenants the login can reach, with the user's roles.
2. `classification_proposals_post(tenant_id, body)`: creates the proposal and returns it with its `id`. It does **not** start processing.
3. `classification_proposal_transit_start_post(tenant_id, id)`: runs the next stage(s) synchronously and returns only when they are done. It answers `HTTP 200 without content`. A stage can take longer than the MCP client waits (about two minutes); the call then fails with a bare `Request POST ... failed`, while the server keeps working. That is expected; see rule 3.
4. `classification_proposal_get(tenant_id, id)`: the full proposal with its `state` and result. Use it once the proposal is finished.
5. `classification_proposals_get(tenant_id, search?)`: the latest proposals. Use it for finding a proposal and for polling `state` and `animation_at` while it runs; pick the entry with the proposal's `id`.


## 1. Choose the tenant

Creating a proposal consumes a proposal of the tenant's plan, so it must land in the right tenant.

1. Call `list_tenants`.
2. One tenant: use it and name it in your reply. Several tenants: ask the user which one, unless the user already named it; match names case-insensitively.
3. Keep the `tenant_id` for the whole task.


## 2. Collect the input

Build `body` for `classification_proposals_post` from the user's request:

| Field | Required | Values and rules |
|---|---|---|
| `description` | yes (or `url`) | The description of the goods, as detailed as the user gave it. Export control depends on technical parameters (performance, resolution, frequency, accuracy, materials, encryption, intended use), so pass every parameter the user named. Do not invent values. |
| `good_kind` | no | Which checks run: `all` (military goods, EU dual-use and German national list; the default), `dual_use` or `military`. Set it only if the user restricts the check. |
| `good_type` | no | `good` (physical goods), `software` or `technology` (information for development, production or use). Pass it only if the request makes it clear. |
| `software_description`, `software_function`, `software_relation_to_hardware` | no | For software: what it is, what it does, which hardware it belongs to. |
| `technology_description`, `technology_maturity`, `technology_relation_to_hardware` | no | For technology (know-how, documentation): what it is, its maturity, which hardware it belongs to. |
| `good_condition` | no | `finished_good` or `component`. Pass it only if the request makes it clear. |
| `good_field_of_application`, `good_main_good`, `good_main_good_field_of_application` | no | Field of application, and for components the main good they belong to and its field of application. Pass them when the user names them. |
| `destination_country` | no | ISO 3166-1 alpha-2, e.g. `US`, if the user names one. |
| `reference_id` | no | The user's article or material number, if given. |
| `url` | no | A product page or data sheet URL, if the user gave one. |
| `search_web` | no | `true` only if the user asks to research the product on the web. |
| `source_customs_tariff_proposal_id` | no | Only when the user wants to check goods of an existing customs tariff proposal of the EU tariff (`de`); its files are copied and both are linked. |

Leave out all other fields (`model`, `dual_use_code`, `files`, `url_content`, `vendor_id`) unless the user explicitly provides them. If the description lacks the technical parameters that decide export control (e.g. "a camera" without resolution or frame rate), ask one short question before creating the proposal, because a proposal consumes plan volume.

Before the run, tell the user in one line that TariffPilot needs a few minutes and that you will report the result when it is finished.


## 3. Run loop: keep the proposal running until `state` is `end`

1. Call `classification_proposals_post(tenant_id, body)`. Keep `data.id`.
2. Call `classification_proposal_transit_start_post(tenant_id, id)`. **Wait for it.** A call that takes minutes is normal and not a hang. Do not call any other tool on this proposal while it runs.
3. Read `state` and `animation_at` of the proposal with `classification_proposals_get(tenant_id, search=<description>)`.
4. If `state` is `end`, the proposal is finished; continue with section 4.
5. Otherwise, post a short progress line to the user (e.g. "Military goods checked, now checking EU dual-use ...") and go back to step 2.

States in order: `start`, `product_done`, `military_done`, `dual_use_eu_done`, `dual_use_de_done`, `agg_done`, `end`. With `good_kind` `all`, a typical proposal needs **four to five** start calls; stages not selected by `good_kind` are skipped automatically. Stop after at most eight start calls and report the state if `end` is still not reached.

Rules:

1. **Never create a second proposal to retry.** Every create consumes plan volume and the first proposal is still there. Always continue the existing `id`.
2. **Never stop after the create.** The proposal shows up as unfinished in the user's Manager otherwise.
3. **If a start call fails or times out on the client side,** the server is usually still working. Never start again while it runs, as a second start would process the same stage twice. Instead poll with `classification_proposals_get` every 30 to 90 seconds (wait between polls if your client can):
   1. `animation_at` is set (epoch milliseconds): the stage is still running. Keep polling; tell the user every few polls that TariffPilot is still working.
   2. `animation_at` is empty and `state` has advanced: the stage is done. Continue with step 4 of the loop.
   3. `animation_at` is empty and `state` is unchanged for three minutes: call the start tool once more.
4. **If `data.errors` is greater than 0** after a start call, report the error to the user and stop; do not loop on a failing stage.


## 4. Read and present the result

The finished proposal can be large because of `result`. Read only the fields you need:

1. `identification_leading`: the identification with the highest export control risk (military goods before EU dual-use before German national list). Lead with it; if it is empty, say that no list position was identified.
2. `identification_military_equipment`, `identification_dual_use_eu`, `identification_dual_use_de`: the result per list. Show each list that was checked.
3. `data_quality` (the worst of all lists checked) and per list `data_quality_military_equipment`, `data_quality_dual_use_eu`, `data_quality_dual_use_de`: `high`, `medium` or `low`. While the proposal runs, `data_quality` is empty; read it at `end`. For `medium` or `low`, say plainly which technical parameters were missing and would make the result more reliable.
4. `agg_result`: the general licences (AGG) check, with the BAFA status it was checked against. Mention applicable general licences when present.
5. `result`: the reasoning per stage. Summarise the reasoning of each list in a few sentences, in the user's language.
6. `product.summary*`: the product analysis, only if the user asks how TariffPilot understood the goods.
7. Link the proposal: `https://cloud.tariffpilot.com/classification_proposals/<id>`.

Close with one sentence that this is an AI-generated proposal for the user's own review, not a binding decision of the licensing authority (BAFA).


## Continue or read an existing proposal

1. Find it with `classification_proposals_get(tenant_id, search)` or take the `id` the user gave.
2. Read it with `classification_proposal_get`. If `state` is not `end` and `animation_at` is empty, it is paused: continue it with the run loop from step 2. If `animation_at` is set (epoch milliseconds, recent), it is running right now: poll as in rule 3.
3. If `state` is `end`, present it as in section 4.


## Errors

1. `HTTP 403` "Classification proposals limit reached": the plan's volume is used up. Tell the user; do not retry.
2. `HTTP 403` otherwise: the login lacks an editing role in this tenant. Tell the user to check the role.
3. `HTTP 400`: read the message (e.g. invalid good kind, invalid country code, missing description) and fix the input.


## Follow-up

If the user also wants the customs tariff code for the same goods, use the skill `tariffpilot-customs-tariff-proposal` and pass `source_classification_proposal_id` = this proposal's `id`, so the files are copied and both proposals are linked.
