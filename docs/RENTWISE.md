# RentWise — AI-Powered Rental Property Platform

> **AntBox Relevance:** Full-stack platform engineering with embedded ML inference, cloud storage pipelines, JWT auth, and responsive design — the same patterns needed for AntBox's student portal and corporate dashboard.

---

## What It Is

A modern rental property marketplace with an embedded machine learning engine that predicts fair rental values. Tenants compare listings against algorithmic price estimates. Landlords get instant pricing intelligence.

**Source:** [github.com/Subhash-Jetty/RentWise](https://github.com/Subhash-Jetty/RentWise)

---

## Architecture

```
┌───────────────────────────────────────────────────┐
│              BROWSER (React SPA)                   │
│  Vite + React + Tailwind CSS + Framer Motion      │
│  Dark/Light Theme │ Responsive Catalog │ Wishlists │
└──────────────────────┬────────────────────────────┘
                       │ HTTPS + JWT Bearer Token
                       ▼
┌───────────────────────────────────────────────────┐
│              FLASK REST API                        │
│  Gunicorn WSGI │ Flask-JWT-Extended │ Werkzeug     │
│                                                    │
│  ┌─────────────┐  ┌───────────────┐  ┌──────────┐│
│  │Auth Routes  │  │Property Routes│  │ML Routes ││
│  │/register    │  │/properties    │  │/predict  ││
│  │/login       │  │/images        │  │/analyze  ││
│  │/refresh     │  │/reviews       │  │          ││
│  └──────┬──────┘  └──────┬────────┘  └─────┬────┘│
│         │                │                  │     │
│         ▼                ▼                  ▼     │
│  ┌────────────┐  ┌──────────────┐  ┌───────────┐ │
│  │SQLAlchemy  │  │Cloudflare R2 │  │Random     │ │
│  │ORM + DB    │  │(Boto3)       │  │Forest     │ │
│  └────────────┘  └──────────────┘  │Regressor  │ │
│                                    └───────────┘ │
└───────────────────────────────────────────────────┘
```

---

## Technology Stack

### Frontend

| Technology | Purpose |
|:---|:---|
| **React** | Component-based UI with dynamic state management |
| **Vite** | Fast dev server with HMR, optimized production builds |
| **Tailwind CSS** | Utility-first styling with class-based dark/light theme support |
| **Framer Motion** | Smooth micro-interactions, page transitions, hover animations |
| **Axios** | HTTP client for JWT-authenticated API calls |
| **React Router** | Client-side routing for SPA navigation |

### Backend

| Technology | Purpose |
|:---|:---|
| **Python 3.12** | Primary backend language |
| **Flask** | Lightweight HTTP framework for REST API |
| **Gunicorn** | Production WSGI server (not Flask dev server) |
| **SQLAlchemy** | Database-agnostic ORM (PostgreSQL + SQLite) |
| **Flask-JWT-Extended** | Token-based auth with refresh tokens and blacklisting |
| **Werkzeug** | Secure password hashing (bcrypt-based) |
| **Boto3** | AWS-compatible SDK for Cloudflare R2 object storage |

### ML System

| Component | Detail |
|:---|:---|
| **Algorithm** | Random Forest Regressor |
| **Library** | Scikit-Learn |
| **Features** | Location, size, amenities, property type, locality |
| **Output** | Fair market rental estimate |

---

## ML Price Prediction Engine

### Why Random Forest?

| Consideration | Random Forest Advantage |
|:---|:---|
| Non-linear relationships | Trees naturally capture non-linear price drivers (location × size interactions) |
| Feature importance | Built-in feature importance ranking without external explainability tools |
| Robustness | Ensemble of 100+ trees reduces variance; resistant to outliers |
| No feature scaling needed | Tree-based models don't require normalization/standardization |

### Data Processing Pipeline

```
Raw Property Data
    │
    ▼
Feature Engineering
    ├── Categorical encoding (location, property_type)
    ├── Numerical scaling (area, bedrooms, bathrooms)
    ├── Missing value imputation
    └── Outlier detection and capping
    │
    ▼
Random Forest Training
    ├── Train/Test split (80/20)
    ├── Cross-validation
    └── Hyperparameter tuning
    │
    ▼
Serialized Model (.pkl)
    │
    ▼
Flask API endpoint: /predict
    └── Accepts property features → Returns estimated rent
```

### Price Classification

Beyond raw prediction, the system classifies listings:
- **Below Market** — listed price < predicted price (good deal for tenants)
- **At Market** — within ±10% of prediction
- **Above Market** — listed price > predicted price (overpriced)

**AntBox parallel:** The same classification pattern applies to student readiness tiers — categorizing candidates as "Needs Development / On Track / Sprint-Ready / PPO Candidate" based on calibrated score thresholds.

---

## Database Design

```
┌─────────────┐     ┌─────────────────┐
│   users     │     │   properties    │
│             │────<│                 │
│ id          │     │ owner_id        │
│ name        │     │ title           │
│ email       │     │ description     │
│ password    │     │ price           │
│ role        │     │ location        │
│ created_at  │     │ property_type   │
└──────┬──────┘     │ area            │
       │            │ bedrooms        │
       │            │ bathrooms       │
       │            │ amenities       │
       │            └────┬──────┬─────┘
       │                 │      │
       │     ┌───────────▼┐  ┌──▼────────────┐
       │     │   images   │  │   reviews     │
       │     │            │  │               │
       │     │ property_id│  │ property_id   │
       │     │ url (R2)   │  │ user_id       │
       │     │ order      │  │ rating        │
       │     └────────────┘  │ comment       │
       │                     └───────────────┘
       │
       │     ┌────────────┐
       └────<│  wishlist  │
             │            │
             │ user_id    │
             │ property_id│
             └────────────┘
```

---

## Cloud Storage Pipeline (Cloudflare R2)

```
User uploads property images
    │
    ▼
Flask receives multipart form data
    │
    ▼
Boto3 streams to Cloudflare R2 bucket
    │ (S3-compatible API)
    │
    ▼
R2 returns public URL
    │
    ▼
URL stored in database (images table)
    │
    ▼
Frontend displays from R2 CDN
    (zero bandwidth cost to application server)
```

**Why R2 over S3?**
- Zero egress fees (S3 charges for every byte served to users)
- S3-compatible API (same Boto3 code, just different endpoint)
- Global CDN distribution

**AntBox parallel:** Students upload portfolio artifacts, code submissions, and video assessments. Streaming these to object storage (not the application database) keeps the platform fast and costs predictable.

---

## Security Features

| Feature | Implementation |
|:---|:---|
| Password hashing | Werkzeug (bcrypt-based, salted) |
| JWT authentication | Flask-JWT-Extended with access + refresh tokens |
| Token blacklisting | Revoked tokens tracked in database |
| Authorization | Role-based — owners can only edit their own properties |
| Input validation | Pydantic-style validation on all endpoints |
| CORS | Flask-CORS with configured allowed origins |

---

## Key Engineering Decisions

| Decision | Why |
|:---|:---|
| Flask over Django | Lightweight API focus; no admin panel or template engine needed |
| SQLAlchemy over raw SQL | Database portability (dev on SQLite, prod on PostgreSQL) |
| Gunicorn over Flask dev server | Production-grade concurrent request handling |
| Cloudflare R2 over local file storage | Scalable, CDN-backed, zero-egress image delivery |
| Framer Motion over CSS animations | Declarative animation API with spring physics and gesture support |

---

*RentWise demonstrates the complete full-stack engineering cycle: responsive frontend, secure authenticated API, cloud storage, and embedded ML — the same foundation AntBox's student-facing platform runs on.*
