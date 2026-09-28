# RailSync-AI: System Architecture & Technical Blueprint
**Smart India Hackathon 2026 | Problem Statement #26027**  
*AI-Powered Automatic Block Planning to Maximize Asset Availability for Train Operations on Indian Railways*

---

## 1. System High-Level Architecture

```
[ TMS (ircep.gov.in) ]   [ TDMS (railsaver) ]   [ SMMS (signalling) ]   [ COA / NTES (cris.org.in) ]
          │                       │                       │                          │
          └───────────────┬───────┴───────────────┬───────┴──────────────────────────┘
                          ▼                       ▼
            ┌───────────────────────────────────────────┐
            │        CENTRAL DATA INGESTION &           │  <-- Automated ETL / Webhook Adapters
            │      STATUTORY RULE VALIDATION HUB        │      (Restricted Intranet Bus)
            └─────────────────────┬─────────────────────┘
                                  ▼
            ┌───────────────────────────────────────────┐
            │               AI CORE ENGINES             │
            │                                           │
            │   [ AI Model 2: Risk & Priority Engine ]  │
            │   • IRPWM 2020 / ACTM Vol II / IRSEM      │
            │   • IMR Rail Fractures, OHE Wear, Dips    │
            │   • 14-Day Mandatory Safety Deadline      │
            │                         │                 │
            │                         ▼                 │
            │   [ AI Model 1: Multi-Dept Scheduler ]    │
            │   • Live Train Movement & Delay Sync      │
            │   • Multi-Dept Bundling (P-Way+TRD+S&T)   │
            │   • OR-Tools MILP Corridor Optimization   │
            └─────────────────────┬─────────────────────┘
                                  ▼
            ┌───────────────────────────────────────────┐
            │    DECISION SUPPORT & WORKFLOW PORTAL     │
            │                                           │
            │  • Human-in-the-Loop Protocol (AI Advises)│
            │  • Post-Based RBAC Matching IR Powers     │
            │  • Dynamic Delay Re-scheduling Modal      │
            │  • Station Master & Controller Memo Sign  │
            └───────────────────────────────────────────┘
```

---

## 2. Deep Dive: Dual AI Model Logic

### AI Model 2: Priority & Regulatory Compliance Engine (Execution Step 1)
Evaluates raw asset inspection feeds from existing Indian Railway portals against statutory maintenance codes.

* **Data Inputs:**
  * **TMS (`ircep.gov.in/TMS/`):** IMR (Immediate Removal) rail fractures, weld defects, gauge spread, Track Geometry Car (OMS/TGC) acceleration spikes, speed restriction logs.
  * **TDMS (`tdms.railsaver.gov.in/`):** Contact wire thickness wear, OHE insulator flashover logs, bracket fatigue, overdue monthly inspection cycles.
  * **SMMS (`smms.indianrailways.gov.in`):** Point machine friction current logs, track circuit fail-safe status, axle counter chattering, relay room temperature/power dips.
* **Rules & Time-Bounds Embedded:**
  * *Hard Safety Constraints:* IMR defects require speed restriction (30 km/h) or block window within 24 hours. Track geometry inspection mandated within **14 days** (IRPWM 2020 Para 4.2). Signal point failure = Zero tolerance.
* **Algorithm Choice:**
  * **XGBoost / CatBoost Gradient Trees + Drools Expert Rules Engine.**
  * *Output:* **Criticality Score (1 to 100)** + **Hard Deadline Window**.

### AI Model 1: Multi-Agent Coordinated Block Scheduler (Execution Step 2)
Takes the prioritized tasks from Model 2 and interleaves them between live train schedules.

* **Multi-Department Corridor Bundling:**
  * If P-Way requests a block on *Km 142 to 146* for 2 hours, Model 1 queries TDMS & SMMS: *"Is OHE wire inspection or signal point testing due on Km 142–146?"*
  * Bundles all 3 departments into **a single combined window**, eliminating 2 extra train halts.
* **Synthetic Window Extraction:**
  * Analyzes COA master timetables and goods train pathing. Holds freight trains on loop lines to extract 90–120 minute maintenance corridors with minimal passenger train impact.
* **Algorithm Choice:**
  * **Mixed-Integer Linear Programming (MILP via Google OR-Tools Solver) + Reinforcement Learning (PPO Agent for Dynamic Slot Allocation).**

---

## 3. Dynamic Real-Time Re-Scheduling Workflow

When a train is delayed unexpectedly on the line:

```
[Live Train Delay Detected (COA / NTES Feed)]
                       │
                       ▼
    [AI Conflict Sub-Routine Triggered]
    "Train 12952 intersects Block at Km 142 @ 11:35 AM"
                       │
                       ▼
    [AI Calculates 2 Safe Options within 15 Seconds]
    • Option A: Shift block window to 12:15 PM – 02:15 PM (Zero train delay).
    • Option B: Reduce block to 45 mins (Execute Priority-1 Track work only).
                       │
                       ▼
    [Instant Advisory Alert to Section Controller & Station Master]
    "Conflict Alert! Approve Option A, Option B, or Manual Override?"
                       │
                       ▼
    [Section Controller Clicks 'Approve Option A']
    -> Cryptographically Signed Digital Memo Broadcast to SM & Field Apps
```

> **Safety Protocol:** AI NEVER auto-finalizes. Final authority rests strictly with the human Section Controller and Station Master.

---

## 4. Role-Based Access Control (RBAC) & Designation Hierarchy

| Role / Post | Platform Access | Permissions & Powers |
| --- | --- | --- |
| **Sr. DEN / Sr. DEE / Sr. DSTE** (Divisional Officers) | Web Dashboard (`officer.html`) | Monthly/Weekly cyclic block plan approvals, inter-departmental conflict resolution, department priority overrides. |
| **Section Controller** (Control Office) | Web Dashboard (`dashboard.html`) | Real-time block grant recommendation approval/rejection, train delay balancing, 1-click BDMS sign-off. |
| **Station Master (SM)** | Station Terminal / Web (`field.html?role=sm`) | Local station disconnection memo verification, track clear acknowledgement. |
| **SSE (Section Senior Engineer)** | Mobile PWA & Web (`field.html?role=sse`) | Feed pending defects, log joint block requests, work completion status punch. |
| **Gang / Field Supervisor** | Mobile PWA (`field.html`) | Trackside check-in, block start/complete safety memo click. |

---

## 5. Security & RailNet Integration Strategy

1. **Air-Gapped / RailNet Compliance:** Deployment inside CRIS / RailNet intranet zones. Zero direct public database exposure.
2. **Reverse Proxy & mTLS:** Field mobile app calls pass through an API Gateway with mutually authenticated TLS (mTLS).
3. **Audit Trail & Digital Signatures:** Immutable logging (timestamp, Officer ID, geolocation, post code) for every block grant, cancellation, or override for complete transparency.

---

## 6. Implementation Tech Stack

* **Frontend (Web & Mobile):** React / Next.js (Dashboard with Gantt charts) + Progressive Web App (PWA) / Flutter (Trackside field app).
* **Backend:** FastAPI (Python) for microservices & fast AI model execution.
* **Optimization & AI Engines:**
  * Prioritization: Python (`scikit-learn`, `xgboost`, `Drools` / Python Rule Dispatchers).
  * Scheduling: `Google OR-Tools` (MILP & Constraint Programming).
* **Data Layer:** PostgreSQL (PostGIS enabled for track kilometer mapping) + Redis (Real-time conflict queues & live alerts).
