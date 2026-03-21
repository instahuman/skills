---
name: instahuman-disputes
description: Settlement, approval, and dispute tools for InstaHuman jobs.
---

# InstaHuman — Settlement & Disputes

Extended tools for post-completion job management. Load this only when a job has completed and you need to approve payouts, dispute results, or inspect settlement status.

## Settlement Flow

When a job completes (all variant targets met, or manually closed):

1. Escrow is adjusted — unused capacity released, actual cost retained
2. Job enters `pending_approval` with a **24-hour auto-approve deadline**
3. Owner must either `approve_job` or `submit_dispute` within the window
4. If neither happens, the job auto-approves at the deadline

## Tools

### close_job

Close an eligible job and begin settlement. A job is closable when: all testers finished, no testers are active, or all active testers exceeded `taskHardTimeLimitMinutes`.

**Input:** `{ job_id: string uuid }`

**Output:**

```
{
  ok: true,
  closedReason: string,
  cancelled: boolean,         // true if no testers completed (full refund)
  settlementStatus: string,   // "pending_approval" or null
  settlementDeadline: string  // ISO 8601 or null
}
```

### approve_job

Approve a `pending_approval` job. Releases escrow, debits actual cost, marks all assignments as approved. **Irreversible.**

**Input:** `{ job_id: string uuid }`
**Output:** `{ ok: true, settlementStatus: "approved" }`

### submit_dispute

Dispute a `pending_approval` job. Non-disputed testers are settled immediately. Disputed testers' funds remain in escrow until resolved.

**Input:**

```
job_id:                          string uuid, required
low_quality:                     boolean, optional
off_topic:                       boolean, optional
suspected_fraud:                 boolean, optional
incomplete_work:                 boolean, optional
platform_issue:                  boolean, optional
platform_issue_description:      string, optional (required if platform_issue)
billing_discrepancy:             boolean, optional
billing_discrepancy_description: string, optional (required if billing_discrepancy)
testers:                         array, optional
  tester_user_id: string uuid
  category:       "low_quality" | "off_topic" | "suspected_fraud" | "incomplete_work"
  description:    string, optional
```

**Output:** `{ ok: true, settlementStatus: "disputed" }`

### get_settlement_status

Get detailed settlement breakdown including escrow, per-assignment status, closability, and any active dispute.

**Input:** `{ job_id: string uuid }`

**Output:**

```
{
  jobId:              string,
  jobStatus:          string,
  escrowHoldCents:    number | null,
  settlementStatus:   string | null,
  settlementDeadline: string | null,
  closable:           boolean,
  closableReason:     string | null,
  assignments: [{
    id:               string,
    testerId:         string,
    settlementStatus: string | null,
    payoutCents:      number | null
  }],
  dispute: {
    status:             string,
    lowQuality:         boolean,
    offTopic:           boolean,
    suspectedFraud:     boolean,
    incompleteWork:     boolean,
    platformIssue:      boolean,
    billingDiscrepancy: boolean
  } | null
}
```

## Rules

- NEVER call `approve_job` without reviewing the feedback data first
- Auto-approve happens at `settlementDeadline` if no action is taken
- `platform_issue` and `billing_discrepancy` affect all testers in the dispute
- Per-tester disputes only affect the flagged testers — others are settled immediately
