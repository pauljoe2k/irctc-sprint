# Impact vs Effort Matrix

## The Matrix

| | Low Effort | High Effort |
|---|---|---|
| **High Impact** | **Quick Wins**<br>P3: Form Persistence (Berth Reset)<br>P2: Persistent Search Filters<br>P5: Refund Calculation Preview | **Major Projects**<br>P1: Tatkal Virtual Waiting Room<br>P4: AI Waitlist Prediction |
| **Low Impact** | **Fill-ins**<br>P6: Accessible TDR Action Card | **Thankless Tasks**<br>*(None identified in this sprint)* |

---

## How I Scored Each Dimension

### Impact Scoring (1–5)
Based on:
* Volume of users actively blocked from completing a transaction.
* Financial severity (money lost or incorrectly deducted).
* Systemic platform health (reducing 502s, database loads).

### Effort Scoring (1–5)
Based on:
* Requirement for new infrastructure (Redis, ML pipelines).
* Cross-service integration complexity (Billing + Booking).
* Degree of risk in mutating core legacy APIs vs isolated frontend changes.

---

## Placement Justifications

### P3: Seat Selection / Berth Preference Resets — [High Impact, Low Effort]
1. **Impact (4/5):** Silently overwriting medical/age-related berth preferences ruins the core utility of the booking for a highly vulnerable demographic.
2. **Effort (1/5):** Requires strictly frontend changes (`sessionStorage` persistence) with zero modifications to the backend databases or APIs.
3. **Priority:** Placed in Quick Wins. It is the easiest problem to solve with the highest immediate return on user trust.

### P2: Persistent Search Filters — [High Impact, Low Effort]
1. **Impact (4/5):** Affects hundreds of thousands of daily searchers who currently abandon the funnel due to overwhelming, unfiltered data.
2. **Effort (2/5):** Primarily frontend routing and UI state management, though it requires wiring the frontend securely to existing search API parameters.
3. **Priority:** Placed in Quick Wins. It drastically improves the core discovery flow without requiring new infrastructure.

### P5: Pre-Cancellation Refund Calculation Preview — [High Impact, Low Effort]
1. **Impact (4/5):** Solves severe financial transparency issues and eliminates the primary driver of zero-refund complaints on Tatkal tickets.
2. **Effort (2/5):** *Updated post-peer review:* While building the 60-second lock requires minor Redis/caching work, the calculation logic already exists in the backend; we are simply exposing a dry-run endpoint.
3. **Priority:** Placed in Quick Wins. Financial clarity is paramount, and exposing existing logic is relatively inexpensive.

### P1: Tatkal Virtual Waiting Room — [High Impact, High Effort]
1. **Impact (5/5):** Solves the single largest PR nightmare and systemic failure of the IRCTC platform, restoring functionality during peak hours.
2. **Effort (5/5):** *Updated post-peer review:* Requires massive architectural shifts at the edge (CDN/Load Balancers), implementation of SSE for 5 million concurrent connections, and secure token lifecycle management.
3. **Priority:** Placed in Major Projects. It is essential for platform survival but will require a dedicated multi-quarter engineering pod.

### P4: AI-Powered Waitlist Probability Indicator — [High Impact, High Effort]
1. **Impact (5/5):** Fundamentally changes how Indian travelers make decisions, proactively reducing system load by encouraging early cancellations of doomed tickets.
2. **Effort (4/5):** Requires historical data pipelines, model training, MLops deployment, and real-time inference integration with the existing booking PNR flow.
3. **Priority:** Placed in Major Projects. The ROI is massive, but it requires specialized ML talent and heavy data engineering.

### P6: Accessible TDR Filing — [Low Impact, Low Effort]
1. **Impact (3/5):** While critical for affected users, the total volume of TDR-eligible passengers is a small fraction compared to daily searchers or Tatkal bookers.
2. **Effort (2/5):** Requires minor frontend CSS/layout fixes on mobile and a basic integration with the existing NTES delay API to trigger the contextual banner.
3. **Priority:** Placed in Fill-ins. It is a vital accessibility fix but should be slotted in alongside larger initiatives.

---

## Recommended Sprint Order

1. **P3 (Berth Resets):** Can be shipped in 3 days by a single frontend engineer. Fixes an embarrassing data-loss bug immediately.
2. **P2 (Search Filters):** Fixes the top-of-funnel discovery experience, clearing the path to conversion.
3. **P5 (Refund Preview):** Plugs the biggest financial transparency hole before tackling heavier infrastructure.
4. **P6 (TDR Access):** Quick mobile layout win that can be batched with the P5 release.
5. **P4 (AI Waitlist):** Start data collection and model training immediately; ship prediction UI in Phase 2.
6. **P1 (Tatkal Queue):** This is a massive infrastructure overhaul. Begin edge architecture design immediately, but full rollout will be the final milestone.
