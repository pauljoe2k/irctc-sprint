# IRCTC Civic-Tech Audit Sprint

> A structured product audit of the Indian Railway Catering and Tourism Corporation (IRCTC) platform — identifying, documenting, and proposing solutions to critical UX and system-level failures affecting millions of Indian rail passengers.

---

## Repository Structure

```
irctc-sprint/
├── README.md
├── part-a/
│   └── PROBLEMS.md          # 6 documented platform failures (3 given + 3 self-discovered)
├── part-b/
│   ├── SPECS.md             # Feature specification for selected problem
│   ├── AI-FEATURE.md        # AI-powered feature proposal
│   └── MATRIX.md            # Prioritisation matrix
└── assets/
    └── screenshots/         # Evidence screenshots from platform investigation
```

---

## Audit Scope

| Area | Covered |
|------|---------|
| Train search & availability | ✅ |
| Tatkal booking flow (10 AM window) | ✅ |
| Seat selection & berth preference | ✅ |
| Cancellation & TDR filing | ✅ |
| Waitlist label interpretation | ✅ |
| Divyang / accessibility flows | ✅ |
| Mobile browser experience | ✅ |
| Refund transparency | ✅ |
| PNR discoverability | ✅ |

---

## Platform Under Audit

- **URL:** [irctc.co.in](https://www.irctc.co.in)
- **Operator:** Indian Railway Catering and Tourism Corporation Ltd.
- **Daily active users:** ~1.2 million bookings/day (peak festival season)
- **Scale:** Handles 8+ billion passenger journeys annually across Indian Railways

---

## Part A — Problem Discovery

See [`part-a/PROBLEMS.md`](./part-a/PROBLEMS.md) for the full documented audit.

**Summary:** 6 high-severity platform failures identified across booking, navigation, accessibility, and information architecture.

---

## Audit Date

April 21, 2026 | Devices: Desktop (Chrome 124) + Mobile (Android, Chrome Mobile)