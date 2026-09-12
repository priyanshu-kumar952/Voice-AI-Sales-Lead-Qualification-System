
# ☎️ Voice AI Sales & Lead Qualification System

> An automated voice AI sales pipeline designed for high-volume real estate lead intake, inventory qualification, live escalation, and post-call automation.

**Status:** 🏗️ Architecture & Design Complete — Implementation in Progress

---

## 📌 Overview

Real estate developers handling large volumes of inbound inquiries and outbound follow-ups often depend on large telecalling teams.

This project explores a production-oriented **Voice AI workforce** that can automate a large part of that workflow.

The system is designed to:

- Answer inbound property inquiries 24/7
- Handle outbound lead follow-ups
- Query live property inventory in real time
- Match properties against buyer requirements
- Qualify leads using business rules
- Escalate qualified buyers to human sales managers during the call
- Extract structured lead information from call transcripts
- Send brochures and location information through WhatsApp
- Alert the sales team when a high-value lead is identified

The architecture is designed around a real-world real estate sales workflow, with emphasis on **low conversational latency, tool calling, reliability, security, and production scalability**.

> ⚠️ **Important:** This repository currently documents the complete system architecture and implementation plan. Components are being built incrementally. It should not be considered a deployed or fully functioning production system yet.

---

# 🎯 Business Problem

A high-volume real estate sales operation may have:

- Large numbers of daily property inquiries
- Multiple property categories and configurations
- Different buyer budgets and requirements
- Human telecallers limited to fixed working hours
- Manual lead qualification
- Manual brochure and location sharing
- Delays between inquiry and sales-team follow-up

The proposed system aims to automate the repetitive parts of this workflow while keeping human sales managers involved when a lead requires direct intervention.

---

# 🏢 Target Business Context

The system architecture was designed around a high-volume residential/commercial real estate developer in India.

The modeled business case includes:

- ₹5 Crore+ monthly property transactions
- 10–12 closed deals per month
- ~₹50 Lakhs+ average ticket size
- Approximately 20 human telecallers
- Residential plots
- 2/3/4 BHK apartments
- Villas
- Commercial units

The system is intended to support buyer filtering based on:

- Location
- Budget
- Property/unit type
- Size
- Facing
- Possession status
- RERA status
- Loan approval

> **Privacy note:** Business-specific financial figures are generalised. 

---

# 🧠 Core Workflow

```text
                     CUSTOMER / BUYER
                            │
                            ▼
                    ┌───────────────┐
                    │    Exotel     │
                    │  Telephony    │
                    └───────┬───────┘
                            │
                       SIP Audio
                            │
                            ▼
                    ┌───────────────┐
                    │   Retell AI   │
                    │ Voice Engine  │
                    └───────┬───────┘
                            │
                ┌───────────┴───────────┐
                │                       │
                ▼                       ▼
        Inventory Tools          Transfer Call
                │                       │
                ▼                       ▼
        ┌───────────────┐       ┌───────────────┐
        │ FastAPI +     │       │    Exotel     │
        │ Redis         │       │ PSTN Bridge   │
        └───────┬───────┘       └───────┬───────┘
                │                       │
                ▼                       ▼
        ┌───────────────┐        SALES MANAGER
        │   Supabase    │
        │  PostgreSQL   │
        └───────┬───────┘
                │
          Call Completed
                │
                ▼
        ┌───────────────┐
        │      n8n      │
        │ Post-Call     │
        │ Automation    │
        └───────┬───────┘
                │
          ┌─────┴─────┐
          ▼           ▼
     Supabase      WhatsApp
     Lead Data     Brochures /
                   Alerts
````

---

# 🏗️ System Architecture

The system consists of six major layers.

## 1. Telephony Gateway — Exotel

Exotel acts as the telephony layer.

Responsibilities:

* Receive inbound calls
* Support outbound campaigns
* Provide virtual numbers
* Connect telephony audio to the AI voice system
* Support SIP-based communication
* Provide failover behavior through passthru applets

Target transport latency:

```text
~200ms
```

---

## 2. Voice Orchestration — Retell AI

Retell AI handles the conversational voice layer.

### Speech-to-Text

**Deepgram Nova-2**

Target:

```text
< 120ms
```

### LLM / Reasoning

```text
GPT-4.1
GPT-4.1-mini
```

The LLM is responsible for:

* Understanding buyer requirements
* Maintaining conversation context
* Deciding when tools should be called
* Applying qualification logic
* Triggering escalation when required

### Text-to-Speech

**Cartesia**

Target first-audio latency:

```text
< 100ms
```

### Voice Interaction

The architecture also uses:

* Voice Activity Detection (VAD)
* Barge-in support
* Function calling
* Context-aware tool routing

Barge-in allows the AI to stop speaking when the caller begins talking.

---

# ⚙️ 3. Backend Middleware — FastAPI + Redis

The backend acts as the integration and business-logic layer between the voice engine and the application's data.

### FastAPI

The planned backend follows an asynchronous Python architecture with:

```text
Router
   ↓
Service
   ↓
Repository
   ↓
Database / Cache
```

Responsibilities include:

* Webhook handling
* Tool-call endpoints
* Request validation
* Business logic
* Inventory lookup
* Call-transfer handling

### Pydantic

Pydantic validates tool-call payloads such as:

```text
check_inventory
transfer_call
```

### Security

Webhook endpoints are designed to use:

* HMAC signature verification
* Rate limiting
* Strict request validation

---

# ⚡ 4. Redis Caching

Redis is planned as the high-speed caching layer.

Property inventory can be cached in memory so frequently requested inventory queries do not always require a database round trip.

Target:

```text
Database lookup: ~300ms
Cached lookup:   < 5ms
```

This is particularly important for voice applications because database latency directly affects conversational responsiveness.

---

# 🗄️ 5. Database — Supabase / PostgreSQL

Supabase provides the PostgreSQL database layer.

The database is designed to store:

### Property Inventory

* Location
* Budget / price
* Unit type
* Size
* Facing
* Possession status
* RERA status
* Loan approval
* Other property metadata

### Lead Information

* Buyer details
* Requirements
* Budget
* Preferred location
* Unit type
* Qualification status
* Lead score

### Call Data

* Transcripts
* Recording references
* Call metadata

---

## 🔎 Database Indexing

The architecture uses multi-column **B-tree indexes** for high-frequency filtering across fields such as:

```text
location
budget
facing
plot size
possession status
```

The goal is to keep inventory queries fast enough for real-time voice conversations.

---

# 📞 6. Live Sales Escalation

One of the core features is **mid-call human escalation**.

The AI can transfer the caller when:

* The buyer explicitly requests a human
* The buyer crosses a business qualification threshold
* The conversation requires human intervention

Example qualification rule:

```text
Buyer budget > ₹1 Crore
        ↓
Qualified Lead
        ↓
transfer_call
        ↓
SIP REFER
        ↓
Exotel
        ↓
Live Sales Manager
```

This creates a hybrid sales workflow:

```text
AI handles qualification
          +
Human handles high-value conversation
```

---

# 🔄 7. Post-Call Automation — n8n

After a call ends, the system is designed to trigger an automated workflow through **n8n**.

### Pipeline

```text
Call Ends
   ↓
Retell Webhook
   ↓
n8n
   ↓
GPT-4o-mini
   ↓
Structured Lead JSON
   ↓
Supabase
   ↓
WhatsApp Delivery
```

GPT-4o-mini extracts structured information from the transcript, such as:

```json
{
  "name": "Buyer Name",
  "budget": "₹1.2 Crore",
  "location": "Preferred Location",
  "unit_type": "3 BHK",
  "lead_score": "hot"
}
```

The extracted information is then stored in Supabase.

---

# 💬 WhatsApp Automation

The planned WhatsApp layer automatically delivers relevant information after the call.

Possible outputs include:

* Property brochures
* Location/maps
* Property information
* Sales-team alerts

The architecture uses:

**Meta WhatsApp Cloud API**

with providers such as:

```text
Wati
AiSensy
```

A hot-lead notification can also be sent to the sales team.

---

# 🧰 Tech Stack

| Layer                | Technology                    |
| -------------------- | ----------------------------- |
| Telephony            | Exotel                        |
| Voice Orchestration  | Retell AI                     |
| Speech-to-Text       | Deepgram Nova-2               |
| LLM / Reasoning      | OpenAI GPT-4.1 / GPT-4.1-mini |
| Text-to-Speech       | Cartesia                      |
| Backend              | FastAPI                       |
| Language             | Python                        |
| Validation           | Pydantic                      |
| Cache                | Redis                         |
| Database             | Supabase / PostgreSQL         |
| Automation           | n8n                           |
| Post-call Extraction | OpenAI GPT-4o-mini            |
| Messaging            | Meta WhatsApp Cloud API       |
| WhatsApp Providers   | Wati / AiSensy                |
| Planned Hosting      | Render / Railway / Hetzner    |

---

# ⏱️ Latency Budget

Voice AI requires low latency because delays are directly noticeable during a conversation.

| Component                      |        Target |
| ------------------------------ | ------------: |
| STT                            |       < 120ms |
| Backend Query                  |       < 400ms |
| Cached Backend Query           |         < 5ms |
| TTS First Audio                |       < 100ms |
| Telephony Transport            |        ~200ms |
| **Total Conversational Delay** | **600–800ms** |

The targets are architectural goals rather than measured production SLAs at the current stage.

---

# 🔐 Security Considerations

Security is considered at the architecture level because the system processes:

* Buyer information
* Financial requirements
* Call transcripts
* Call recordings
* Lead information

Planned security mechanisms include:

### Webhook Authentication

HMAC signature verification for incoming webhook requests.

### Rate Limiting

Rate limiting on sensitive backend endpoints.

### Request Validation

Pydantic validation for tool-call payloads.

### Data Protection

Production deployment should include:

* Encryption at rest
* Access control
* Secure secret management
* Restricted database access
* Appropriate PII handling

The production design also considers India's **Digital Personal Data Protection (DPDP) Act** requirements.

---

# 📊 Production-Grade Roadmap

The architecture currently covers the primary happy path, but several production concerns remain.

## Observability

Planned:

* Prometheus / OpenTelemetry
* Grafana dashboards
* Sentry
* Distributed tracing

The objective is to answer questions such as:

```text
Why were calls slower today?
Which service caused the latency?
How many calls failed?
Where did a webhook fail?
```

---

## High Availability

The production system needs defined fallback behavior for:

* Redis failures
* Supabase downtime
* OpenAI timeouts
* Deepgram timeouts
* Cartesia failures
* Exotel disconnects

---

## Retry & Queue System

Webhooks cannot be assumed to always succeed.

A queue layer such as:

```text
RabbitMQ
SQS
Redis Streams
```

can sit between the voice platform and post-call automation.

Example:

```text
Retell
  ↓
Webhook
  ↓
Queue
  ↓
n8n
  ↓
Supabase
```

This reduces the risk of losing events during transient failures.

---

# 🖥️ Planned Admin Dashboard

Because the target business does not have an in-house technical team, an administrative dashboard is planned.

The dashboard should allow business users to:

* View today's calls
* Review hot leads
* Update property inventory
* Review recordings
* Monitor AI performance
* Configure sales-manager routing
* Inspect lead information

The goal is to allow routine operations without developer intervention.

---

# 💰 Cost Monitoring

Voice AI costs can scale rapidly with usage.

The production system should track:

```text
STT usage
LLM tokens
TTS usage
Telephony minutes
WhatsApp messages
Infrastructure
```

This allows the business to calculate the operational cost per call and per qualified lead.

---

# 🚧 Current Build Status

| Component                    | Status         |
| ---------------------------- | -------------- |
| System architecture & design | ✅ Complete     |
| FastAPI middleware           | 🔲 Not started |
| Redis caching                | 🔲 Not started |
| Supabase schema              | 🔲 Not started |
| Local STT → LLM → TTS loop   | 🔲 Not started |
| Retell AI integration        | 🔲 Not started |
| Exotel telephony             | ⛔ Blocked      |
| Call transfer                | 🔲 Not started |
| n8n post-call automation     | 🔲 Not started |
| WhatsApp sandbox             | 🔲 Not started |

### Current Blocker

The Exotel telephony layer requires a registered business entity for virtual-number KYC.

This currently blocks the real PSTN/virtual-number portion of the project.

The rest of the architecture can be developed independently using simulated/local audio streams and sandbox environments.

---

# 🛠️ Build Plan

The implementation is planned in the following order:

### 1. Local Voice Loop

Build:

```text
STT → LLM → TTS
```

without PSTN.

Purpose:

* Validate conversational logic
* Test function calling
* Measure local latency

### 2. Backend Middleware

Implement:

```text
FastAPI
+
Redis
+
Pydantic
```

with:

```text
check_inventory
transfer_call
```

tool contracts.

### 3. Database

Create Supabase/PostgreSQL schemas for:

```text
properties
leads
```

and implement the planned indexing strategy.

### 4. Post-Call Automation

Build:

```text
Transcript
   ↓
GPT-4o-mini
   ↓
Structured JSON
   ↓
Supabase
```

and test against sample transcripts.

### 5. WhatsApp Sandbox

Implement brochure and alert delivery using the WhatsApp Cloud API sandbox.

### 6. Telephony Integration

Connect the completed voice pipeline to Exotel once a compliant business/entity path is available.

---

# 🧩 Engineering Challenges

This project focuses on several engineering problems that are different from a conventional web application.

### Real-Time Voice Latency

A voice system cannot tolerate the same delays as a traditional API-driven application.

Every layer contributes to conversational latency:

```text
Caller
 ↓
Telephony
 ↓
STT
 ↓
LLM
 ↓
Tool Call
 ↓
Database / Cache
 ↓
LLM
 ↓
TTS
 ↓
Telephony
 ↓
Caller
```

The architecture therefore separates fast inventory access from slower persistence operations and uses Redis for frequently accessed data.

---

### AI + Deterministic Business Logic

The LLM should not directly control critical business data.

Instead:

```text
LLM
 ↓
Function Call
 ↓
FastAPI
 ↓
Validation
 ↓
Business Logic
 ↓
Database
```

This allows the AI to interact with controlled application tools rather than directly manipulating the database.

---

### Human-in-the-Loop AI

The system is not designed to completely replace sales staff.

Instead:

```text
AI → handles repetitive qualification
Human → handles qualified/high-value leads
```

This provides a practical hybrid model for sales operations.

---

### Failure Handling

A production voice pipeline has many external dependencies:

```text
Exotel
Retell
Deepgram
OpenAI
Cartesia
Redis
Supabase
n8n
WhatsApp
```

The production roadmap therefore includes retries, queues, observability, fallback behavior, and cost monitoring.

---

# 📈 Expected Business Impact

The architecture is intended to help a real estate sales operation:

* Extend lead handling to 24/7
* Reduce repetitive telecalling workload
* Standardize initial qualification
* Respond to inventory questions instantly
* Escalate high-value leads immediately
* Automate post-call data entry
* Deliver brochures without manual follow-up
* Improve lead-response consistency
* Give management better visibility into sales conversations

Actual ROI and performance would need to be measured after the system is implemented and tested with real call volume.

---

# 📁 Planned Project Structure

```text
voice-ai-sales-system/
│
├── backend/
│   ├── app/
│   │   ├── routers/
│   │   ├── services/
│   │   ├── repositories/
│   │   ├── models/
│   │   └── main.py
│   │
│   └── tests/
│
├── database/
│   ├── migrations/
│   └── schema.sql
│
├── n8n/
│   └── workflows/
│
├── prompts/
│   └── voice-agent/
│
├── docs/
│   ├── architecture/
│   ├── api/
│   └── deployment/
│
├── .env.example
├── README.md
└── LICENSE
```

> The exact repository structure may evolve as implementation progresses.

---

# 🎓 What This Project Demonstrates

This project explores practical engineering across:

* Voice AI systems
* LLM function calling
* Real-time conversational systems
* Speech-to-text
* Text-to-speech
* FastAPI
* Async Python
* Redis caching
* PostgreSQL
* Database indexing
* Webhooks
* HMAC authentication
* Rate limiting
* SIP/PSTN architecture
* Human-in-the-loop AI
* n8n workflow automation
* WhatsApp Cloud API
* Distributed-system reliability
* Observability
* PII/data-security considerations
* Production architecture

---

# 🚀 Project Status

**Current stage:**

```text
Architecture
      ↓
Design Complete
      ↓
Implementation
      ↓
Testing
      ↓
Production Integration
```

The architecture and technical design are complete.

Implementation is being developed incrementally, starting with components that do not depend on the telephony-provider KYC requirement.

---

# 👨‍💻 Author

## Priyanshu Kumar

**B.Tech — Computer Science & Technology**
SAGE University, Indore
2026–2030

### Contact

* 📧 Email: `krpriyanshu952@gmail.com`
* 💼 LinkedIn: `linkedin.com/in/anshu-kumar-8735ba377`
* 🐙 GitHub: `github.com/priyanshu-kumar952`
* 🌐 Portfolio: Coming Soon

---

# 📄 License

No license has been specified yet.

Until a `LICENSE` file is added, all rights are reserved by the author.

