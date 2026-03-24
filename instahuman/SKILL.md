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

Create a new active job. Returns immediately with the job id and variant summaries.

**Input:**

```
title:                          string, required
description:                    string, optional
description_mime_type:          "text/plain" | "text/markdown", optional
category:                       string, optional (category slug)
project_id:                     string uuid, optional
target_testers:                 integer, required
job_posting_time_limit_minutes: integer, required (min 5, max 10080)
task_paid_time_limit_minutes:   integer, optional (for per_minute pricing)
task_hard_time_limit_minutes:   integer, required (min 1, max 1440)
starts_at:                      string, optional (ISO 8601)
payment_rules:                  array, required
  type:           "fixed" | "per_minute"
  amount_cents:   integer, optional (for fixed)
  rate_cents:     integer, optional (for per_minute)
  cap_minutes:    integer, optional (for per_minute)
legal_agreements:               array, optional
  type:           string
  version:        string, optional
  agreement_id:   string, optional
proof_of_completion:            object | null, optional
  method:         "password"
  password:       string
variants:                       array, required (min 1)
  description:          string, required (task instructions)
  description_mime_type: "text/plain" | "text/markdown", optional
  url:                  string, optional (URL testers visit; required when placement is iframe or screen_capture is set)
  placement:            object, optional
    element:            "window" | "iframe"
    position:           "top" | "bottom" | "left" | "right" (required when element is "iframe")
    reveal_questions_during_test: boolean, optional (default false)
  screen_capture:       array, optional (requires placement.element="iframe" and url)
    type:               "scheduled" | "testerTriggered"
    interval_seconds:   integer (required for scheduled)
    min:                integer, optional (for testerTriggered)
    max:                integer, optional (for testerTriggered)
  feedback_questions:   array, required
    id:                 string
    question_markdown:  string
    input_type:         "text" | "textarea" | "multiple_choice" | "file_upload" | "star_rating" | "switch" | "slider" | "range_slider" | "date" | "integer" | "decimal" | "thumbs"
    required:           boolean, optional
    options:            array (required for multiple_choice, min 2)
      label:            string
      value:            string
      exclusive:        boolean, optional
    multi_select:       boolean, optional (for multiple_choice)
    labels:             array of strings (required for switch [2], slider [2+], range_slider [2+])
    min:                number, optional (for integer, decimal)
    max:                number, optional (for integer, decimal)
    display_condition:  object, optional (conditional display based on another question's answer)
      question_id:      string
      op:               "in" | "not_in"
      values:           string[]
  target_testers_min:   integer, required
  target_testers_max:   integer, optional
  audience_filters:     array, optional
    field:    string
    op:       "in" | "not_in" | "between"
    values:   string[], optional
    min:      number, optional
    max:      number, optional
  tech_constraints:     array, optional
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

**Output:** `{ job: { id, title, status, variants: [{ id, targetTestersMin, assignmentCount, completedCount }] } }`

### wait_job

Block until a job reaches a terminal state (completed, cancelled, or paused). Returns same shape as `get_job`.

**Input:**

```
job_id:           string uuid, required
timeout_ms:       integer, optional (default 1h, max 24h)
```

**Output:** `{ job: { ...jobFields, variants: [{ ...variantFields, completedAssignments: [{ assignmentId, testerId, status, completedAt, durationMinutes, feedbackData }] }] } }`

On timeout, returns error with `job_id`, current `job` snapshot, and `next_action` hint.

### get_job

Get current state of a job including completed assignments with feedback data. Non-blocking.

**Input:** `{ job_id: string uuid }`

**Output:** Same as `wait_job`.

### list_jobs

List jobs owned by this API key's owner.

**Input:**

```
status: "draft" | "active" | "paused" | "closing" | "completed" | "cancelled", optional
limit:  integer (1-100), optional (default 20)
```

**Output:** `{ jobs: [{ id, title, status, createdAt, ... }] }`

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
