---
name: Manage MoEngage segments and flows
description: Create and maintain file-based and filter-based segments, sync partner cohorts, and read or pause MoEngage flows.
api: openapi/moengage-custom-segments-openapi.yml
operations:
  - createFilterSegment
  - listCustomSegments
  - getCustomSegment
  - updateFilterSegment
  - createFileSegment
  - addUsersToFileSegment
  - removeUsersFromFileSegment
  - replaceUsersInFileSegment
  - archiveCustomSegment
  - unarchiveCustomSegment
  - searchFlows
  - getFlow
  - getFlowVersion
  - updateFlowStatus
---

# Manage MoEngage segments and flows

## Two kinds of segment

**Filter segments** are defined by event and user-attribute conditions and re-evaluate continuously.
**File segments** are static lists of user identifiers you upload and then mutate.

| Task | Operation |
| --- | --- |
| List segments | `listCustomSegments` |
| Read one | `getCustomSegment` |
| Create a filter segment | `createFilterSegment` |
| Update a filter segment | `updateFilterSegment` |
| Create a file segment | `createFileSegment` |
| Add users to a file segment | `addUsersToFileSegment` |
| Remove users | `removeUsersFromFileSegment` |
| Replace the whole membership | `replaceUsersInFileSegment` |
| Archive / restore | `archiveCustomSegment` / `unarchiveCustomSegment` |

Prefer `replaceUsersInFileSegment` over remove-then-add when you are reconciling a full list — it is one
operation against a rate limit that counts operations, not rows.

## Rate limits differ sharply between the two

- **Filter segments**: 50 requests/min, 200/hour, 1,000/day.
- **File segments**: **10 operations/hour, 100/day.** A single file may be up to 150 MB.

Batch file-segment work. Ten operations per hour will not absorb a per-user loop.

## Partner cohorts

`POST /v1/integrations/cohortsync` (`openapi/moengage-cohort-audience-openapi.yml`) synchronizes a cohort
or audience built in a partner platform into MoEngage. `415` from this endpoint means the `Content-Type`
header was missing or unsupported — set `application/json` explicitly.

## Flows

Flows are multi-step cross-channel journeys.

- `searchFlows` — find flows by name, status, or other criteria.
- `getFlow` — read a flow's configuration.
- `getFlowVersion` — read a specific version; flows are versioned and editing produces new versions.
- `updateFlowStatus` — pause or stop a flow.

Flows use the v5 error envelope (`response_id` + `error.code`). There is no public operation to *create*
a flow — authoring stays in the dashboard, and the API covers read plus status control.

## Counting an audience

There is no public REST operation that returns a segment's user count. The count-with-reachability
capability exists only on the MCP server (`start_segment_count` / `poll_segment_count`), which is
OAuth-gated per user. See `mcp/moengage-tool-crosswalk.yml`.
