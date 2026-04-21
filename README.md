# Clinia 🩺

> **AI-powered medical receptionist** — automating patient registration, appointment scheduling, and EMR record management through a real-time conversational voice agent.

---

## Overview

Clinia replaces the traditional front-desk receptionist with a fully automated, AI-driven voice agent named **Chelsea**. Patients call Houston Medical Center, speak naturally in English or Spanish, and Chelsea handles everything — verifying existing patients, registering new ones, checking appointment availability, and booking slots — all synced live into OpenEMR.

Built at the intersection of conversational AI, healthcare automation, and clinical systems integration, Clinia demonstrates how modern AI tooling can meaningfully reduce administrative burden without requiring clinics to overhaul their existing infrastructure.

**Status:** Working MVP — live against a real OpenEMR instance.

---

## The Problem

Healthcare front desks are overwhelmed. The average medical receptionist spends over 60% of their time on repetitive, low-value tasks: answering phones, collecting patient information, scheduling appointments, and manually entering data into EMR systems. This leads to:

- Long patient hold times and abandoned calls
- Data entry errors and inconsistent records
- Burnout among administrative staff
- Delayed care due to scheduling bottlenecks

Clinia addresses this directly with a voice-first, always-available AI agent that handles these workflows end-to-end.

---

## How It Works

```
Patient calls Houston Medical Center
              │
              ▼
   ┌──────────────────────┐
   │  Retell AI — Chelsea │  ← Conversational voice agent (GPT-5.1)
   │  Language Selection  │    Voice: Rita | en-US | es-ES transfer
   └──────────┬───────────┘
              │  Tool call webhook (per intent)
              ▼
   ┌──────────────────────┐
   │  Make.com Scenario   │  ← "Houston Medical Center Flow (Retell+OpenEMR)"
   │  Router + Logic      │    Validates, routes, transforms payloads
   └──────┬──────┬────────┘
          │      │
          ▼      ▼
   ┌──────────┐ ┌────────────────────┐
   │ OpenEMR  │ │  Webhook Response  │
   │ REST API │ │  back to Chelsea   │
   └──────────┘ └────────────────────┘
```

---

## Conversation Flow — Chelsea's 32-Node State Machine

The agent is built as a structured conversation flow with 32 nodes, covering two primary patient journeys:

### New Patient Path
| Step | Node | Action |
|---|---|---|
| 1 | Language Selection | Detects English or Spanish; transfers to Andrea (Spanish agent) if needed |
| 2 | Welcome + Intent Detection | Greets patient, identifies new vs. existing |
| 3 | Collect Full Name | First and last name |
| 4 | Collect Date of Birth | With clarification fallback node if DOB is unclear |
| 5 | Collect Phone Number | Used as unique patient identifier |
| 6 | Collect Reason for Visit | Maps to clinic's service categories |
| 7 | Check Insurance | Determines coverage; collects insurance info if applicable |
| 8 | **`Create_Patient` tool call** | Registers patient in OpenEMR via POST |
| 9 | Check Available Appointments | Queries OpenEMR for open slots |
| 10 | Confirm Appointment Details | Reads back date, time, doctor to patient |
| 11 | **`POST--appointments-create` tool call** | Books appointment in OpenEMR |
| 12 | Booking Confirmation + Closing | Confirms and ends call gracefully |

### Existing Patient Path
| Step | Node | Action |
|---|---|---|
| 1 | Language Selection | Same as above |
| 2 | Welcome + Intent Detection | Identifies as existing patient |
| 3 | Collect Phone Number | Lookup key |
| 4 | **`GET-Patient-Information` tool call** | Pulls existing record from OpenEMR |
| 5 | Confirm Identity | Verifies name + DOB against record |
| 6 | Reason for Visit | Collects visit reason |
| 7 | Check Slot Availability | `GET-available-appointment` + `check_specific_slot` |
| 8 | Book + Confirm | Same booking flow as new patient |

---

## Tech Stack

### 🎙️ Retell AI — Voice Agent (Chelsea / Rita)

| Setting | Value |
|---|---|
| Agent Name | Houston Medical Center Receptionist |
| Voice | `retell-Rita` |
| Version Title | `Rita-HMC-V0` |
| Language | `en-US` (Spanish transfer via agent swap node) |
| LLM Model | GPT-5.1 (cascading) |
| Post-call Analysis | GPT-4.5-nano |
| Voice Speed | 1.2x |
| Interruption Sensitivity | 0.95 (highly responsive) |
| Dynamic Responsiveness | Enabled |
| Denoising | Noise cancellation |
| Max Call Duration | 60 minutes |
| Response Engine | Conversation Flow (32 nodes) |

**Tools registered on the agent (7 total):**

| Tool | Type | Description |
|---|---|---|
| `Create_Patient` | Custom webhook | Registers a new patient — sends `first_name`, `last_name`, `dob`, `phone`, `sex`, `insurance` |
| `GET-Patient-Information` | Custom webhook | Looks up existing patient by phone number |
| `GET-available-appointment` | Custom webhook | Fetches open slots for Dr. Smith Davis |
| `POST--appointments-create` | Custom webhook | Books appointment with full patient + slot details |
| `check_specific_slot` | Custom webhook | Validates a specific requested time slot |
| `check_appt_detail` | Custom webhook | Retrieves existing appointment by name + DOB |
| `update_appt_detail` | Custom webhook | Modifies an existing appointment time |

**Global system prompt highlights:**
- Chelsea is instructed to ask one question at a time and always wait for a response before proceeding
- Refuses to answer questions outside the clinic's scope
- Clinic hours, services, insurance, parking, and amenities are all baked into context
- Always books the earliest available slot unless the patient specifies otherwise

---

### ⚙️ Make.com — Automation & Orchestration

**Scenario:** `Houston Medical Center Flow (Retell+OpenEMR)`

The scenario uses a **webhook trigger** (`Retell_Conversational_Voice`) and routes each incoming tool call to the correct OpenEMR API operation via a `BasicRouter`. Each route is **filter-gated by `tool_call_name`**, so a single webhook endpoint handles all 7 agent tools.

**Route map:**

| Filter | HTTP Method | OpenEMR Endpoint | Purpose |
|---|---|---|---|
| `create_patient` | POST | `/apis/default/api/patient` | Create new patient record |
| `GET-Patient-Information` | GET | `/apis/default/api/patient?phone_cell={{phone}}` | Look up patient by phone |
| `POST--appointments-create` | POST | `/apis/default/api/patient/{{patient_id}}/appointment` | Book appointment |
| `GET-available-appointment` | GET | `/apis/default/api/appointment?pc_aid=5` | Fetch available slots |
| `check_specific_slot` | GET | `/apis/default/api/appointment?pc_aid=5` | Validate specific slot |

**Payload sent to OpenEMR on patient creation:**
```json
{
  "fname": "{{first_name}}",
  "lname": "{{last_name}}",
  "DOB": "{{formatDate(dob, 'YYYY-MM-DD')}}",
  "phone_cell": "{{phone}}",
  "sex": "{{sex}}"
}
```

**Appointment booking payload:**
```json
{
  "pc_aid": "5",
  "pc_facility": 3,
  "pc_catid": 5,
  "pc_eventDate": "{{formatDate(slot_datetime, 'YYYY-MM-DD')}}",
  "pc_startTime": "{{formatDate(slot_datetime, 'HH:mm')}}"
}
```

**Error handling:** Each route has a secondary `BasicRouter` that branches on success or failure, returning a structured webhook response to Chelsea in both cases — so the agent always gets a meaningful reply to continue the conversation.

Authentication to OpenEMR uses **OAuth 2.0** via Make's native credential manager.

---

### 🏥 OpenEMR — Electronic Medical Records

- **Instance:** `demo.dafiniti-emr.com` (Houston Medical Center deployment)
- **Auth:** OAuth 2.0 — token managed via Make.com connection
- **APIs used:** Patient CRUD, Appointment availability, Appointment creation
- **Provider:** Dr. Smith Davis (`pc_aid: 5`, `pc_facility: 3`)
- Open-source EMR — no vendor lock-in, REST API documented and extensible

---

## Architecture Highlights

| Concern | Approach |
|---|---|
| **Latency** | Retell AI's sub-200ms voice response keeps conversations natural |
| **Routing** | Single webhook endpoint — `tool_call_name` field routes all 7 tools |
| **Data integrity** | Make.com validates all fields; date formatting normalized before EMR write |
| **Fault tolerance** | Every route has success + failure branches; Chelsea receives structured feedback either way |
| **Multilingual** | English-first; Spanish patients trigger an agent swap to a dedicated Spanish agent |
| **Interoperability** | OpenEMR's open REST API avoids proprietary lock-in |

---

## Current MVP Capabilities

- [x] Inbound call handling with AI voice agent (Chelsea / Rita voice)
- [x] Language detection — English flow + Spanish agent transfer
- [x] New patient registration with full intake collection
- [x] Existing patient lookup by phone number
- [x] Real-time appointment availability check
- [x] Appointment booking written directly to OpenEMR
- [x] Slot-specific validation before booking
- [x] Graceful error handling — agent always receives a usable response
- [x] Post-call analysis via GPT-4.5-nano

### Roadmap
- [ ] Outbound call campaigns for appointment reminders
- [ ] Real-time insurance eligibility verification
- [ ] SMS/email confirmation after booking
- [ ] Multi-provider support (currently single doctor)
- [ ] Analytics dashboard for call volume and outcomes
- [ ] EHR expansion (Epic, Athena integrations)

---

## Project Structure

```
clinia/
├── make/
│   └── Houston_Medical_Center_Flow_blueprint.json   # Importable Make.com scenario
├── retell/
│   └── Houston_Medical_Center_Receptionist.json     # Retell AI agent blueprint
├── docs/
│   ├── architecture.md       # Detailed system design
│   └── conversation-flow.md  # Node-by-node agent walkthrough
└── README.md
```

---

## Setup / Replication

> ⚠️ This project integrates with external services. You will need active accounts on Retell AI and Make.com, and a running OpenEMR instance.

### Prerequisites
- Retell AI account — import the agent blueprint
- Make.com account — import the scenario blueprint
- OpenEMR instance (self-hosted or demo)

### Steps
1. Import `make/Houston_Medical_Center_Flow_blueprint.json` into Make.com
2. Update the `Retell_Conversational_Voice` webhook URL in Retell's tool settings
3. Configure OpenEMR OAuth credentials in Make.com's connection manager
4. Import `retell/Houston_Medical_Center_Receptionist.json` into Retell AI
5. Publish the agent and assign a phone number

---

## About

Clinia was built to explore how AI can reduce friction at the point of entry into the healthcare system — making care more accessible for patients and more manageable for the clinics serving them.

This project reflects hands-on experience integrating production AI voice systems with real healthcare infrastructure, building automation pipelines that handle sensitive patient data reliably, and navigating the operational constraints of clinical environments.

---

*Built with Retell AI · Make.com · OpenEMR*
