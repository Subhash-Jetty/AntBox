# VoltPath — EV Trip Planning & Charging Intelligence Platform

> **AntBox Relevance:** Enterprise-grade Java backend engineering with Spring Boot, spatial computation, interactive visualization, and polyglot capability (Java + Python + TypeScript) proving ability to work across any stack AntBox needs.

---

## What It Is

An electric vehicle trip-planning application that calculates whether an EV can travel from origin to destination and recommends charging stations when the vehicle cannot complete the journey on a single charge. It combines a Spring Boot spatial computation API with an interactive React map interface.

**Source:** [github.com/Subhash-Jetty/VoltPath](https://github.com/Subhash-Jetty/VoltPath)

---

## Architecture

```
┌───────────────────────────────────────────────────────────────┐
│                    BROWSER (React SPA)                        │
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐│
│  │ React Leaflet│  │   Three.js   │  │  TripForm + Route   ││
│  │ Interactive  │  │  "Hyperspeed"│  │  Result Components  ││
│  │ Map + Markers│  │  Landing FX  │  │  Vehicle Selection  ││
│  └──────┬───────┘  └──────────────┘  └──────────┬───────────┘│
│         └───────────────────────────────────────┘            │
│                            │                                  │
└────────────────────────────┼──────────────────────────────────┘
                             │ REST API / JSON
                             ▼
┌───────────────────────────────────────────────────────────────┐
│               SPRING BOOT APPLICATION                         │
│                                                               │
│  ┌──────────────┐  ┌───────────────────┐  ┌────────────────┐ │
│  │  Routing     │  │  Haversine Engine │  │  Station       │ │
│  │  Controller  │→ │  + Greedy Planner │→ │  Repository    │ │
│  │  /api/routes │  │  Range-aware      │  │  (JPA)         │ │
│  └──────────────┘  │  10% reserve      │  └───────┬────────┘ │
│                    │  Connector match   │          │          │
│                    └───────────────────┘          │          │
│                                           ┌──────▼────────┐  │
│  Spring Security (CORS, stateless)       │  PostgreSQL   │  │
│  Spring Validation (@Valid)              │  / H2 (test)  │  │
│  Lombok (boilerplate reduction)          └───────────────┘  │
│                                                               │
│  Build: Maven │ Test: JUnit + Spring Test │ Java 17          │
└───────────────────────────────────────────────────────────────┘
```

---

## Technology Stack

### Frontend

| Technology | Purpose |
|:---|:---|
| **React** | Component-based SPA: `TripForm`, `MapView`, `RouteResult` |
| **Vite** | Fast bundler with HMR for development |
| **React Router** | Navigation between landing page and planner page |
| **React Leaflet + Leaflet** | Interactive maps, charging station markers, route polylines, popups |
| **Three.js** | Animated "hyperspeed" visual effect on landing page |
| **Custom CSS** | Glass-style panels, dark map theme, responsive layout |

### Backend

| Technology | Purpose |
|:---|:---|
| **Java 17** | Strongly-typed, enterprise-grade backend language |
| **Spring Boot** | REST API framework with dependency injection and auto-configuration |
| **Spring Web** | REST controllers (`/api/routes/calculate`) |
| **Spring Data JPA** | Entity-to-table mapping, repository abstraction |
| **Spring Validation** | Request validation via `@Valid` annotations |
| **Spring Security** | CORS configuration, stateless API security |
| **Lombok** | Eliminates boilerplate (getters, setters, constructors) |
| **Maven** | Build automation, dependency management, test execution |

### Database

| Environment | Database |
|:---|:---|
| **Production** | PostgreSQL — full relational features, indexing |
| **Testing** | H2 (in-memory) — fast, isolated, disposable per test run |

---

## Core Engineering: Route Planning Algorithm

### The Problem

Given:
- An EV with a known range (e.g., 300 km)
- Origin and destination coordinates
- A database of charging stations with connector types and availability

Calculate: Can the EV make the trip? If not, which charging stations should it stop at?

### The Algorithm: Range-Aware Greedy Planner

```
1. Calculate total distance (Haversine: origin → destination)
2. If distance ≤ range × 0.9 (10% reserve) → direct route, no stops needed
3. Otherwise:
   a. Find all charging stations with compatible connectors
   b. Filter to stations within current remaining range
   c. From reachable stations, pick the one closest to destination
      (greedy "best next station" strategy)
   d. "Charge" at that station (reset range to full)
   e. Repeat from station until destination is reachable
4. If no reachable station exists at any step → route is UNREACHABLE
```

### Why Haversine Distance?

```
Formula:
  a = sin²(Δlat/2) + cos(lat₁) · cos(lat₂) · sin²(Δlon/2)
  c = 2 · atan2(√a, √(1-a))
  d = R · c     where R = 6,371 km
```

Earth is a sphere. Euclidean distance on lat/lon gives wrong answers — especially over distances > 50 km. Haversine is the standard for geographic distance calculation without a full geodesic library. Accurate to ~0.3%.

### Why 10% Reserve?

Real EVs don't run to 0%. A 10% reserve prevents the algorithm from suggesting routes where the driver arrives on empty — matching real-world driving behavior and safety margins.

---

## Key Features

### Vehicle Selection
Users select from a database of EV models with different ranges and connector types. The planner adapts route calculations to each vehicle's specifications.

### Interactive Map Interface
- **Leaflet markers** for origin, destination, and each charging stop
- **Route polylines** showing the calculated path
- **Station popups** with connector type, availability status, and distance info
- **Click-to-select** coordinates directly on the map

### Connector Compatibility Filtering
Not all chargers work with all EVs. The planner filters stations by connector type (CCS, CHAdeMO, Type 2, etc.) to only suggest compatible stops.

### Reachable vs. Unreachable Results
The UI clearly communicates whether a route is possible. If unreachable, it explains why (e.g., "No compatible charging station within 270 km of your current range").

---

## Clean Layered Architecture

```
┌──────────────────────────────────────────────────┐
│                   Controllers                     │
│  @RestController, @RequestMapping                │
│  Input validation, HTTP status codes, DTOs       │
├──────────────────────────────────────────────────┤
│                    Services                       │
│  @Service, @Transactional                        │
│  Route planning logic, Haversine computation     │
│  Station filtering, range calculations           │
├──────────────────────────────────────────────────┤
│                  Repositories                     │
│  @Repository, Spring Data JPA                    │
│  Charging station queries, vehicle lookups       │
│  PostgreSQL (prod) / H2 (test)                   │
├──────────────────────────────────────────────────┤
│                   Entities                        │
│  @Entity, @Table, Lombok annotations             │
│  ChargingStation, Vehicle, RouteResult           │
└──────────────────────────────────────────────────┘
```

**Why this separation matters:**
- Controllers handle HTTP only — no business logic leaks into the API layer
- Services contain pure planning logic — testable without HTTP framework
- Repositories abstract database access — swap PostgreSQL for H2 without changing service code
- This is the same architecture pattern used in production enterprise systems

---

## API Structure

| Endpoint | Method | Description |
|:---|:---|:---|
| `/api/routes/calculate` | POST | Calculate EV route with charging stops |
| `/api/stations` | GET | List all charging stations |
| `/api/stations/{id}` | GET | Get station details and availability |
| `/api/vehicles` | GET | List available EV models and ranges |

Request validation via `@Valid`:
```json
{
  "originLat": 28.6139,
  "originLon": 77.2090,
  "destLat": 19.0760,
  "destLon": 72.8777,
  "vehicleId": 1
}
```

---

## Why This Demonstrates Polyglot Engineering

Most SDE candidates submit projects in one language. This portfolio spans:

| Language | Project | Demonstrates |
|:---|:---|:---|
| **Python** | RiskFlow, QueueCTL, RentWise, XENO CRM (backend) | ML pipelines, distributed systems, REST APIs |
| **TypeScript** | XENO CRM (frontend) | Type-safe React, Next.js App Router |
| **JavaScript** | RentWise, VoltPath (frontend) | React SPAs, WebGL, interactive maps |
| **Java** | VoltPath (backend) | Enterprise architecture, Spring ecosystem |
| **SQL** | All projects | PostgreSQL, SQLite, H2, complex queries |

**AntBox needs polyglot engineers.** The platform spans multiple languages and frameworks. An engineer who can debug a React rendering issue, optimize a Python ML pipeline, AND understand a Java microservice is more valuable than one who only knows one stack deeply.

---

## AntBox Relevance

| VoltPath Capability | AntBox Application |
|:---|:---|
| Haversine distance computation | Geographic hiring analytics — "How many students are within 50km of corporate partner offices?" |
| Range-aware route planning | Sprint milestone planning — breaking large goals into achievable checkpoints |
| Connector compatibility filtering | Skill-to-role matching — filtering candidates by required competency types |
| Spring Boot enterprise architecture | Production backend services for AntBox's platform |
| Interactive map visualization | Geographic cohort distribution dashboards for corporate partners |

---

*VoltPath proves enterprise Java engineering alongside Python and TypeScript fluency — demonstrating the polyglot, full-stack depth AntBox needs to build and scale its platform.*
