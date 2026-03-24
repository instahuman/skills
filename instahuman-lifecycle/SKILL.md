---
name: instahuman-lifecycle
description: Job lifecycle management tools for InstaHuman — pause, resume, cancel, stop, and tester search.
---

# InstaHuman — Job Lifecycle

Extended tools for managing job state. Load this only when you need to pause, resume, stop, cancel a job, or search for testers.

## Tools

### pause_job

Pause an active job. The job becomes invisible to new testers but testers already working can still submit feedback. Resumable.

**Input:** `{ job_id: string uuid }`
**Output:** `{ job: { id, status: "paused", ... } }`

### resume_job

Resume a paused job. Sets it back to active — new testers can discover and join it again.

**Input:** `{ job_id: string uuid }`
**Output:** `{ job: { id, status: "active", ... } }`

### stop_job

Gracefully stop an active or paused job. If testers are still working, the job enters `closing` state — no new testers can join but existing testers can finish. Once all testers finish or their time expires, settlement begins automatically. If no testers are working, settlement begins immediately.

**Input:** `{ job_id: string uuid }`
**Output:** `{ job: { id, status, ... } }`

### cancel_job

Cancel an active, paused, or draft job. If completed assignments exist, begins settlement. Otherwise releases escrow fully. Cannot cancel a job that is already `closing`.

**Input:** `{ job_id: string uuid }`
**Output:** `{ job: { id, status: "cancelled", ... } }`

### list_jobs

List jobs owned by this API key's owner. Cursor-paginated, max 50 per page.

**Input:**

```
status: "draft" | "active" | "paused" | "closing" | "completed" | "cancelled", optional
limit:  integer (1-50), optional (default 20)
cursor: string, optional — opaque cursor from a previous `next` value
```

**Output:** `{ jobs: [{ id, title, status, createdAt, ... }], next: string | null }`

Pass the returned `next` value as `cursor` in the next call to fetch the following page. `next` is `null` when there are no more results.

### search_testers

Find available testers matching criteria. Use before job creation to estimate fill likelihood or narrow targeting.

**Input:**

```
skills:         string[], optional
location:       string, optional (partial match on country/state)
category:       string, optional (interest category slug)
task_embedding: number[] (512-dim), optional (vector similarity search)
confidence:     number 0-1, optional (default 0.5)
limit:          integer 1-100, optional (default 10)
```

**Output:** `{ testers: [{ id, name, skills, location, hourlyRate, bio, similarity? }] }`

## Job Status Flow

```
draft → active → paused → active (resume)
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
