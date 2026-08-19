---
name: Source a print job through Noosh
description: Take a marketing print/POS job from a project and a spec through RFQ, supplier quotes, and a buy order using the Noosh API.
api: openapi/noosh-openapi.yml
operations:
  - getWorkgroupList
  - getProjectList
  - postProject
  - getSpecTemplateList
  - postSpec
  - getSpecList
  - getRfqList
  - getRfq
  - getQuoteStateList
  - getQuoteList
  - getQuote
  - putQuote
  - postBuyOrder
  - getBuyOrder
generated: '2026-08-13'
method: generated
source: openapi/noosh-openapi.yml (harvested from https://api.noosh.com/api/swagger.json)
---

# Source a print job through Noosh

Noosh is marketing-execution and print-procurement software. The sourcing chain is
**project → spec → RFQ → quote → buy order**, and every one of those objects lives inside a
*workgroup*. This skill covers the buyer side of that chain.

## Before you start

- **Auth is HTTP Basic and nothing else.** The contract declares one securityDefinition, `HTTP_BASIC`.
  Send `Authorization: Basic <base64(user:password)>` on every request. There is no OAuth, no API key
  and no scope model, so you cannot narrow a token — credentials carry the user's full Noosh
  permissions. Treat them accordingly.
- **Host:** `https://api.noosh.com`. The published document declares the placeholder host
  `example.com:80`; the real host is where the spec and Swagger UI are served. Unauthenticated calls
  return `403 {"message":"Forbidden"}` from AWS API Gateway — that is not a documented API error, it
  means your credentials never reached the application.
- **Everything is workgroup-scoped.** `workgroup_id` is a path parameter on 105 of the 107 operations.
  Resolve it first and carry it through.
- **There is no idempotency key.** Every POST in this flow (`postProject`, `postSpec`, `postBuyOrder`)
  is unconditionally re-executed on retry. Before retrying a failed create, re-read the list operation
  and check whether the object already exists.
- **There is no pagination.** List operations return the whole collection. Plan for large responses on
  busy workgroups rather than paging.

## Steps

1. **Resolve the workgroup.** `getWorkgroupList` — `GET /v1/workgroups`. Optional query filters:
   `workgroup_name`, and `workgroup_types` where `1000001` = Buyer, `1000002` = Supplier,
   `1000003` = Agent, `1000004` = Broker/Outsourcer, `1000005` = Partner. You are the buyer, so filter
   to `1000001` when you manage several. Keep `workgroup_id`.

2. **Find or create the project.** `getProjectList` — `GET /v1/workgroups/{workgroup_id}/projects` —
   and match on the job. If it does not exist, create it with `postProject`
   (`POST /v1/workgroups/{workgroup_id}/projects`). Valid category and status values come from
   `getProjectCategoryList` and `getProjectStatus`; read them first rather than sending free text.
   Keep `project_id`.

3. **Attach the specification.** A spec is what suppliers actually bid on. Look at
   `getSpecTemplateList` (`GET /v1/workgroups/{workgroup_id}/specTemplates`) for a reusable template,
   and at `getSpecTypeFields` for the fields a given spec type requires, then create with `postSpec`
   (`POST /v1/workgroups/{workgroup_id}/projects/{project_id}/specs`). Verify with `getSpecList`.
   To amend an existing spec use `putSpec` — prefer the `/1.1/` route
   (`PUT /1.1/workgroups/{workgroup_id}/projects/{project_id}/specs/{spec_id}`), which is the newer
   generation of the same operationId.

4. **Read the RFQs on the project.** `getRfqList`
   (`GET /v1/workgroups/{workgroup_id}/projects/{project_id}/rfqs`) then `getRfq` for one.
   **The API is read-only on RFQs** — there is no create/update RFQ operation published. RFQ issue is
   a web-application action; the API lets an agent observe it, not drive it. Say so rather than
   looping.

5. **Collect the quotes that come back.** `getQuoteList` at project scope
   (`GET /v1/workgroups/{workgroup_id}/projects/{project_id}/quotes`) or workgroup scope
   (`GET /v1/workgroups/{workgroup_id}/quotes` — same operationId, different path). To filter by state,
   first call `getQuoteStateList` (`GET /v1/workgroups/{workgroup_id}/quoteStates`) for valid ids, then
   pass the documented form `filters={"quote_state_id":111111}`. Never guess a state id.
   Read a single quote with `getQuote`.

6. **Act on a quote.** `putQuote`
   (`PUT /v1/workgroups/{workgroup_id}/projects/{project_id}/quotes/{quote_id}`). This changes
   commercial state — surface the quote total and supplier to a human before calling it.

7. **Raise the buy order.** `postBuyOrder`
   (`POST /v1/workgroups/{workgroup_id}/projects/{project_id}/buyOrders`). Confirm with `getBuyOrder`
   on the returned `order_id`; amend with `putBuyOrder`. Workgroup-wide views are
   `getBuyOrderListOfWorkgroup` / `getBuyOrderOfWorkgroup`.

## Errors

Noosh does not use RFC 9457. Every documented error returns the `HTTPStatusVO` envelope —
`status_code` plus `status_reason` — and the same pair is also present on `200` responses, so **do not
treat the presence of `status_code` as failure.** Branch on the HTTP status:

| Status | Meaning | What to do |
|---|---|---|
| `404` | "There are not any result matching your search condition" | An empty result, not always a missing object. Re-check `workgroup_id`/`project_id` before reporting the object as absent. |
| `422` | "Invalid data" | Your body failed validation. Re-read the required fields from `getSpecTypeFields` / the relevant list operation. Do not retry unchanged. |
| `500` | Internal server error | Documented on all 107 operations. Retry with backoff — but see the idempotency warning: never blind-retry a POST. |
| `403` | Not in the spec | AWS API Gateway rejecting the call before it reaches Noosh. Fix credentials; retrying will not help. |

There is no `429` and no rate-limit header anywhere in the contract, so you have no backoff signal.
Be conservative with request volume.
