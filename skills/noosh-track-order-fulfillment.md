---
name: Track order fulfillment and shipments in Noosh
description: Follow a Noosh buy or sell order through shipment, delivery locations, tasks and invoicing.
api: openapi/noosh-openapi.yml
operations:
  - getWorkgroupList
  - getBuyOrderListOfWorkgroup
  - getBuyOrder
  - putBuyOrder
  - getSellOrderList
  - getSellOrder
  - putSellOrder
  - getOrder
  - getShipmentList
  - getShipment
  - postShipment
  - putShipmentLocation
  - getTaskListOfProject
  - postTaskForProject
  - getInvoices
  - getInvoice
generated: '2026-08-13'
method: generated
source: openapi/noosh-openapi.yml (harvested from https://api.noosh.com/api/swagger.json)
---

# Track order fulfillment and shipments in Noosh

Once a buy order exists, the downstream chain is **order → shipment → locations → invoice**, with tasks
running alongside. Noosh models both sides of the trade: a *buy order* is what you place with a
supplier, a *sell order* is what a supplier or outsourcer receives. Pick the right one for the
workgroup you authenticated as.

## Before you start

- HTTP Basic auth on `https://api.noosh.com`; `workgroup_id` on effectively every call.
- No pagination anywhere — order and shipment lists come back whole.
- No webhooks and no event stream. Noosh publishes no AsyncAPI and no callbacks, so **fulfillment
  tracking is polling.** Choose an interval deliberately: there is no published rate limit and no
  `Retry-After` header to guide you.

## Steps

1. **Find the orders.** Workgroup-wide: `getBuyOrderListOfWorkgroup`
   (`GET /v1/workgroups/{workgroup_id}/buyOrders`) or `getSellOrderListOfWorkgroup`
   (`.../sellOrders`). Project-scoped: `getBuyOrderList` / `getSellOrderList` under
   `/v1/workgroups/{workgroup_id}/projects/{project_id}/`.

2. **Read one order.** `getBuyOrder` / `getSellOrder` at project scope, or
   `getBuyOrderOfWorkgroup` / `getSellOrderOfWorkgroup` at workgroup scope. `getOrder`
   (`GET /v1/workgroups/{workgroup_id}/projects/{project_id}/orders/{order_id}`) is the generic
   read when you do not know which side of the trade the id belongs to.

3. **Amend an order.** `putBuyOrder` / `putSellOrder`. These change commercial commitments — put a
   human in the loop before calling either. There is no idempotency key, so a retried PUT re-applies.

4. **Read shipments.** `getShipmentList`
   (`GET /v1/workgroups/{workgroup_id}/projects/{project_id}/shipments`) and `getShipment` for one.

5. **Create a shipment.** `postShipment`
   (`POST /v1/workgroups/{workgroup_id}/projects/{project_id}/shipments`). Before retrying a
   timed-out create, re-list — a duplicate shipment is a real-world duplicate delivery.

6. **Update a delivery location.** `putShipmentLocation`
   (`PUT /v1/workgroups/{workgroup_id}/projects/{project_id}/shipments/{shipment_id}/locations/{location_id}`).
   A shipment fans out to multiple locations; each is addressed by its own `location_id`.

7. **Track the work alongside it.** `getTaskListOfProject` and `getTaskOfProject` for project tasks,
   `getTaskListOfWorkgroup` / `getTaskOfWorkgroup` for the workgroup view. Create with
   `postTaskForProject`. Valid values come from `getTaskTypesOfWorkgroup`, `getCustomTaskTypesOfWg`,
   `getDefaultTaskStatusList`, `getWgTaskStatusListOfWorkgroup` and `TaskPriorityList`
   (`GET /v1/workgroups/{workgroup_id}/defaultTaskPriority`) — read the vocabulary before writing,
   because there is no enum in the contract to validate against.

8. **Close the loop on billing.** `getInvoices`
   (`GET /v1/workgroups/{workgroup_id}/projects/{project_id}/invoices/orders/{order_id}`) returns the
   invoices for an order; `getInvoice` reads one; `getInvoiceFiles` returns its attachments.
   Recipients come from `getBillingRecipients`. Multi-currency projects should read
   `getExchangeRateList` — and note `postExchangeRate` writes a rate, which is a finance-affecting
   operation and should never be called autonomously.

## Errors

`HTTPStatusVO` (`status_code` + `status_reason`) on `404` / `422` / `500`; the same field pair also
appears on `200`, so branch on the HTTP status, not on the presence of `status_code`. `403` with body
`{"message":"Forbidden"}` is the API gateway, not Noosh. No `429` is documented.
