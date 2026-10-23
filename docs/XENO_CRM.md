# XENO CRM — AI-Native CRM & Marketing Automation Platform

> **AntBox Relevance:** The exact architecture needed for AntBox's student engagement, mentor communication, and corporate partner notification workflows.

---

## What It Is

An AI-native customer relationship management platform where marketers describe campaigns in plain English and the AI handles everything — finding customers, creating segments, drafting messages, launching campaigns, and tracking delivery.

**Live Demo:** [xenocrm.netlify.app](https://xenocrm.netlify.app)  
**Source:** [github.com/Subhash-Jetty/XenoCRM](https://github.com/Subhash-Jetty/XenoCRM)

### The Core Workflow

```
"Find customers who spent more than ₹5,000 and send them
 a WhatsApp campaign with 20% off cold brew."
                        |
                        v
            ┌───────────────────────┐
            │  AI Intent Classifier │
            │  (Groq Llama 3.3 70B)│
            └─────────┬────────────┘
                      │
            ┌─────────▼────────────┐
            │ Strict Tool Calling  │
            │ Schema Validation    │
            └─────────┬────────────┘
                      │
       ┌──────────────┼──────────────┐
       │              │              │
       ▼              ▼              ▼
 query_customers  create_segment  create_campaign
 (SQL Builder)   (Materialize)   (Omnichannel)
       │              │              │
       └──────────────┼──────────────┘
                      │
            ┌─────────▼────────────┐
            │   PostgreSQL/Supabase │
            └─────────┬────────────┘
                      │
            ┌─────────▼────────────┐
            │  Async Delivery      │
            │  Receipt Tracking    │
            └─────────┬────────────┘
                      │
            ┌─────────▼────────────┐
            │  Next.js 16 + React  │
            │  19 Live Dashboard   │
            └──────────────────────┘
```

---

## Technology Stack

### Frontend
| Technology | Version | Why This Choice |
|:---|:---|:---|
| **Next.js** | 16 | App Router for organized dashboard routes; built-in SSR/SSG; Vercel deployment |
| **React** | 19 | Dynamic UI: chat messages, loading states, campaign stats, modals, forms, pagination, theme switching |
| **TypeScript** | 5.x | Type safety across component props, API responses, and tool-call schemas |
| **Vanilla CSS** | Custom | Full control over glassmorphism design system — no framework constraints on animation or theming |

### Backend
| Technology | Why |
|:---|:---|
| **FastAPI** | Async-native, auto-generated OpenAPI docs, Pydantic validation, lightweight for API-focused architecture |
| **Python 3.12** | Primary language for both API logic and AI orchestration |
| **SQLAlchemy 2.0** | Async ORM with PostgreSQL + SQLite portability |
| **asyncpg / aiosqlite** | Non-blocking database I/O for handling concurrent AI calls + DB queries |
| **Uvicorn** | ASGI server for production async workloads |

### AI System
| Component | Technology | Fallback |
|:---|:---|:---|
| **Primary LLM** | Groq (Llama 3.3 70B) | Sub-second inference for chat-first UX |
| **Fallback LLM** | Google Gemini 2.5 Flash | Activated if Groq is unavailable or rate-limited |
| **Architecture** | Dual-provider with automatic failover | User never sees an error |

### Database
| Component | Technology |
|:---|:---|
| **Production** | PostgreSQL (hosted via Supabase) |
| **Local Dev** | SQLite via aiosqlite |
| **ORM** | SQLAlchemy 2.0 with async sessions |

---

## AI System Deep Dive

### Tool-Calling Architecture

The AI doesn't just chat — it executes structured operations against the database:

| Tool | What It Does | Schema Example |
|:---|:---|:---|
| `query_customers` | Builds parameterized SQL queries from natural language | `{"field": "total_spend", "operator": ">", "value": 5000}` |
| `create_segment` | Materializes a reusable customer cohort with frozen membership | Rules array + member ID snapshot |
| `draft_message` | Generates personalized message templates with `{{placeholders}}` | `"Hello {{first_name}}, you've spent {{total_spend}}..."` |
| `create_campaign` | Creates a multi-channel campaign targeting a segment | Channel + template + segment reference |
| `send_campaign` | Dispatches personalized messages to all segment members | Async batch send with per-recipient tracking |
| `get_campaign_stats` | Returns real-time engagement metrics | Sent/Delivered/Opened/Clicked/Converted counts |
| `get_insights` | Generates analytical summaries from customer data | AI-powered business intelligence |

### Why Dual-Provider AI?

```
User message
     │
     ▼
Groq / Llama 3.3 70B  ← Primary (ultra-fast inference)
     │
     │ if unavailable / rate limited
     ▼
Gemini 2.5 Flash       ← Fallback (high availability)
```

For a chat-first CRM, latency matters. Groq's inference speed makes AI feel instant. But APIs fail. The dual-provider design ensures the user always gets a response.

**AntBox parallel:** The AI Readiness Engine needs the same pattern — fast scoring for real-time student dashboards, with a fallback model for peak evaluation windows.

---

## Database Design

### Entity Relationship

```
┌─────────────┐     ┌─────────────┐     ┌─────────────────┐
│  customers  │────<│   orders    │     │    segments      │
│             │     │             │     │                 │
│ id          │     │ customer_id │     │ id              │
│ name        │     │ amount      │     │ name            │
│ email       │     │ items       │     │ rules (JSONB)   │
│ phone       │     │ channel     │     │ customer_count  │
│ total_spend │     │ status      │     │ ai_created      │
│ order_count │     │ created_at  │     └────────┬────────┘
│ avg_order   │     └─────────────┘              │
│ tags (JSONB)│                          ┌───────▼────────┐
└──────┬──────┘                          │segment_members │
       │                                 │ segment_id     │
       │                                 │ customer_id    │
       │                                 └────────────────┘
       │
       │     ┌─────────────────┐     ┌──────────────────┐
       └────<│  campaigns      │────<│ communications   │
             │                 │     │                  │
             │ segment_id      │     │ campaign_id      │
             │ message_template│     │ customer_id      │
             │ channel         │     │ status           │
             │ sent_count      │     │ (per-recipient   │
             │ delivered_count │     │  tracking)       │
             │ open_count      │     └──────────────────┘
             │ click_count     │
             │ conversion_count│
             └─────────────────┘

┌──────────────────┐
│ ai_conversations │
│ context (JSONB)  │
│ message_history  │
└──────────────────┘
```

### Why PostgreSQL Over MongoDB?

The data has **strong relational structure**: customers have orders, belong to segments, receive campaign communications. PostgreSQL handles these joins efficiently while still supporting JSONB for flexible fields (tags, order items, segment rules).

### Why Materialized Segments?

When a segment is created, matching customer IDs are immediately stored in `segment_members`:

- **Campaign dispatch is O(1)** — no re-evaluating rules at send time
- **Audit trail** — the exact audience is recorded, even if customer data changes later
- **Performance** — 10,000 customers with complex rules would be expensive to re-evaluate per campaign

**AntBox parallel:** Sprint cohorts need the same pattern — freeze the participant list at sprint start, even as new students enroll on the platform.

---

## Campaign Engine

### Multi-Channel Dispatch

```
Campaign Created → Load Segment Members → Personalize Messages → Dispatch
                                                                    │
                                    ┌───────────┬───────────┬───────┴───────┐
                                    ▼           ▼           ▼               ▼
                                WhatsApp      SMS        Email            RCS
                                    │           │           │               │
                                    └───────────┴───────────┴───────┬───────┘
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
Template: "Hello {{first_name}}, you've spent {{total_spend}} with us. 
           Here's 20% off your next cold brew!"

For customer {name: "Raj Kumar", total_spend: 7500}:
→ "Hello Raj, you've spent ₹7,500 with us. Here's 20% off your next cold brew!"
```

### Delivery Receipt State Machine

Status transitions are **forward-only and idempotent** — a message that's been DELIVERED can never go back to PENDING. This prevents metric corruption even if webhooks arrive out of order.

**AntBox parallel:** Sprint notification workflows (submission confirmations, mentor assignment alerts, PPO offers) need exactly this multi-channel, receipt-tracked architecture.

---

## Key Engineering Decisions

| Decision | Alternative Considered | Why I Chose This |
|:---|:---|:---|
| FastAPI over Django | Django has more batteries | This is an API-focused system. Django's template engine, admin panel, and middleware overhead weren't needed. FastAPI's auto-docs and Pydantic validation are more valuable here. |
| Async everywhere | Sync Flask-style | The app makes concurrent DB queries, AI API calls, and webhook sends. Blocking on any of these serializes work that should be parallel. |
| Separate channel service simulator | Integrated sends | Separating the simulator demonstrates microservice architecture. In production, you'd swap it for a real Twilio/SendGrid/WhatsApp Business API without touching CRM code. |
| Glassmorphic CSS over Material UI | Pre-built component library | The design needed a specific visual identity. Custom CSS gives full control over animations, themes, and responsive behavior without framework constraints. |

---

*This system demonstrates AI-agent orchestration, production database design, multi-channel communication, and modern full-stack architecture — all directly transferable to AntBox's talent engagement platform.*
