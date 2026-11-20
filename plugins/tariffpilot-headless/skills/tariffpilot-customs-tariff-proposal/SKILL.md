---
name: tariffpilot-customs-tariff-proposal
description: Classify goods in the customs tariff with TariffPilot through the TariffPilot Manager MCP server, i.e. create a customs tariff proposal, drive it to completion, and explain the proposed tariff code with its reasoning. Use this skill whenever goods should be classified or a tariff code is wanted, for example "classify this product", "which HS code", "customs tariff number", "CN code", "HTS code", "commodity code", "tariff classification", and on German phrasing such as "tarifieren", "einreihen", "Einreihung", "Zolltarifnummer", "Warentarifnummer", "Tarifierungsvorschlag", "welche KN", or a product description plus "in TariffPilot". Also use it to continue or read an existing customs tariff proposal. Use it even for a single quick product, because a proposal only finishes if the assistant keeps it running.
---

# Customs tariff proposals with TariffPilot

A customs tariff proposal is not answered by one tool call. TariffPilot processes it in several steps (product analysis, tariff code, BTIs, rulings, AI database, overlay), a step takes from under a minute to several minutes, and **the proposal pauses after some steps until it is started again**. If you create a proposal and stop, it stays unfinished forever. The core of this skill is the run loop below: keep calling the start tool until the proposal is finished.


## Tools

All tools belong to the TariffPilot Manager MCP server (`https://app-cloud.tariffpilot.com/mcp`). Every tool except `list_tenants` takes `tenant_id`; a JSON request body goes into the argument `body`.

1. `list_tenants`: the tenants the login can reach, with the user's roles.
2. `customs_tariff_proposals_post(tenant_id, body)`: creates the proposal and returns it with its `id`. It does **not** start processing.
3. `customs_tariff_proposal_transit_start_post(tenant_id, id)`: runs the next step(s) synchronously and returns only when they are done. It answers `HTTP 200 without content`. The tariff code step often takes longer than the MCP client waits (about two minutes); the call then fails with a bare `Request POST ... failed`, while the server keeps working. That is expected; see rule 3.
4. `customs_tariff_proposal_get(tenant_id, id)`: the full proposal with its `state` and result, often more than 50 KB. Use it once the proposal is finished.
5. `customs_tariff_proposals_get(tenant_id, search?)`: the latest proposals without the large `result`. Use it for finding a proposal and for polling `state` and `animation_at` while it runs; pick the entry with the proposal's `id`, `search` matches description, reference id and codes.


## 1. Choose the tenant

Creating a proposal consumes a proposal of the tenant's plan, so it must land in the right tenant.

1. Call `list_tenants`.
2. One tenant: use it and name it in your reply. Several tenants: ask the user which one, unless the user already named it; match names case-insensitively.
3. Keep the `tenant_id` for the whole task.


## 2. Collect the input

Build `body` for `customs_tariff_proposals_post` from the user's request:

| Field | Required | Values and rules |
|---|---|---|
| `description` | yes (or `url`) | The goods description, as detailed as the user gave it: material, function, use, composition, dimensions, power. Pass the user's wording; do not invent properties. |
| `direction` | yes | `import` or `export`. Infer it from the request ("Einfuhr", "import into the EU" → `import`; "Ausfuhr", "export to" → `export`). If it is not clear, ask. Never omit it; the server cannot process a proposal without it. |
| `tariff_system` | no | `de` (EU Combined Nomenclature and German TARIC, the default), `ch` (Switzerland), `us` (US HTS), `uk` (UK Trade Tariff), `cn` (China). "EU", "Germany", "KN", "TARIC" → `de`. |
| `destination_country` | only `uk` + `export` | ISO 3166-1 alpha-2, e.g. `US`. Leave it out otherwise. |
| `origin_country` | no | ISO 3166-1 alpha-2, e.g. `CN`. |
| `reference_id` | no | The user's article or material number, if given. |
| `url` | no | A product page URL, if the user gave one. |
| `search_web` | no | `true` only if the user asks to research the product on the web. |
| `source_classification_proposal_id` | no | Only when the user wants to tariff goods of an existing classification proposal; its files are copied and both are linked. |

Leave out all other fields (`model`, `gr2a`, `gr3b*`, `general_use`, `files`, `url_content`, `vendor_id`) unless the user explicitly provides them. If the description is too thin to classify (e.g. "a part"), ask one short question for the missing properties before creating the proposal, because a proposal consumes plan volume.

Before the run, tell the user in one line that TariffPilot needs a few minutes and that you will report the result when it is finished.


## 3. Run loop: keep the proposal running until `state` is `end`

1. Call `customs_tariff_proposals_post(tenant_id, body)`. Keep `data.id`.
2. Call `customs_tariff_proposal_transit_start_post(tenant_id, id)`. **Wait for it.** A call that takes minutes is normal and not a hang. Do not call any other tool on this proposal while it runs.
3. Read `state`, `animation_at` and `data_quality` of the proposal with `customs_tariff_proposals_get(tenant_id, search=<description>)`.
4. If `state` is `end`, the proposal is finished; continue with section 4.
5. If `state` is `code_done`, note `data_quality` now: the next start call clears it (known issue), so the finished proposal may show it empty.
6. Otherwise, post a short progress line to the user (e.g. "Product analysis done, now determining the tariff code ...") and go back to step 2.

A typical proposal needs **three** start calls: the first ends at `product_done`, the second at `code_done`, the third runs through BTIs, rulings, AI database and overlay to `end`, usually within a minute. States in order: `start`, `product_done`, `code_done`, `bti_done`, `ruling_done`, `training_done`, `overlay_done`, `end`. Stop after at most eight start calls and report the state if `end` is still not reached.

Rules:

1. **Never create a second proposal to retry.** Every create consumes plan volume and the first proposal is still there. Always continue the existing `id`.
2. **Never stop after the create.** The proposal shows up as unfinished in the user's Manager otherwise.
3. **If a start call fails or times out on the client side,** the server is usually still working; the tariff code step regularly takes four to six minutes. Never start again while it runs, as a second start would process the same step twice. Instead poll with `customs_tariff_proposals_get` every 30 to 90 seconds (wait between polls if your client can):
   1. `animation_at` is set (epoch milliseconds): the step is still running. Keep polling; tell the user every few polls that TariffPilot is still working.
   2. `animation_at` is empty and `state` has advanced: the step is done. Continue with step 4 of the loop.
   3. `animation_at` is empty and `state` is unchanged for three minutes: call the start tool once more.
4. **If `data.errors` is greater than 0** or `data.result.errors` is not empty after a start call, report the error to the user and stop; do not loop on a failing step.


## 4. Read and present the result

The finished proposal is large (often more than 50 KB), mostly because of `btis` and `result.steps`. Read only the fields you need:

1. `code_effective`: the code that applies (the code of the selected BTI if TariffPilot adopted one, otherwise the proposed code). Lead with it.
2. `code`: the code TariffPilot derived itself. If it differs from `code_effective`, show both and name the BTI from `bti_reference` (and `bti_expired` if true).
3. `proposed_customs_description_en` / `_de` (and `_short_*`): the customs description of the goods.
4. `data_quality` (or the value noted at `code_done` if it is empty now; if both are empty, do not state one): `high`, `medium` or `low`. For `medium` or `low`, say plainly that information was missing and which properties would make the result more reliable.
5. `result.steps[]`: the reasoning per step, with `state_from`, `state_to` and `text_en` / `text_de` (Markdown). Summarise the step `product_done` → `code_done` (headings and positions considered, general interpretative rules applied) and, if present, the BTI step `code_done` → `bti_done`. Answer in the user's language and use the matching `text_*`.
6. Hints worth mentioning when non-empty: `dual_use_rows` (possible dual-use relevance), `cbam`, `eudr`, `hs_revision_changes` / `hs_revision_deleted`, `validity`.
7. Link the proposal: `https://cloud.tariffpilot.com/customs_tariff_proposals/<id>`.

Close with one sentence that this is an AI-generated proposal for the user's own review, not a binding tariff information (BTI).


## Continue or read an existing proposal

1. Find it with `customs_tariff_proposals_get(tenant_id, search)` (searches description, reference id and codes) or take the `id` the user gave.
2. Read it with `customs_tariff_proposal_get`. If `state` is not `end` and `animation_at` is empty, it is paused: continue it with the run loop from step 2. If `animation_at` is set (epoch milliseconds, recent), it is running right now: poll as in rule 3.
3. If `state` is `end`, present it as in section 4.


## Errors

1. `HTTP 403` "Customs tariff proposals limit reached": the plan's volume is used up. Tell the user; do not retry.
2. `HTTP 403` otherwise: the login lacks an editing role (`OWNER`, `EDITOR` or `CUSTOMS_TARIFF_PROPOSALS_EDITOR`) in this tenant. Tell the user to check the role.
3. `HTTP 400`: read the message (e.g. invalid country code, missing description) and fix the input.


## Follow-up

If the user also wants an export control check (dual-use, military goods) for the same goods, use the skill `tariffpilot-classification-proposal` and pass `source_customs_tariff_proposal_id` = this proposal's `id` (only for `tariff_system` `de`), so the files are copied and both proposals are linked.
