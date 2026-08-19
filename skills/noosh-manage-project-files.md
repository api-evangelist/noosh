---
name: Manage project files and folders in Noosh
description: Upload, tag, browse and retrieve artwork and production files on a Noosh project, choosing the right file API generation.
api: openapi/noosh-openapi.yml
operations:
  - getWorkgroupList
  - getProjectList
  - getFolders
  - createFolder
  - getFiles
  - getFile
  - getFileTags
  - uploadFile
  - uploadProfileImage
generated: '2026-08-13'
method: generated
source: openapi/noosh-openapi.yml (harvested from https://api.noosh.com/api/swagger.json)
---

# Manage project files and folders in Noosh

Marketing execution runs on artwork, proofs and production files attached to a project. Noosh publishes
**four different upload routes under one operationId**, which is the single easiest thing to get wrong
here — this skill exists mostly to get that choice right.

## Before you start

- Auth is HTTP Basic (`HTTP_BASIC`), on `https://api.noosh.com`. See
  `authentication/noosh-authentication.yml`.
- Files are scoped to a project inside a workgroup: resolve `workgroup_id` with `getWorkgroupList` and
  `project_id` with `getProjectList` first.
- Uploads are `multipart/form-data` with exactly **two** parts, named `file` and `tags`, per the
  operation summary in the contract.

## Choosing the upload route

All four are published simultaneously and all four are called `uploadFile`. None is marked deprecated,
so the spec gives you no guidance — this is the ordering the paths themselves imply:

| Path | Notes |
|---|---|
| `POST /3/workgroups/{workgroup_id}/projects/{project_id}/files` | **Prefer this.** Newest generation ("Upload File V3 to Project With Notification"), and the only generation that also ships folders. |
| `POST /1.2/workgroups/{workgroup_id}/projects/{project_id}/filesByRole` | Role-targeted upload with notification. |
| `POST /1.1/workgroups/{workgroup_id}/projects/{project_id}/filesByRole` | Superseded by the 1.2 route. |
| `POST /1.1/workgroups/{workgroup_id}/projects/{project_id}/files` | Oldest. Use only against an older tenant that rejects V3. |

Because the operationId is identical across all four, an SDK generated from this contract will collapse
them. Address the path directly rather than trusting a generated method name.

## Steps

1. **Browse folders.** `getFolders` — `GET /3/workgroups/{workgroup_id}/projects/{project_id}/folders`
   lists all folders under a project. Create one with `createFolder`
   (`POST /3/workgroups/{workgroup_id}/projects/{project_id}/folders`). Folders exist only on the `/3`
   generation.

2. **Read the tag vocabulary before you tag.** `getFileTags` —
   `GET /1.1/workgroups/{workgroup_id}/projects/{project_id}/fileTags` returns the tags available on
   that workgroup and project. Send tags from this list on upload; do not invent tag strings.

3. **Upload.** `uploadFile` on the `/3` route, `multipart/form-data`, parts `file` and `tags`.
   The V3 and `filesByRole` routes notify project members; the plain `/1.1/files` route does not.
   Consider that before bulk-loading a hundred files into an active project.

4. **List and retrieve.** `getFiles`
   (`GET /1.1/workgroups/{workgroup_id}/projects/{project_id}/files`) and `getFile`
   (`.../files/{file_id}`). Both are documented as working for **regular and remote** files, so a
   returned record may reference an external location rather than Noosh-hosted bytes — check the record
   before assuming a download URL.

5. **Invoice attachments are a separate surface.** Files attached to an invoice are read with
   `getInvoiceFiles` (`GET /v1/workgroups/{workgroup_id}/projects/{project_id}/invoices/{invoice_id}/files`),
   not with `getFiles`.

6. **Profile images are separate too.** `uploadProfileImage`
   (`POST /v1/workgroups/{workgroup_id}/profileImage`) — workgroup-scoped, not project-scoped.

## Errors and retries

Errors return the `HTTPStatusVO` envelope (`status_code`, `status_reason`), not problem+json.
`404` means "no result matching your search condition", `422` means invalid data, `500` is documented
on every operation. A bare `403 {"message":"Forbidden"}` is AWS API Gateway rejecting unauthenticated
calls, not a Noosh error.

**There is no idempotency key on upload.** A retried `uploadFile` creates a second copy of the file.
On a failed or timed-out upload, call `getFiles` and check for the filename before retrying.
