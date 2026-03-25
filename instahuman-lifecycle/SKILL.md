---
name: instahuman-lifecycle
description: Job lifecycle management tools for InstaHuman — pause, resume, cancel, stop.
---

# InstaHuman — Job Lifecycle

Extended tools for managing job state. Load this only when you need to pause, resume, stop, or cancel a job.

## Tools

### pause_job

Pause an active job. The job becomes invisible to new testers but testers already working can still submit feedback. Resumable.

**Input:** `{ job_id: string uuid }`
**Output:** `{ job: { id, status } }`

### resume_job

Resume a paused job. Sets it back to active — new testers can discover and join it again.

**Input:** `{ job_id: string uuid }`
**Output:** `{ job: { id, status } }`

### stop_job

Gracefully stop an active or paused job. If testers are still working, the job enters `closing` state — no new testers can join but existing testers can finish. Once all testers finish or their time expires, settlement begins automatically. If no testers are working, settlement begins immediately.

**Input:** `{ job_id: string uuid }`
**Output:** `{ job: { id, status } }`

### resize_job

Change the number of tester slots in an active or paused job. Adjusts escrow accordingly.

**Input:** `{ job_id: string uuid, target_testers: integer (≥1) }`
**Output:** `{ previousTargetTesters, newTargetTesters, previousEscrowHoldCents, newEscrowHoldCents, escrowDeltaCents, job: { id, status, ... } }`

**Rules:**
- Increasing: places additional escrow hold (requires sufficient balance)
- Decreasing: releases excess escrow as `escrow_release_unused`
- Cannot reduce below the number of already-taken slots (pending + started + completed assignments)
- Cannot reduce below the sum of variant minimums
- If the new target is already met (enough completed assignments, no active testers), settlement begins automatically
- If the new target is met but testers are still working, the job transitions to `closing`
- Ledger entries include a description indicating the resize operation and slot delta

### cancel_job

Cancel an active or paused job. If completed assignments exist, begins settlement. Otherwise releases escrow fully. Cannot cancel a job that is already `closing`.

**Input:** `{ job_id: string uuid }`
**Output:** `{ job: { id, status } }`

### list_jobs

List jobs owned by this API key's owner. Returns lean summaries — use `get_job` for full details. Null/empty/zero fields are omitted.

**Input:**

```
status: "active" | "paused" | "closing" | "completed" | "cancelled", optional
limit:  integer (1-50), optional (default 20)
cursor: string, optional — opaque cursor from a previous `next` value
```

**Output:** `{ jobs: [{ id, title, status, targetTesters, projectId, paymentRules, escrowHoldCents, settlementStatus, createdAt }], next: string | null }`

Pass the returned `next` value as `cursor` in the next call to fetch the following page. `next` is `null` when there are no more results.

## Job Status Flow

```
active → paused → active (resume)
       → closing → completed (settlement)
       → cancelled
```

- `active`: visible to testers, accepting new assignments
- `paused`: hidden from new testers, existing testers can finish
- `closing`: no new testers, waiting for active testers to finish
- `completed`: all variant targets met or settlement triggered
- `cancelled`: job terminated, escrow released or settled

## Project Tools

### create_project

Create a project to group related jobs. `{ name: string, description?: string }`

### assign_job_to_project

Assign/reassign/unlink a job. `{ job_id: string uuid, project_id: string uuid | null }`

### list_project_testers

List testers associated with a project (auto-added on assignment completion). Shows `trusted`, `highValue`, `blocked` flags.

### update_project_tester

Update advisory flags. `{ project_id, tester_id, trusted?, high_value?, blocked? }`. Blocked testers cannot see jobs in this project.
