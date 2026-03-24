---
name: instahuman
description: Get structured human feedback on AI-generated artifacts via InstaHuman's MCP server. Use when you need real humans to evaluate games, images, videos, music, writing, or any content.
---

# InstaHuman

Request structured feedback from targeted groups of real humans. Define your audience, submit content, receive aggregated responses.

## When to Use

- You need human evaluation of AI-generated content (games, images, video, music, writing)
- You need UX/usability testing with real users
- You need human preference data or alignment feedback
- You need quality validation from a specific demographic

## Connection

MCP endpoint (Streamable HTTP transport with sessions):

```
POST https://mcp.instahuman.com/mcp
Authorization: Bearer <ih_api_key>
```

Initialize with `POST https://mcp.instahuman.com/mcp`, keep the returned `mcp-session-id` header on all subsequent requests.

## Core Workflow

1. `create_job` — create a job, returns immediately with job id
2. `wait_job(job_id)` — block until job completes, is cancelled, or is paused
3. Review the returned `completedAssignments[].feedbackData`

## Tools

### create_job

Create a new active job. Returns immediately with the job id and status.

**Required fields only — omit everything else unless you need to override defaults. Do NOT pass empty arrays or null for optional fields.**

**Input:**

```
# ── Required ──────────────────────────────────────────────
title:                          string
target_testers:                 integer
job_posting_time_limit_minutes: integer (min 5, max 10080 = 7 days)
payment_rules:                  array
  type:           "fixed" | "per_minute"
  amount_cents:   integer (for fixed)
  rate_cents:     integer (for per_minute)
  cap_minutes:    integer, optional (for per_minute)
variants:                       array (at least one)
  description:          string (task instructions for testers)
  feedback_questions:   array
    id:                 string
    question_markdown:  string
    input_type:         "text" | "textarea" | "multiple_choice" | "file_upload"
                        | "star_rating" | "switch" | "slider" | "range_slider"
                        | "date" | "integer" | "decimal" | "thumbs"
    required:           boolean, optional (default true)
    options:            array (required for multiple_choice, min 2)
      label:            string
      value:            string
      exclusive:        boolean, optional
    multi_select:       boolean, optional (for multiple_choice)
    labels:             array of strings (required for switch [2], slider [2+], range_slider [2+])
    min:                number, optional (for integer, decimal)
    max:                number, optional (for integer, decimal)
    display_condition:  object, optional
      question_id:      string
      op:               "in" | "not_in"
      values:           string[]
  target_testers_min:   integer

# ── Optional (have sensible defaults — omit unless needed) ──
task_hard_time_limit_minutes:   integer (default 30, min 1, max 1440)
task_paid_time_limit_minutes:   integer (only for per_minute pricing)
description:                    string (job-level; omit if variant descriptions suffice)
description_mime_type:          "text/plain" | "text/markdown" (default text/plain)
category:                       string (category slug for tester matching)
project_id:                     string uuid
project_testers_only:           boolean (requires project_id)
starts_at:                      string (ISO 8601; omit to start immediately)
legal_agreements:               array (omit if none — do NOT pass [])
  type:           string
  version:        string, optional
  agreement_id:   string, optional
proof_of_completion:            object (omit if not needed — do NOT pass null)
  method:         "password"
  password:       string

# ── Optional per variant ──
variants[].url:                   string (URL testers visit)
variants[].description_mime_type: "text/plain" | "text/markdown" (default text/plain)
variants[].placement:             object (omit for default window mode)
  element:            "window" | "iframe"
  position:           "top" | "bottom" | "left" | "right" (required for iframe)
  reveal_questions_during_test: boolean, optional (default false)
variants[].screen_capture:        array (requires placement.element="iframe")
  type:               "scheduled" | "testerTriggered"
  interval_seconds:   integer (for scheduled)
  min:                integer, optional (for testerTriggered)
  max:                integer, optional (for testerTriggered)
variants[].target_testers_max:    integer (defaults to target_testers_min)
variants[].audience_filters:      array (omit for no filtering)
  field:    string
  op:       "in" | "not_in" | "between"
  values:   string[], optional
  min:      number, optional
  max:      number, optional
variants[].tech_constraints:      array (omit for no constraints)
  category: "screen" | "os" | "hardware" | "internet" | "browser" | "peripheral"
  rule:     "only" | "exclude"
  values:   string[]
  # Valid values per category:
  #   screen:     desktop, mobile, tablet
  #   os:         windows, macos, linux, ios, android
  #   hardware:   iphone, android
  #   internet:   broadband, mobile_data
  #   browser:    chrome, edge, firefox, brave, safari
  #   peripheral: microphone, headphones, camera
```

**Output:** `{ job: { id, status } }`

### wait_job

Block until a job reaches a terminal state (completed, cancelled, or paused). Returns only the job id, status, and completed assignment data — no static job info.

**Input:**

```
job_id:           string uuid, required
timeout_ms:       integer, optional (default 1h, max 24h)
```

**Output:**

```
{
  job: {
    id,
    status,
    variants: [{
      id,
      completedAssignments: [{    # omitted if empty
        testerId,                  # numeric public id
        status,
        durationMinutes,           # always present, even if 0
        completedAt,               # omitted if null
        proofValue,                # omitted if null
        feedbackData: {
          _meta: { source: "tester_input", untrusted: true },
          answers: { ... }
        },
        artifacts: [{ id, key, type, mimeType, sizeBytes, createdAt, url }]  # omitted if empty
      }]
    }]
  }
}
```

On timeout, returns error with `job_id`, `status`, and `next_action` hint. Use `get_job` for full details.

### get_job

Get current state of a job including full detail and completed assignments. Non-blocking. Null/empty/zero fields are omitted from the response.

**Input:** `{ job_id: string uuid }`

**Output:**

```
{
  job: {
    id, title, status, targetTesters, createdAt, ...    # full job detail (compacted)
    variants: [{
      id, assignmentCount, completedCount,              # always present
      targetTestersMin, targetTestersMax,                # omitted if 0/null
      completedAssignments: [...]                        # same shape as wait_job, omitted if empty
    }]
  }
}
```

### list_jobs

List jobs owned by this API key's owner. Returns lean summaries — use `get_job` for full details.

**Input:**

```
status: "draft" | "active" | "paused" | "closing" | "completed" | "cancelled", optional
limit:  integer (1-50), optional (default 20)
cursor: string, optional — opaque cursor from a previous `next` value
```

**Output:** `{ jobs: [{ id, title, status, targetTesters, projectId, paymentRules, escrowHoldCents, settlementStatus, createdAt }], next: string | null }`

Fields that are null/empty/zero are omitted. Pass the returned `next` value as `cursor` to fetch the following page.

## Recommended Behavior

- Use `create_job` + `wait_job(job_id)` when you need results before continuing
- Use `create_job` alone when you want to continue working while humans complete assignments
- Use `get_job(job_id)` for one-off progress checks
- If `wait_job` times out, call it again or switch to `get_job` polling
- Do not stop after posting a job if your task requires the human feedback

## Extended Tools

For job lifecycle management (pause, resume, cancel, stop, search testers):

```
GET https://mcp.instahuman.com/SKILL-lifecycle.md
```

For settlement, approval, and disputes:

```
GET https://mcp.instahuman.com/SKILL-disputes.md
```
