# AI Feature Specification: Predictive Waitlist Confirmation Engine

## Problem It Solves
**Problem 4: Waitlist Status Labels Are Cryptic and Provide Zero Decision Support.**
Currently, users are presented with raw database codes (e.g., `PQWL 8/PQWL 5`) with no context regarding the probability of that ticket actually confirming. This leads to users holding onto doomed tickets, resulting in automatic cancellations at chart preparation and last-minute travel stranding.

---

## Proposed Feature — User Perspective
When a user views a waitlisted ticket on the "My Bookings" page or at the time of booking, they will see a color-coded "Confirmation Probability" badge (High/Medium/Low) alongside a percentage (e.g., 85%). If the chance is "Low", the UI proactively surfaces a recommendation to book an alternative train or use the Vikalp scheme. The user no longer has to guess what `PQWL` or `GNWL` means; the platform gives them data-backed confidence.

---

## Model or API Choice
**LightGBM (Gradient Boosting Framework)** deployed via a scalable inference microservice (e.g., Vertex AI or AWS SageMaker).
* **Why LightGBM:** It is highly optimized for tabular data (which booking records are) and trains exceptionally fast on large datasets. It handles categorical features (Train Number, Quota Type, Source, Destination) natively without requiring massive one-hot encoding payloads.
* **Why not LLMs (GPT-4, etc.):** This is a structured numerical prediction problem, not a generative text problem. LLMs would be slow, overly expensive, prone to hallucination, and fundamentally the wrong mathematical tool for predicting binary classification outcomes (Confirmed vs. Not Confirmed) based on historical tabular data.

---

## Training or Input Data
**Data Needed (Historical):**
* Minimum 2 years of historical IRCTC PNR data.
* **Features:** Train Number, Date of Journey (mapped to seasonality/festivals), Day of Week, Source Station, Destination Station, Class of Travel (1A, 2A, 3A, SL), Waitlist Type (GNWL, PQWL, RLWL), Initial Waitlist Position.
* **Target Variable:** Final Status at Chart Preparation (1 = Confirmed/RAC, 0 = Waitlisted/Cancelled).

**Where it comes from:**
This data is already available in IRCTC's central CRIS databases. It requires extraction, anonymization (removing PII), and feature engineering.

**Data Needed (Live Inference):**
* The user's current PNR details and real-time waitlist position, passed to the model endpoint.

---

## How Output Is Shown to the User
The output is displayed directly on the booking card in the "My Bookings" page and on the final checkout screen before payment.
* **Visual:** A small badge next to the status.
  * 🟢 High Chance (>75%)
  * 🟡 Medium Chance (40% - 74%)
  * 🔴 Low Chance (<40%)
* **Interaction:** Clicking the badge opens a tooltip: "Based on historical trends for Train 12951, a PQWL 8 has a 12% chance of confirmation."
* **Reference:** Matches the `waitlist-probability-card.png` wireframe detailed in SPECS.md.

---

## Confidence Threshold and Fallback
* **High Confidence Constraint:** The model output should only be shown if the prediction confidence interval is narrow. If the model hasn't seen enough data for a specific new train route, it should return a "Prediction Unavailable" state rather than a wild guess.
* **Chart Prep Fallback:** Predictions become highly volatile right before chart preparation due to VIP/HO quota releases. Within 12 hours of departure, the UI should gracefully degrade the percentage to a generic "Chart Preparation Pending" message to avoid false confidence.
* **Service Failure:** If the AI microservice times out (>800ms) or returns a 500 error, the UI simply defaults to the current state (showing only the raw `WL 8` code) without breaking the page layout.

---

## Success Metrics
1. **Cancellation Rate of Low-Probability Tickets:** Increase by 30% (indicating users are making informed decisions to cancel early, freeing up DB resources).
2. **Vikalp Adoption:** 20% increase in users opting for alternative trains when shown a "Low Chance" badge.
3. **Customer Support Deflection:** 40% reduction in support queries asking "Will my ticket confirm?".

---

## Limitations and Risks
* **Government Constraints:** The Ministry of Railways may object to officially labelling a ticket as "Low Chance", as it might cause political or public relations backlash. The phrasing may need to be softened to "Historically Low Confirmation Rate".
* **False Confidence:** If the model predicts "90% chance" and the ticket fails to confirm, the user will blame IRCTC, potentially leading to consumer court cases. A strict disclaimer must be present: *"Prediction based on past trends, not a guarantee."*
* **Special Quota Volatility:** The model cannot predict emergency VIP travel or sudden military deployments that consume quotas and completely invalidate historical trends.
