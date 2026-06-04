# Infrastructure Issue Pattern Detection Workflow

A systematic approach to identifying whether a support ticket represents an isolated customer issue or a systemic infrastructure failure across an entire cluster, data center, or service pod.

---

## Overview

Every day, support teams investigate tickets that *look* like customer problems but are actually infrastructure failures. A customer can't export data — is their config wrong, or is the database cluster down? A user can't log in — is their SSO misconfigured, or is the authentication service degraded?

This workflow answers that question in **2 minutes** before you spend 2 hours troubleshooting the wrong thing.

**Core principle:** Before investigating ticket details, check if the infrastructure node is broken.

---

## Project Scope

### What This Covers
- ✅ Historical ticket pattern detection across clustered infrastructure
- ✅ Systemic failure identification (3+ same-type issues in a tight timeframe)
- ✅ Risk severity assessment (isolated vs infrastructure-level)
- ✅ Escalation routing (which team should fix this)
- ✅ Pattern trend tracking (is this pod getting worse?)

### What This Doesn't Cover
- ❌ Root cause investigation (that comes after)
- ❌ Fix validation
- ❌ Future capacity planning

---

## How It Works: The 5-Step Process

### **Step 1: Identify the Infrastructure Node**

**Goal:** Extract which cluster, pod, shard, server, or logical partition the ticket belongs to.

**Sources to check:**
- Ticket subject: "Database pod 5 error"
- Ticket tags: `prod-cluster-2`, `shard-us-west`
- Customer metadata: account → infrastructure assignment database
- API logs: request path includes node identifier
- Your message: "customer is on pod 3"

**Normalize to a standard format** for consistent searching. Examples:
- "POD-5" (numeric pod)
- "CLUSTER-us-east-1" (region-based)
- "SHARD-12" (data shard)
- "NODE-db-primary-03" (physical node)

**If unclear:** Ask the user before proceeding. Do not guess infrastructure assignment.

---

### **Step 2: Define the Search Window**

**Goal:** Determine how far back to look for related tickets.

**Standard windows:**
- **30 days:** Tight window, catches acute failures (service just started failing)
- **90 days:** Standard window, catches persistent issues (slow-burn degradation)
- **180 days:** Extended window, used when 90-day results are sparse (< 5 tickets)

**Decision logic:**
```
IF (tickets found in 90 days) > 5:
  Use 90-day window

ELSE IF (tickets found in 90 days) >= 1 and < 5:
  Use 90-day window (still valid)

ELSE IF (tickets found in 90 days) == 0:
  Expand to 180-day window
  Flag as "historical pattern check"
```

**Default:** Always start with 90 days. Only expand if initial search is empty.

---

### **Step 3: Search Historical Tickets**

**Goal:** Find all support tickets historically associated with this infrastructure node.

**Search strategy:**
Use your ticketing system's native search or an internal logging tool. Two parallel queries for coverage:

**Query A — Exact node identifier:**
```
infrastructure_pod:"POD-5" 
OR node:"POD-5"
created:>90d
type:ticket
```

**Query B — Keyword variants:**
```
subject:("pod 5" OR "pod5" OR "POD5" OR "POD-5")
description:("pod 5" OR "pod5" OR "POD5" OR "POD-5")
created:>90d
type:ticket
```

**Constraints:**
- Return up to **50 tickets** across both searches
- **Deduplicate** by ticket ID
- Sort by date (newest first)
- Exclude the current ticket being investigated (to avoid self-reference)

**If sparse results (< 3 tickets):**
- Try alternate pod naming: "pod{N}", "POD {N}", "p{N}", "shard{N}", "cluster{N}"
- Check if the pod is multi-product — search on each product separately
- Widen to 180-day window before concluding "no history"

---

### **Step 4: Categorize Issues**

**Goal:** Classify each historical ticket into an issue type to spot patterns.

**Standard categories for cloud/SaaS platforms:**

| Category | Description | Keywords |
|---|---|---|
| **Query / Data Retrieval Failure** | Export, report, data pull, read operations fail | "export", "query", "timeout", "error occurred", "result set", "ORA-", "could not fetch" |
| **Write / Data Mutation Failure** | Create, update, delete operations fail | "cannot save", "failed to create", "update error", "database write", "persist" |
| **Send / Delivery Failure** | Outbound operations (email, SMS, API calls) fail | "not delivered", "bounce", "queue stuck", "send failed", "API rejected" |
| **Authentication / Login Failure** | Login, SSO, session, token validation fails | "cannot login", "401", "unauthorized", "session expired", "SSO failed" |
| **UI / Rendering Failure** | Web interface errors, forms broken, content won't display | "drag and drop", "preview broken", "form error", "UI freeze", "component crash" |
| **Performance Degradation** | Slow responses, timeouts, capacity-related | "slow", "503", "timeout", "taking forever", "lagging", "unresponsive" |
| **Data Integrity / Sync Failure** | Data mismatch, replication lag, sync issues | "missing data", "sync failed", "inconsistent", "replication lag", "data loss" |
| **Infrastructure / Deployment** | Maintenance, deployment, migration issues | "maintenance window", "upgrade", "rollout", "deployment failed" |
| **Other** | Doesn't fit above categories | — |

**Classification rule:**
- Read the ticket **subject and description summary** (not full resolution)
- Pick the **dominant category** (if a ticket mentions multiple issues, choose the primary one)
- When in doubt, ask: "What stopped working first?" — that's the category

**Examples:**
- "Customer can't export reports due to database timeout" → **Query / Data Retrieval Failure**
- "Login page shows 503 error during peak hours" → **Performance Degradation** (or **Authentication**, depending on root cause; if the page doesn't load, it's Performance)
- "Bulk import stuck in processing" → **Write / Data Mutation Failure**

---

### **Step 5: Build the Pattern Assessment**

**Goal:** Synthesize findings into a clear systemic risk flag and recommendation.

**Structure:**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  🖥️ Infrastructure Pattern Assessment
  Node: [POD-5 | CLUSTER-us-west | SHARD-7]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Lookback window: 90 days
Historical tickets found: N
Current ticket under investigation: #XXXXXX

## Issue Frequency by Category

| Category | Count | % of Pod Tickets |
|---|---|---|
| Query / Data Retrieval | 4 | 44% |
| Write / Data Mutation | 2 | 22% |
| Performance Degradation | 2 | 22% |
| Other | 1 | 11% |

## Recent Tickets (newest first)

| Ticket | Customer | Subject | Category | Status | Date |
|---|---|---|---|---|---|
| #409234 | OrgA | Export takes 5 min | Query / Data Retrieval | Solved | 2025-11-14 |
| #408891 | OrgB | Cannot export dataset | Query / Data Retrieval | Solved | 2025-11-12 |
| #408456 | OrgC | Export times out | Query / Data Retrieval | Open | 2025-11-10 |
| #407823 | OrgD | Slow API responses | Performance | Solved | 2025-11-05 |

## Systemic Risk Assessment

**Diagnosis:** 4 Query/Data Retrieval failures on POD-5 in 28 days (newest: Nov 14, oldest: Nov 1).

**Flag:** 🔴 **SYSTEMIC — Escalate to Engineering**

**Reasoning:** Threshold criteria met:
- Same category (Query/Data Retrieval)
- Tight timeframe (28 days < 30-day threshold)
- Multiple distinct customers
- Pattern matches current ticket (#XXXXXX)

This indicates an infrastructure-level issue on POD-5 affecting data retrieval operations. Not a customer misconfiguration.

## Recommended Action

### Immediate (Next Step)
- [ ] Do NOT investigate customer config — this is infrastructure-level
- [ ] Open/reference Jira incident: "POD-5 Query Timeout Cluster"
- [ ] Assign to Infrastructure/Database team
- [ ] Notify on-call engineer

### Parallel (Support Team)
- [ ] Notify customer: "We've identified a platform issue on your infrastructure node. Engineering is investigating."
- [ ] Add internal note to ticket linking to incident Jira key
- [ ] Link all 4 related tickets to the incident

### Follow-up
- [ ] Engineering provides ETA for fix
- [ ] Re-test after fix is deployed
- [ ] Close tickets once remediated
```

---

## Systemic Risk Thresholds

Use this decision matrix to assign a flag:

| Condition | Flag | Action |
|---|---|---|
| **3+ tickets, same category, within 30 days** | 🔴 Systemic | Escalate to Engineering immediately. File incident. |
| **3+ tickets, same category, within 90 days** | 🟡 Pattern Detected | Create Jira tracking ticket. Monitor closely. Consider escalation if trend continues. |
| **2 tickets, same category, within 30 days** | 🟠 Possible Pattern | Cross-check both tickets' root causes. If similar, escalate. If different, note correlation. |
| **No repeated categories** | 🟢 No Pattern | Proceed with standard investigation. Issue is isolated. |

**Important notes:**
- **Self-exclusion:** If the current ticket you're investigating is in the historical results, exclude it from the count. You're checking for *prior* patterns, not counting yourself.
- **Multi-product considerations:** If the platform runs multiple services/products, track patterns separately per product. A Query failure in Product A and a Query failure in Product B may be independent.
- **Time decay:** Older tickets (80+ days) matter less than recent ones. A pattern of 6 Query failures over 180 days is less urgent than 3 in 30 days (slope matters).

---

## Decision Tree: Is This Systemic?

```
Ticket received for investigation.
│
├─ Step 1: Extract infrastructure node
│  └─ Clear node identifier? YES → Continue | NO → Ask user
│
├─ Step 2: Search historical tickets (90 days)
│  └─ Found >= 1 ticket? YES → Continue | NO → Expand to 180 days, try again
│
├─ Step 3: Categorize all historical tickets
│  └─ Any category appears 3+ times in 30 days?
│     ├─ YES (🔴) → Systemic failure
│     │  ├─ Escalate to Engineering
│     │  ├─ File incident Jira
│     │  └─ SKIP customer investigation
│     │
│     └─ NO → Any category appears 3+ times in 90 days?
│        ├─ YES (🟡) → Pattern detected
│        │  ├─ Create tracking Jira
│        │  ├─ Monitor for escalation
│        │  └─ Proceed with investigation (may be customer + infrastructure)
│        │
│        └─ NO → Any category appears 2+ times in 30 days?
│           ├─ YES (🟠) → Possible pattern
│           │  ├─ Investigate both tickets in detail
│           │  ├─ Compare root causes
│           │  └─ Escalate if similar, proceed if different
│           │
│           └─ NO (🟢) → Isolated incident
│              └─ Proceed with standard investigation
```

---

## Use Cases & Trigger Phrases

**Use this skill whenever:**

1. **Explicit pod/node mention:**
   - "Investigate ticket #408045 for POD-5"
   - "Customer is on SHARD-12, having issues"

2. **Ambiguous pod issues:**
   - "This customer says they can't export data"
   - "Login page is timing out for some users"
   - "API responses are slow"

3. **User asks about patterns:**
   - "Have we seen this before?"
   - "Is this just them or others too?"
   - "Is this a pod issue?"
   - "Should I escalate this to infrastructure?"

4. **Known problematic issue types:**
   - Any Query/Export timeout → check for pod patterns
   - Any authentication failure → check for pod patterns
   - Any performance complaint → check for pod patterns
   - Any data sync/integrity issue → check for pod patterns

**Do NOT trigger:**
- Questions about a specific customer's config (unless you're checking for patterns first)
- Requests to investigate ticket content in detail (that comes after pattern check)
- Questions unrelated to infrastructure issues (billing, feature requests, etc.)

---

## Sample Output: Three Scenarios

### Scenario A: 🔴 Systemic Pattern Detected

```
Infrastructure Pattern Assessment — POD-5
Lookback: 90 days | Tickets found: 9

Issue Frequency:
- Query / Data Retrieval Failure: 4 (44%)
- Performance Degradation: 3 (33%)
- Write / Data Mutation: 2 (22%)

Recent Tickets:
#409234 (Nov 14, OrgA) — Export timeout
#408891 (Nov 12, OrgB) — Export timeout
#408456 (Nov 10, OrgC) — Export timeout
#407823 (Nov 5, OrgD) — API slow

FLAG: 🔴 SYSTEMIC — Escalate to Engineering
─────────────────────────────────────────
4 Query/Data Retrieval failures on POD-5 in 28 days.
Same issue type, tight timeframe, multiple customers.

RECOMMENDATION:
- DO NOT investigate customer config
- Open incident: "POD-5 Query Retrieval Timeout"
- Assign to Infrastructure team
- Notify customer of platform issue
```

### Scenario B: 🟡 Pattern Detected, But Looser Timeframe

```
Infrastructure Pattern Assessment — CLUSTER-us-west
Lookback: 90 days | Tickets found: 6

Issue Frequency:
- Write / Data Mutation: 3 (50%)
- Other: 3 (50%)

Recent Tickets:
#409145 (Nov 12, OrgX) — Bulk import stuck
#408234 (Oct 28, OrgY) — Data sync failed
#407891 (Oct 8, OrgZ) — Replication lag

FLAG: 🟡 PATTERN DETECTED — Monitor
──────────────────────────────────
3 Write/Mutation failures on CLUSTER-us-west in 65 days.
Same issue type but spread across 90 days (not acute).

RECOMMENDATION:
- Create Jira tracking issue: "CLUSTER-us-west Write Failures"
- Proceed with customer investigation
- If customer issue is similar root cause, escalate both
- Monitor for acceleration (if becomes 3-in-30, escalate)
```

### Scenario C: 🟢 No Pattern — Isolated Issue

```
Infrastructure Pattern Assessment — SHARD-3
Lookback: 90 days | Tickets found: 2

Issue Frequency:
- Query / Data Retrieval: 1 (50%)
- Performance Degradation: 1 (50%)

Recent Tickets:
#408123 (Sep 15, OrgM) — Export slow
#407234 (Aug 20, OrgN) — Login timeout

FLAG: 🟢 NO PATTERN — Isolated Incident
────────────────────────────────────────
Different issue types. No cluster of same category.
Each ticket likely has a distinct root cause.

RECOMMENDATION:
- Proceed with standard investigation
- Investigate current ticket's specific customer config
- No infrastructure escalation needed
```

---

## Workflow Checklist

Use this for every ticket:

```
[ ] Pattern Detection
    [ ] Infrastructure node identified (POD, SHARD, CLUSTER, etc.)
    [ ] Historical search executed (90 or 180 days)
    [ ] Tickets found: ___

[ ] Categorization
    [ ] All historical tickets categorized into issue types
    [ ] Distribution documented

[ ] Risk Assessment
    [ ] Systemic threshold check: 3+ same category in 30 days? ___
    [ ] Pattern threshold check: 3+ same category in 90 days? ___
    [ ] Possible pattern check: 2+ same category in 30 days? ___
    [ ] Flag assigned (🔴 / 🟡 / 🟠 / 🟢)

[ ] Recommendation
    [ ] Action determined (escalate vs proceed)
    [ ] Jira ticket created (if 🔴 or 🟡)
    [ ] Customer notification drafted (if 🔴)
    [ ] Handoff ready to next team

[ ] Follow-up
    [ ] Related tickets linked to incident (if 🔴)
    [ ] Pattern tracked in project dashboard
    [ ] Closure criteria defined (when is this fixed?)
```

---

## Impact Metrics

Track these to measure the skill's value:

| Metric | Target | Why It Matters |
|---|---|---|
| **Avg time to escalate systemic issues** | < 10 min | Faster incident detection |
| **% of tickets preventing redundant investigation** | > 30% | Time saved per ticket |
| **% of escalated tickets that are actually systemic** | > 70% | Signal quality (avoid false alarms) |
| **Reduction in duplicate root causes solved** | > 50% | Engineering efficiency |
| **Customer tickets resolved per incident (after pattern escalation)** | > 3 | Multiplier effect |

---

## Common Pitfalls & How to Avoid

| Pitfall | Why It Happens | How to Avoid |
|---|---|---|
| **Threshold too sensitive** — Flags every 2-ticket coincidence | Trying to catch *all* patterns | Stick to the thresholds: 3 in 30 days minimum |
| **Self-reference** — Counts current ticket in historical pattern | Easy to miss when checking | Explicitly exclude current ticket ID from count |
| **Stale patterns** — Flags a cluster from 60 days ago that's been fixed | Time decay not applied | Weight recent tickets heavier; note age in assessment |
| **Wrong infrastructure node** — Checks the wrong pod | Customer metadata wrong or assumed | Always ask for confirmation if unclear |
| **Category confusion** — Misclassifies "slow query" as "performance" vs "query failure" | Boundary between categories unclear | Use keywords as guide; when unsure, default to the *symptom* not the *presumed cause* |
| **No context in escalation** — Passes brief to engineering without linking tickets | Rushed handoff | Always include list of recent related tickets and links |

---

## Integration Points

### With Other Workflows

**After Pattern Detection → Issue Investigation:**
- If 🟢 (no pattern): Proceed to standard ticket investigation
- If 🟠 (possible pattern): Do root cause analysis, compare with prior tickets
- If 🟡 (pattern detected): Escalate but continue supporting customer
- If 🔴 (systemic): Stop investigation, escalate entirely

**Ticketing System Integration:**
- Tag tickets with infrastructure node: `pod-5`, `cluster-us-west`
- Create custom field: `infrastructure_node` (searchable)
- Enable pattern detection to query this field automatically

**Incident Management Integration:**
- When 🔴 flag detected, auto-create incident in Jira: "POD-5 Query Failures"
- Link all related support tickets to incident
- Track incident resolution across all affected customers

---

## Example: Real Workflow Session

**User message:**
> "Check ticket #408456 for me. Customer says they can't export data."

**Bot/Agent:**
1. ✅ Extract pod: Ticket metadata shows "POD-5"
2. ✅ Search: Found 9 tickets mentioning POD-5 in 90 days
3. ✅ Categorize: 4 Query/Export failures, 3 Performance, 2 Write
4. ✅ Assess: 4 Query failures in 28 days = 🔴 Systemic
5. ✅ Output pattern brief (see Scenario A above)
6. ✅ Recommend escalation to Engineering

**User's next decision:**
- Escalate ticket to Engineering
- Open Jira incident
- Notify customer
- Do NOT investigate customer config

**Outcome:**
- Engineering fixes the database query timeout on POD-5
- Fix rolls out, affecting all 9 customers simultaneously
- Support tickets auto-close
- Total time: 2 minutes for pattern detection + 10 minutes escalation vs. 2 hours investigating the wrong thing

---

## References

**Related documents:**
- Incident Escalation Policy (when to file a Jira incident)
- Infrastructure Node Mapping (which customers are on which pods)
- Ticket Categorization Guide (for consistent issue type classification)
- Root Cause Analysis Workflow (what comes after pattern detection)

---

## Document Version

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-06-05 | Initial workflow |

---

*Last Updated: June 5, 2026*
