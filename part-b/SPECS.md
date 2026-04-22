# Feature Specifications

## Feature Spec 1: Tatkal Virtual Waiting Room

### Problem Statement
During the daily 10:00 AM and 11:00 AM Tatkal windows, millions of concurrent users overwhelm the IRCTC infrastructure, leading to 502/503 errors, session invalidations, and silent timeouts. This primarily affects time-sensitive travelers who lose critical booking opportunities and suffer financial anxiety due to money deductions without ticket confirmation. The sheer lack of request orchestration results in a complete platform denial of service for genuine users.

---

### Current State (from Part A)
Failure occurs at the Step 6 → 7 transition. When the user submits the passenger form right at 10:00 AM, the server request queue saturates. The `POST /book/initiate` request either hits a load balancer timeout (502 Bad Gateway) or an application-level rejection (503). Existing sessions are dropped, forcing re-authentication, or worse, users reach the payment gateway, pay, and receive no ticket.

---

### Proposed Solution
We will implement a Virtual Waiting Room (VWR) that transparently queues users before they hit the booking application server. When a user clicks "Book Now" during peak load, they will see a clear, branded waiting screen displaying their exact position in the queue and an estimated wait time (ETA). The system orchestrates traffic into the core booking engine at a manageable rate. If their turn arrives, they automatically proceed to payment; if inventory exhausts while they wait, they are gracefully informed before paying.

---

### Proposed User Flow — Step by Step
1. **Initiation:** User clicks "Continue" on the passenger form during peak Tatkal hours.
2. **Queue Assignment:** The load balancer intercepts the request, assigns a cryptographic queue token, and redirects the user to the Virtual Waiting Room.
3. **VWR UI:** User sees the "Tatkal Virtual Waiting Room" screen.
4. **Status Update:** Screen displays "You are in line. Current position: 4,512. Estimated wait: 3 minutes." 
5. **Auto-Polling:** The client periodically polls the edge server (via SSE/Smart Polling) to update position and ETA.
6. **Graceful Reconnect:** If the user loses internet connection, their token guarantees their spot for a 2-minute grace period upon reconnection.
7. **Entry:** Once the queue reaches the user's position, the token is validated, and the user is automatically redirected to the payment gateway.
8. **Sold-Out Fallback:** If seats hit zero while the user is queued, the queue terminates early with a "Tatkal Quota Exhausted" message, preventing unnecessary wait and failed payments.

---

### Technical Implementation Plan

#### System components affected:
* Load Balancers / Edge Gateway (CDN level)
* Session Management Service
* Booking Engine (to report real-time inventory to the queue)

#### New data requirements:
* Redis-backed sorted sets for queue management
* Ephemeral queue tokens linked to user session IDs
* Live inventory counters synced to the queue edge

#### API changes:
* `POST /queue/join` - Assigns token and position
* `GET /queue/status` - Returns current position, ETA, and inventory status
* Modified `POST /book/initiate` to require and validate a cryptographic queue token

#### Frontend changes:
* New `VirtualQueue` React component
* Polling logic with exponential backoff
* Token storage in `sessionStorage` for reconnects
* Prevention of browser "Back" button navigation out of the queue without warning

#### Third-party services (if any):
* Cloudflare Waiting Room or AWS SQS + ElastiCache (Redis)

---

### Success Metrics
* Reduce 502/503 error rates during Tatkal windows by 95%.
* Reduce payment gateway transaction failures from X% → <1%.
* Increase successful booking throughput per minute (optimized server utilization).
* Decrease customer support tickets related to "money deducted, no ticket" by 90%.

---

### Edge Cases and Constraints
* **Network Drop:** Handled by the 2-minute token grace period. (Added post-peer review).
* **Inventory Depletion:** The queue must listen to inventory streams and halt immediately when seats hit 0.
* **Concurrency:** WebSocket connections for 5 million users will crash the edge. Using Server-Sent Events (SSE) falling back to intelligent polling is required. (Updated post-peer review).
* **Bypass Attempts:** Users trying to hit the booking API directly must be rejected if they lack a cryptographically signed queue token.

---

### Wireframe
![Tatkal Queue Screen](../assets/wireframes/tatkal-virtual-queue-screen.png)

1. **Screen name:** Tatkal Virtual Queue
2. **Full layout breakdown:** Minimalist centered card layout on a branded IRCTC background. No complex navigation to prevent accidental clicks.
3. **Every visible component label:**
   - Headline: "You are in the Tatkal Queue"
   - Large dynamic number: "Your position: 4,512"
   - Subtext: "Estimated wait time: ~3 minutes"
   - Progress bar: Indeterminate animated train graphic.
   - Warning text: "Do not refresh or press back. If disconnected, you have 2 minutes to return."
   - Cancel Button: "Leave Queue"
4. **Key interaction annotations:** "Leave Queue" triggers a confirmation modal. No other interactions allowed.
5. **Loading state:** Shimmer effect on position numbers during the first 2 seconds of connection.
6. **Empty state:** N/A (Always data-driven).
7. **Error state:** "Connection lost. Reconnecting... (Your spot is saved)."
8. **What changed:** Replaced the spinning loader and sudden 502 white-screen-of-death with a deterministic, informative waiting state.

---

## Feature Spec 2: Persistent & Deterministic Search Filters

### Problem Statement
Users attempting to filter train search results by time, class, or quota find that the filters are merely cosmetic and do not reliably re-render the data. Furthermore, pagination completely destroys the filter state. This forces users into manual visual scanning, causing immense frustration, high cognitive load, and frequent abandonment for users with specific constraints (e.g., evening AC trains only).

---

### Current State (from Part A)
Failure occurs at Steps 4–5 and Step 7. When a user clicks "Evening Departure", the UI highlights the button, but morning trains remain visible. If the user navigates to Page 2, all active filters are wiped out, and the full unsorted list returns. The state is purely client-side and disconnected from the data layer and routing.

---

### Proposed Solution
Implement deterministic, stateful filtering where every filter click instantly updates the results via an asynchronous request, and the URL parameters are immediately updated to reflect the filter state. Users will see a skeleton loader while results update, ensuring clear feedback. Pagination will read from the URL state, guaranteeing that filters survive across pages, browser refreshes, and shared links.

---

### Proposed User Flow — Step by Step
1. **Execution:** User clicks "Evening (18:00–23:59)" on the search page.
2. **State Sync:** The UI instantly pushes `?departure=1800-2359` to the browser URL history.
3. **Feedback:** The main train list dims and shows a skeleton loading state.
4. **Data Fetch:** The frontend fires an AJAX request with the exact filter parameters.
5. **Render:** Results re-render, showing strictly evening trains. A clear "1 Filter Applied" badge appears.
6. **Pagination:** User clicks "Page 2".
7. **Persistence:** The system routes to `?departure=1800-2359&page=2`. The filters remain active, the UI shows them as pressed, and only evening trains appear on the second page.

---

### Technical Implementation Plan

#### System components affected:
* Search Frontend (React/Angular router)
* Search API (Controller layer)

#### New data requirements:
* No new DB fields. Requires serialization of all filter combinations into query parameters.

#### API changes:
* Modify `GET /trains/search` to strict-match array parameters for `departureTime`, `class`, and `quota`.

#### Frontend changes:
* Implementation of a URL-driven state management pattern (e.g., React Router `useSearchParams`).
* Debouncing filter clicks (300ms) to prevent API spam.
* Skeleton loaders for the results component.
* "Clear All Filters" global action button.

#### Third-party services (if any):
* None.

---

### Success Metrics
* Increase successful filter application rate to 100%.
* Reduce average time spent on the search results page (indicating faster decision-making).
* Eliminate pagination-related filter resets (measured via frontend error/event tracking).

---

### Edge Cases and Constraints
* **No Results:** If a filter combination yields zero trains, display a friendly "No trains match these exact filters" state with a one-click "Clear Filters" button.
* **Rapid Clicking:** Users clicking multiple filters rapidly should not trigger race conditions. Debouncing and API request cancellation (AbortController) must be implemented.

---

### Wireframe
![Persistent Filter Bar](../assets/wireframes/persistent-filter-bar.png)

1. **Screen name:** Search Results with Stateful Filters
2. **Full layout breakdown:** Left sidebar for filter facets, main right column for results. Sticky "Active Filters" row above results.
3. **Every visible component label:**
   - Active Filters Row: Chips like "[x] Evening", "[x] 3AC", "Clear All".
   - Left Sidebar: Collapsible accordions for Journey Class, Departure Time, Quota.
4. **Key interaction annotations:** Clicking a sidebar filter instantly adds a chip to the top row and appends to URL.
5. **Loading state:** 3–4 skeleton train cards replacing the actual data during API fetch.
6. **Empty state:** "No trains found for your filters. [Clear All Filters]"
7. **Error state:** "Failed to apply filters. Please try again." with a retry button.
8. **What changed:** Introduction of the active filters chip row, skeleton loaders for feedback, and URL-parameter sync.

---

## Feature Spec 3: Client-Side Form Persistence & Review

### Problem Statement
Passenger berth preferences (e.g., "Lower Berth" for elderly users) silently reset to "No Preference" if the user briefly navigates away or switches between passengers. Because the confirmation screen hides this data, users only discover the error after payment. This causes severe hardship for vulnerable demographics who medically require specific berths.

---

### Current State (from Part A)
Failure occurs at Step 4 → 5. The passenger form does not persist state. Clicking the browser "Back" button and returning re-initializes the form with default values. The booking review screen omits the berth preference entirely, preventing the user from catching the silent reset.

---

### Proposed Solution
Implement local storage persistence for the passenger form so that data survives page reloads, session warnings, and accidental back-button navigation. Additionally, explicitly display the requested berth preference on the final Review & Payment screen, forcing visual confirmation before the user commits financially.

---

### Proposed User Flow — Step by Step
1. **Input:** User fills out Passenger 1, selecting "Lower Berth".
2. **Auto-Save:** The frontend instantly saves `{p1: {berth: 'LB'}}` to `sessionStorage`.
3. **Interruption:** User accidentally clicks the browser Back button.
4. **Return:** User navigates forward to the passenger form.
5. **Recovery:** On component mount, the frontend checks `sessionStorage`, finds the data, and pre-fills the entire form, strictly preserving "Lower Berth".
6. **Review:** User proceeds to the Review screen.
7. **Explicit Confirmation:** The summary card clearly states: "Passenger 1: Ram Kumar | 65 | M | Preference: Lower Berth". User pays with confidence.

---

### Technical Implementation Plan

#### System components affected:
* Passenger Entry Frontend
* Booking Review Frontend

#### New data requirements:
* Use of browser `sessionStorage` (cleared upon successful booking or manual session termination).

#### API changes:
* No backend API changes required for data entry.
* Minor change to `GET /booking/summary` (if review screen is server-rendered) to ensure preference is returned in the payload.

#### Frontend changes:
* Hook form state to `sessionStorage`.
* Add a "Clear Form" button in case users actually want to start over.
* Update the Review UI component to map and render the `berthPreference` variable.

#### Third-party services (if any):
* None.

---

### Success Metrics
* Reduce post-booking support tickets regarding "wrong berth allocated despite preference" by 80%.
* 100% retention of form data during accidental intra-session navigations.
* Decrease form completion time for returning users.

---

### Edge Cases and Constraints
* **Privacy:** Do not use `localStorage` as it persists across tabs/sessions on shared computers. `sessionStorage` is strictly tab-scoped and safer for PII.
* **Quota Clashes:** If the user changes the train or class while navigating back, the saved preference might be invalid (e.g., selecting AC First Class which doesn't have side berths). The form must validate `sessionStorage` data against the current train class schema on mount.

---

### Wireframe
![Passenger Form Persistent State](../assets/wireframes/passenger-form-persistent-state.png)

1. **Screen name:** Passenger Details & Review Screen
2. **Full layout breakdown:** Standard passenger entry form. At the bottom, a prominent "Review Journey" modal/section.
3. **Every visible component label:**
   - Dropdown: "Berth Preference (Saved)"
   - Review Card: Name, Age, Gender, **Requested Berth: Lower**.
   - Action: "Clear saved details".
4. **Key interaction annotations:** A small toast "Draft saved" appears momentarily when inputs change.
5. **Loading state:** N/A (synchronous).
6. **Empty state:** Standard blank form.
7. **Error state:** "Preference invalid for this class, reset to No Preference." (Highlighted in yellow).
8. **What changed:** Addition of the toast notification, explicit rendering of the preference in the review card, and state recovery logic.

---

## Feature Spec 4: AI-Powered Waitlist Probability Indicator

### Problem Statement
Users are presented with cryptic waitlist codes (e.g., `PQWL 8/PQWL 6`) with zero contextual translation, leading to poor decision-making. Passengers hold onto doomed tickets (like PQWL) until chart preparation, resulting in automatic cancellations, stranded travelers, and massive anxiety. The platform provides no decision support.

---

### Current State (from Part A)
Failure occurs at Steps 2–3. The system directly surfaces raw database strings (`WL 45/GNWL 18`) to the user on the booking confirmation and My Bookings pages. There is no guidance on whether to cancel, wait, or seek alternatives.

---

### Proposed Solution
Translate raw waitlist codes into human-readable, AI-backed probability scores. Next to every waitlisted ticket, display a clean indicator: "High Chance (85%)", "Medium Chance (40%)", or "Low Chance (5%)". Provide an informational tooltip explaining the specific waitlist type (e.g., "Pooled Quota Waitlist: Usually confirms only if a passenger on your specific route cancels").

---

### Proposed User Flow — Step by Step
1. **View Ticket:** User opens the "My Bookings" page.
2. **Status Display:** The ticket shows `PQWL 8`.
3. **Probability Badge:** Right next to it, a red badge states "Low Confirmation Chance (12%)".
4. **Contextual Action:** User clicks the badge. A modal opens.
5. **AI Explanation:** "Based on historical data for this train, PQWL 8 has a 12% chance of confirming. We recommend booking an alternative or utilizing the Vikalp scheme."
6. **Decision:** User clicks "View Alternative Trains" directly from the modal.
7. **Resolution:** User secures a confirmed ticket on a different train, avoiding last-minute stranding.

---

### Technical Implementation Plan

#### System components affected:
* Booking Engine (read-only)
* New AI Inference Microservice
* My Bookings Frontend

#### New data requirements:
* Historical PNR confirmation data (train number, date, WL type, initial position, final status).
* API gateway integration for the prediction model.

#### API changes:
* New endpoint: `GET /pnr/predict?pnr=1234567890` returning `{probability: 0.12, label: 'LOW', recommendation: '...'}

#### Frontend changes:
* UI badges (Green/Yellow/Red) mapping to probability thresholds.
* Tooltip/Modal component for explanations.

#### Third-party services (if any):
* Internal ML model (e.g., LightGBM) or Vertex AI endpoint.

---

### Success Metrics
* Increase early voluntary cancellations of low-probability tickets (freeing up system resources).
* Increase adoption of the "Vikalp" (alternative train) scheme by 20%.
* Reduce support queries asking "Will my ticket confirm?".

---

### Edge Cases and Constraints
* **False Positives/Negatives:** The model will occasionally be wrong. The UI must explicitly state "This is a prediction based on historical data, not a guarantee."
* **VIP Quota Release:** Last-minute HO (Headquarters) quota releases can drastically change probabilities 4 hours before departure. Predictions must degrade to "Chart Preparation Pending" within 12 hours of departure.

---

### Wireframe
![Waitlist Probability Card](../assets/wireframes/waitlist-probability-card.png)

1. **Screen name:** My Bookings - Waitlist Detail
2. **Full layout breakdown:** Existing ticket card with a new right-aligned "Prediction Panel".
3. **Every visible component label:**
   - Text: `PQWL 8/PQWL 5`
   - Badge: 🔴 **Low Chance (12%)**
   - Link: "Why is this low?"
   - Button: "Find Alternative Trains"
4. **Key interaction annotations:** Clicking "Why is this low?" opens a bottom-sheet on mobile explaining PQWL mechanics.
5. **Loading state:** Shimmer block over the probability badge.
6. **Empty state:** N/A.
7. **Error state:** Badge reads "Prediction Unavailable" in neutral grey.
8. **What changed:** Transformation of a cryptic string into actionable, color-coded, data-backed guidance.

---

## Feature Spec 5: Pre-Cancellation Refund Calculation Preview

### Problem Statement
Users cancelling tickets are not shown the exact refund amount before the irreversible cancellation API executes. This violates basic financial transparency. Users cancelling Tatkal tickets are completely blind to the zero-refund rule until after the transaction is executed, leading to severe financial shock and loss of trust.

---

### Current State (from Part A)
Failure occurs at Step 3 → 5. The user clicks "Cancel Ticket" on a screen that shows no financial breakdown. The button immediately processes the cancellation. The `₹0` refund outcome (in the case of Tatkal) is only revealed on the final receipt.

---

### Proposed Solution
Insert a mandatory "Refund Quote" step before executing a cancellation. Clicking "Cancel" on a booking will calculate the precise penalty and refund based on the current timestamp. The user will be presented with an itemised breakdown (Total Fare - Cancellation Charges = Estimated Refund). For Tatkal, a high-contrast warning will state "Zero Refund Applicable". The user must explicitly click "Confirm & Cancel" to proceed.

---

### Proposed User Flow — Step by Step
1. **Initiation:** User clicks "Cancel Ticket" from My Bookings.
2. **Async Calculation:** Frontend shows a loading spinner: "Calculating Refund..."
3. **Quote Generation:** The backend calculates the refund based on train departure time (e.g., >48 hrs, 12-48 hrs).
4. **Preview UI:** User sees the Refund Preview screen. "Total Paid: ₹2400. IRCTC Penalty: ₹500. Expected Refund: ₹1900."
5. **Timer Lock:** A 60-second countdown begins, locking in that exact refund quote to prevent edge-case time-boundary disputes.
6. **Confirmation:** User clicks a red "Confirm Cancellation" button.
7. **Execution:** Cancellation is processed, and the final receipt matches the previewed quote exactly.

---

### Technical Implementation Plan

#### System components affected:
* Cancellation Engine
* Billing / Refund Service

#### New data requirements:
* Ephemeral "Quote IDs" valid for 60 seconds.

#### API changes:
* New endpoint: `POST /cancel/quote` (returns calculation without mutating DB).
* Modified endpoint: `POST /cancel/execute` (accepts Quote ID to ensure idempotency and price locking).

#### Frontend changes:
* New Refund Preview modal/page.
* Timer component for the 60-second quote validity.
* High-contrast warning banners for zero-refund scenarios.

#### Third-party services (if any):
* None.

---

### Success Metrics
* Eliminate complaints and chargebacks related to "unexpected zero refund" on Tatkal tickets.
* Decrease abandonment rate during the cancellation flow (users trust the transparency).
* Increase clarity measured via post-cancellation NPS.

---

### Edge Cases and Constraints
* **Time Boundary Crossing:** If a user requests a quote at 47h:59m before departure, the backend must lock that refund tier for the 60-second UI timer. If the timer expires, the user must fetch a new quote, which may fall into the harsher penalty tier. (Implemented post-peer review).
* **System Load:** Generating a quote requires reading chart status and DB rules. Ensure `POST /cancel/quote` is heavily cached or optimized to prevent DDOSing the billing engine.

---

### Wireframe
![Pre-Cancellation Refund Preview](../assets/wireframes/pre-cancellation-refund-preview.png)

1. **Screen name:** Refund Preview Modal
2. **Full layout breakdown:** A receipt-style breakdown centered on the screen, darkening the background.
3. **Every visible component label:**
   - Title: "Cancellation Summary"
   - Line Items: Base Fare, GST, Cancellation Penalty (-).
   - Large Text: **Estimated Refund: ₹1,900**
   - Timer text: "Quote valid for 00:45"
   - Warning (if Tatkal): "⚠️ Tatkal Tickets are non-refundable. You will receive ₹0."
   - Action: "Confirm Cancellation" (Red), "Keep Ticket" (Grey).
4. **Key interaction annotations:** "Confirm" triggers the irreversible API.
5. **Loading state:** "Calculating current refund rules..." with a spinner.
6. **Empty state:** N/A.
7. **Error state:** "Unable to calculate refund right now. Please try again."
8. **What changed:** Decoupled the intent to cancel from the execution, adding a mandatory financial preview step.

---

## Feature Spec 6: Accessible and Context-Aware TDR Filing

### Problem Statement
Ticket Deposit Receipt (TDR) filing is the only way to claim refunds for railway failures (e.g., 3+ hour delays), but it is buried 3 levels deep in the UI and cut off by mobile viewports. Furthermore, users don't know the strict deadlines for filing. This causes hundreds of thousands of users to miss their legal refund window due to poor UX.

---

### Current State (from Part A)
Failure occurs at Steps 3–4 and Step 6. TDR is hidden inside past booking details. On standard mobile screens, the action link requires an undiscoverable horizontal scroll. There is no context provided regarding time limits.

---

### Proposed Solution
Elevate the "File TDR" action. If a train's live running status shows a delay of >3 hours, or if the ticket falls into a valid post-journey TDR window, automatically surface a prominent "Claim Refund (TDR)" button directly on the primary "My Bookings" list view. Include a countdown timer indicating exactly when the filing window closes. Fix the mobile layout by using a vertical action stack instead of horizontal scrolling.

---

### Proposed User Flow — Step by Step
1. **Trigger Event:** User's train is delayed by 4 hours.
2. **Discovery:** User opens the app and navigates to My Bookings.
3. **Contextual Action:** Directly on the booking card, a yellow banner appears: "Train delayed >3 hrs. You are eligible for a full refund."
4. **Actionability:** A button says "File TDR Now. Window closes in 14 hours."
5. **Mobile Friendly:** The button is full-width, perfectly visible on all mobile viewports without scrolling.
6. **Filing:** User taps the button, selects the reason ("Train Late > 3 Hours") from a pre-filled dropdown, and submits.
7. **Confirmation:** System confirms TDR is filed and provides a tracking ID.

---

### Technical Implementation Plan

#### System components affected:
* TDR Service
* NTES (National Train Enquiry System) API Integration
* My Bookings Frontend

#### New data requirements:
* Caching NTES delay data against active PNRs.

#### API changes:
* Modify `GET /bookings/my` to append `eligibleForTDR: boolean`, `tdrReasonCode: string`, and `tdrDeadline: timestamp`.

#### Frontend changes:
* Refactor mobile CSS: change `flex-row overflow-x-auto` to a CSS Grid or `flex-col` stack for action buttons.
* Inject conditional TDR banners on booking cards.

#### Third-party services (if any):
* NTES / CRIS Live Train Status API.

---

### Success Metrics
* Increase TDR filing success rate within the legal deadline by 40%.
* Reduce customer support calls regarding "How do I claim a refund for a delayed train?".
* Eliminate mobile UI truncation issues (measured via click tracking on the button).

---

### Edge Cases and Constraints
* **NTES Latency:** Train delay data from NTES can sometimes be inaccurate. If NTES says 2.5 hours, but the user claims 3 hours, the TDR UI should still allow manual filing (with standard manual review by railway staff).
* **Multiple Passengers:** TDRs can be filed for partial passengers. The UI must cleanly allow selecting which passengers did not travel.

---

### Wireframe
![Accessible TDR Action Card](../assets/wireframes/accessible-tdr-action-card.png)

1. **Screen name:** My Bookings List - Mobile View
2. **Full layout breakdown:** Vertical list of ticket cards. Active cards dynamically expand to show alerts.
3. **Every visible component label:**
   - Card Header: Train 12951 • New Delhi to Mumbai
   - Alert Banner (Yellow background): "⚠️ Train delayed by 4 hours."
   - Deadline Text: "TDR filing window closes at 23:59 tonight."
   - Full-width Button: "File TDR (Claim Refund)"
4. **Key interaction annotations:** Tap button -> Opens full-screen TDR form.
5. **Loading state:** Standard card loading.
6. **Empty state:** N/A.
7. **Error state:** N/A.
8. **What changed:** Moved TDR out of the hidden detail view onto the primary list. Replaced horizontal scroll with a vertical, full-width mobile button layout. Added predictive context (delay integration).

---
