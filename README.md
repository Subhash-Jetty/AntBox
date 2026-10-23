# XENO CRM — AI-Native Customer Engagement Platform

[![Live Demo](https://img.shields.io/badge/🚀_Live_Demo-x--crm--one.vercel.app-8e43ac?style=for-the-badge)](https://x-crm-one.vercel.app/)
[![GitHub](https://img.shields.io/badge/GitHub-X--CRM-181717?style=for-the-badge&logo=github)](https://github.com/Subhash-Jetty/X-CRM)
[![Next.js](https://img.shields.io/badge/Next.js-16-black?style=for-the-badge&logo=next.js&logoColor=white)](https://nextjs.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-Python-009688?style=for-the-badge&logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-Supabase-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)](https://www.postgresql.org/)

> **One sentence, one campaign, zero clicks.**  
> A marketer types *"Find customers who spent over ₹5,000 and send them a WhatsApp campaign with 20% off cold brew"* — and the AI finds the customers, builds the segment, drafts the message, launches the campaign, and tracks every delivery.

### 🔗 **[Try the Live Platform →](https://x-crm-one.vercel.app/)**

---

## Why This Project Exists in an AntBox Submission

[AntBox](https://theantbox.com) is a **Talent Readiness AI Platform** that replaces traditional interviews with **Proof of Work**. It runs Career Sprints, scores students across 100+ performance metrics via an AI Readiness Engine, and manages the entire talent supply chain — from university cohort onboarding to corporate Pre-Placement Offers.

**XENO CRM solves the same class of engineering problems AntBox faces every day:**

| What AntBox Needs to Build | What XENO CRM Already Does |
|:---|:---|
| AI that evaluates student submissions and generates structured outputs | AI that parses natural language and executes structured database operations via tool calling |
| Multi-channel student engagement (WhatsApp, SMS, Email) across 4-12 week sprints | Omnichannel campaign engine with WhatsApp / SMS / Email / RCS, personalization, and delivery tracking |
| Segment students into cohorts by performance, skills, and readiness tier | Dynamic segmentation engine with composable rules, materialized membership, and AI-generated audiences |
| Real-time dashboards for corporate partners showing sprint metrics | Live campaign analytics with sent → delivered → opened → clicked → converted funnels |
| Scalable async backend handling thousands of concurrent evaluations | FastAPI async backend with PostgreSQL, SQLAlchemy 2.0 async, and non-blocking I/O |
| LLM integration with fallback when APIs are rate-limited during peak sprints | Dual-provider AI — Groq (Llama 3.3 70B) primary with Gemini Flash automatic failover |

**This isn't a hypothetical mapping. The architecture is identical.**

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        USER INTERFACE                               │
│          Next.js 16 · React 19 · TypeScript · Glassmorphism         │
│                                                                     │
│  ┌─────────┐  ┌──────────┐  ┌───────────┐  ┌────────────────────┐  │
│  │ AI Chat │  │ Segments │  │ Campaigns │  │ Customer Explorer  │  │
│  │ Console │  │ Builder  │  │ Manager   │  │ & Analytics        │  │
│  └────┬────┘  └────┬─────┘  └─────┬─────┘  └────────┬───────────┘  │
└───────┼─────────────┼──────────────┼─────────────────┼──────────────┘
        │             │              │                 │
        └─────────────┴──────────────┴─────────────────┘
                              │
                              ▼  HTTPS / REST API
┌─────────────────────────────────────────────────────────────────────┐
│                     APPLICATION BACKEND                              │
│               FastAPI · Python 3.12 · Uvicorn · Async               │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    AI ORCHESTRATION LAYER                     │   │
│  │                                                              │   │
│  │   User Query ──→ Intent Classification ──→ Tool Selection    │   │
│  │                                                              │   │
│  │   ┌─────────────────────┐    ┌─────────────────────────┐     │   │
│  │   │   Groq              │    │   Gemini Flash          │     │   │
│  │   │   Llama 3.3 70B     │───→│   (Automatic Fallback)  │     │   │
│  │   │   Sub-second latency│    │   High availability     │     │   │
│  │   └─────────┬───────────┘    └─────────────────────────┘     │   │
│  │             │                                                │   │
│  │   ┌─────────▼───────────────────────────────────────────┐    │   │
│  │   │              TOOL CALLING ENGINE                     │    │   │
│  │   │                                                     │    │   │
│  │   │  query_customers  ·  create_segment  ·  draft_message│   │   │
│  │   │  create_campaign  ·  send_campaign   ·  get_stats    │   │   │
│  │   │  get_insights                                        │   │   │
│  │   └─────────────────────────────────────────────────────┘    │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌──────────────────┐  ┌─────────────────┐  ┌──────────────────┐   │
│  │ Segmentation     │  │ Campaign Engine │  │ Ingestion        │   │
│  │ Engine           │  │ & Dispatch      │  │ Pipeline         │   │
│  │ (segmentation.py)│  │(campaign_engine)│  │ (ingestion.py)   │   │
│  └────────┬─────────┘  └────────┬────────┘  └────────┬─────────┘   │
└───────────┼──────────────────────┼─────────────────────┼────────────┘
            └──────────────────────┼─────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      DATA LAYER                                     │
│          PostgreSQL (Supabase) · SQLAlchemy 2.0 Async               │
│                                                                     │
│  customers · orders · segments · segment_members                    │
│  campaigns · communications · ai_conversations                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## The AI — Not a Chatbot, an Autonomous Agent

Most "AI-powered" apps bolt a chat window onto a CRUD dashboard. XENO CRM's AI is fundamentally different — it **executes structured operations** against the database through constrained tool calling.

### How It Works

```
User types: "Find customers who spent more than ₹5,000 and haven't 
             ordered in 30 days. Send them a WhatsApp message with 
             20% off cold brew."

                    │
                    ▼
         ┌──────────────────────┐
         │ Groq Llama 3.3 70B  │
         │ Intent Decomposition │
         └──────────┬───────────┘
                    │
    ┌───────────────┼───────────────┐
    ▼               ▼               ▼
Step 1          Step 2          Step 3
query_customers create_segment  draft_message
{                {               {
 field:           name:           template:
  total_spend,    "High-value     "Hello {{first_name}},
 operator: ">",   Lapsed",        enjoy 20% off..."
 value: 5000,    rules: [...]   }
 inactive: 30d  }
}               
    │               │               │
    ▼               ▼               ▼
Step 4          Step 5          Step 6
create_campaign send_campaign   get_campaign_stats
{               {               → sent: 847
 segment: ...,   campaign: ...,   → delivered: 831
 channel:        batch_send:      → opened: 412
  "whatsapp"     true              → clicked: 89
}               }
```

**Each tool call has a strict JSON schema.** The AI can't hallucinate SQL or execute arbitrary operations. It proposes; the schema validates; the system executes.

### Dual-Provider Architecture — Zero Downtime AI

```
Primary:  Groq (Llama 3.3 70B)  ← Ultra-fast inference for chat UX
                │
                │ if unavailable / rate limited
                ▼
Fallback: Gemini 2.5 Flash      ← Seamless failover, user never sees an error
```

**Why this matters for AntBox:** During peak sprint submission windows, when hundreds of students trigger AI evaluation simultaneously, a single-provider architecture fails. XENO CRM's dual-provider pattern is exactly what AntBox's Readiness Engine needs.

---

## Segmentation Engine — The Pattern Behind AntBox's Cohort Builder

AntBox segments students into readiness tiers. XENO CRM segments customers into marketing audiences. **The underlying engineering is identical:**

### Composable Rule System

```json
[
  { "field": "total_spend", "operator": ">", "value": 5000 },
  { "field": "last_order_date", "operator": "<", "value": "30_days_ago" },
  { "field": "tags", "operator": "contains", "value": "premium" }
]
```

Rules combine with `AND` logic. The engine supports: `>`, `<`, `=`, `!=`, `contains`, `is_null`, `is_not_null`, `in_last_N_days`, `not_in_last_N_days`.

### Why Materialized Segments?

When a segment is created, matching member IDs are **frozen** into `segment_members`:

| Benefit | Why It Matters |
|:---|:---|
| Campaign dispatch is O(1) | No re-evaluating rules at send time across 10,000+ records |
| Audit trail | The exact audience is recorded — even if data changes later |
| Reproducibility | Re-sending the campaign hits the same people, not a different set |

**AntBox parallel:** When a Career Sprint cohort is created, the participant list should be frozen at sprint start. If a student's profile updates mid-sprint, the original cohort composition must remain auditable for corporate partners.

---

## Campaign Engine — Multi-Channel Engagement at Scale

### The Full Lifecycle

```
Segment Created → Members Loaded → Messages Personalized → Dispatched
                                                               │
                          ┌──────────┬──────────┬──────────────┤
                          ▼          ▼          ▼              ▼
                      WhatsApp     SMS       Email           RCS
                          │          │          │              │
                          └──────────┴──────────┴──────┬───────┘
                                                       │
                                              Delivery Receipts
                                                       │
                                        PENDING → SENT → DELIVERED
                                                       │
                                                 OPENED → CLICKED
                                                       │
                                                  CONVERTED
```

### Message Personalization

```
Template:  "Hello {{first_name}}, you've spent {{total_spend}} with us!
            Here's 20% off your next cold brew ☕"

Customer:  { first_name: "Raj", total_spend: "₹7,500" }

Output:    "Hello Raj, you've spent ₹7,500 with us!
            Here's 20% off your next cold brew ☕"
```

### Forward-Only State Machine

Status transitions are **idempotent and irreversible**: `DELIVERED` can never go back to `PENDING`. Even if webhooks arrive out of order, metrics stay consistent.

**AntBox parallel:** Sprint notification workflows — submission confirmations, mentor assignments, PPO offers — need this exact multi-channel, receipt-tracked, personalized communication architecture.

---

## Database Design

```
┌─────────────┐     ┌─────────────┐     ┌─────────────────┐
│  customers  │────<│   orders    │     │    segments      │
│             │     │             │     │                 │
│ name        │     │ customer_id │     │ name            │
│ email       │     │ amount      │     │ rules (JSONB)   │
│ phone       │     │ items (JSON)│     │ customer_count  │
│ total_spend │     │ channel     │     │ ai_created      │
│ order_count │     │ status      │     │ description     │
│ avg_order   │     └─────────────┘     └────────┬────────┘
│ tags (JSONB)│                                  │
└──────┬──────┘                          ┌───────▼────────┐
       │                                 │segment_members │
       │                                 │ (many-to-many) │
       │                                 └────────────────┘
       │
       │     ┌─────────────────┐     ┌──────────────────┐
       └────<│  campaigns      │────<│ communications   │
             │                 │     │                  │
             │ segment_id      │     │ campaign_id      │
             │ message_template│     │ customer_id      │
             │ channel         │     │ status           │
             │ sent / delivered│     │ (per-recipient   │
             │ open / click    │     │  delivery track) │
             │ conversion count│     └──────────────────┘
             └─────────────────┘

┌──────────────────┐
│ ai_conversations │  ← Chat history & context for AI continuity
└──────────────────┘
```

**Why PostgreSQL over MongoDB?** Strong relational structure: customers *have* orders, *belong to* segments, *receive* campaign communications. These joins and aggregations are PostgreSQL's strength. JSONB handles flexible fields (tags, order items, segment rules) without sacrificing relational integrity.

---

## Technology Stack

| Layer | Technology | Why This Choice |
|:---|:---|:---|
| **Frontend** | Next.js 16, React 19, TypeScript | App Router, server components, type safety, Vercel deployment |
| **UI Design** | Custom CSS, Glassmorphism | Full control over animations, dark theme, responsive behavior |
| **Backend** | FastAPI, Python 3.12, Uvicorn | Async-native, auto OpenAPI docs, Pydantic validation, lightweight |
| **ORM** | SQLAlchemy 2.0 (async) | Database abstraction, PostgreSQL + SQLite portability, async sessions |
| **Database** | PostgreSQL (Supabase) | Relational integrity, JSONB flexibility, hosted with management UI |
| **Primary AI** | Groq (Llama 3.3 70B) | Sub-second inference — critical for chat-first UX |
| **Fallback AI** | Gemini 2.5 Flash | Automatic failover if Groq is unavailable or rate-limited |
| **Deployment** | Vercel (frontend), Cloud (backend) | Production-grade, globally distributed |

---

## Key Engineering Decisions & Tradeoffs

| Decision | Alternative | Why I Chose This |
|:---|:---|:---|
| **FastAPI** over Django | Django has admin panel, ORM, templates | This is an API-focused system. Django's batteries add overhead without value here. FastAPI's auto-docs and Pydantic validation are more useful. |
| **Async everywhere** | Sync Flask-style | Concurrent DB queries + AI API calls + webhook sends. Blocking serializes work that should run in parallel. |
| **Materialized segments** over dynamic queries | Re-evaluate rules at dispatch time | Performance at scale + audit trail + reproducibility. The tradeoff is stale membership, but that's acceptable for campaign use cases. |
| **Separate channel service** | Integrated sends | Demonstrates microservice architecture. In production, swap the simulator for Twilio/SendGrid without touching CRM code. |
| **Dual-provider AI** | Single LLM provider | APIs fail. Rate limits hit. The user should never see "Service Unavailable." |
| **Custom CSS** over Material UI | Pre-built component library | The glassmorphic design needed pixel-level control. No framework gives this without fighting its defaults. |

---

## How This Directly Translates to AntBox's Platform

```
┌──────────────────────────────────────────────────────────────────┐
│                    XENO CRM  →  ANTBOX MAPPING                   │
├───────────────────────────┬──────────────────────────────────────┤
│ XENO CRM Feature         │ AntBox Platform Equivalent           │
├───────────────────────────┼──────────────────────────────────────┤
│ AI Agent with 7 tools     │ AI Readiness Engine that evaluates   │
│ (query, segment, draft,   │ student submissions, generates      │
│  campaign, send, stats,   │ readiness scores, and provides      │
│  insights)                │ actionable feedback                  │
├───────────────────────────┼──────────────────────────────────────┤
│ Composable segmentation   │ Cohort builder for Career Sprints — │
│ with materialized members │ group students by skill, university, │
│                           │ performance tier, sprint status      │
├───────────────────────────┼──────────────────────────────────────┤
│ WhatsApp / SMS / Email /  │ Student engagement across sprint    │
│ RCS campaign dispatch     │ lifecycle — reminders, feedback,    │
│                           │ PPO notifications, mentor alerts    │
├───────────────────────────┼──────────────────────────────────────┤
│ Delivery receipt tracking │ Sprint submission tracking —        │
│ (sent→delivered→opened→   │ submitted → evaluated → scored →   │
│  clicked→converted)       │ reviewed → PPO offered             │
├───────────────────────────┼──────────────────────────────────────┤
│ Dual-provider LLM with    │ Fault-tolerant AI scoring that     │
│ automatic failover        │ doesn't crash during peak sprint   │
│                           │ evaluation windows                  │
├───────────────────────────┼──────────────────────────────────────┤
│ Campaign analytics        │ Corporate partner dashboards —      │
│ dashboard                 │ real-time cohort performance        │
│                           │ metrics and conversion rates        │
├───────────────────────────┼──────────────────────────────────────┤
│ PostgreSQL + async Python │ Scalable backend handling 10,000+   │
│ + Next.js frontend        │ concurrent students and corporates  │
└───────────────────────────┴──────────────────────────────────────┘
```

---

## Run Locally

```bash
# Frontend
cd frontend
npm install
npm run dev
# → http://localhost:3000

# Backend
cd backend
pip install -r requirements.txt
uvicorn main:app --reload
# → http://localhost:8000/docs (auto-generated API docs)
```

**Environment variables:** `GROQ_API_KEY`, `GEMINI_API_KEY`, `DATABASE_URL`

---

## What This Submission Proves

AntBox believes the best way to evaluate an engineer is through **Proof of Work** — not a 45-minute whiteboard session. This submission is itself proof of that philosophy:

- ✅ **Deployed and live** at [x-crm-one.vercel.app](https://x-crm-one.vercel.app/) — not a localhost screenshot
- ✅ **AI that acts, not just chats** — 7 autonomous tool-calling operations
- ✅ **Production architecture** — dual-provider LLM failover, async I/O, materialized data patterns
- ✅ **The same engineering problems** AntBox solves — segmentation, multi-channel engagement, AI evaluation, real-time analytics
- ✅ **Full-stack ownership** — frontend, backend, database, AI orchestration, deployment — one engineer, end to end

---

**Repository:** [github.com/Subhash-Jetty/X-CRM](https://github.com/Subhash-Jetty/X-CRM) · **Live Platform:** [x-crm-one.vercel.app](https://x-crm-one.vercel.app/)  
**Engineered by [Subhash](https://github.com/Subhash-Jetty) for [AntBox](https://theantbox.com)**
