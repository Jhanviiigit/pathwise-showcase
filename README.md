# PathWise — Context-Aware Multi-Agent Routing

> AI-powered routing that understands your vehicle, your environment, and your intent.

## 🎥 Demo video

[![PathWise demo video](https://img.youtube.com/vi/HbhLA6_D-FM/maxresdefault.jpg)](https://youtu.be/HbhLA6_D-FM)

▶️ [Watch on YouTube](https://youtu.be/HbhLA6_D-FM)

> **Note:** This is a showcase repository. The source code is private — access can be shared on request for hiring or review purposes.

---

## What it does

PathWise is a multi-agent AI routing system built on Google's AI and Maps stack. Standard navigation optimises for time. PathWise also asks:

- **Can my vehicle actually fit on this road?** (computer-vision road-width estimation)
- **Is this route beautiful right now?** (Gemini Vision scenic scoring with season, weather and golden-hour context)
- **Is it safe for my vehicle in today's conditions?** (weather-aware risk scoring)

The user enters a destination, a vehicle type, and a mode — **Touristy** or **Time Crunch**. An orchestrator built on Google ADK runs four specialist agents in parallel, each producing a JSON output. A Gemini fusion layer reads all four, ranks the routes by mode, and narrates the recommendation conversationally, like a tour guide.

---

## Example result

**Touristy mode — Bangalore → Madikeri (Coorg)**

| Route | Distance | Duration | Scenic Index | Recommended |
|-------|----------|----------|--------------|-------------|
| NH 75 | 275.4 km | 341 mins | 0.558 | ✅ |
| NH 275 | 257.0 km | 294 mins | 0.553 | — |

NH 75 wins in Touristy mode despite being 47 minutes longer — Gemini Vision scored plantation canopy and open sky near Madikeri at 0.925, the highest single-frame score on either route.

---

## Architecture

```
User request (vehicle · destination · Touristy / Time Crunch mode)
        ↓
PathWise Orchestrator — Google ADK · Vertex AI Agent Builder
        ↓ dispatches all routes to all agents simultaneously
┌─────────────────────────────────────────────────────────┐
│ PHASE 1 — parallel filtering (independent)              │
│  Gully Detection Agent     → widths.json                │
│  Vehicle Suitability Agent → vehicle_profile.json       │
└─────────────────────────────────────────────────────────┘
        ↓
┌─────────────────────────────────────────────────────────┐
│ PHASE 2 — parallel scoring (independent)                │
│  Scenic Route Agent        → scenic_profile.json        │
│  Vehicle Scoring Agent     → safety_score.json          │
└─────────────────────────────────────────────────────────┘
        ↓
Orchestrator aggregates all 4 JSONs
        ↓
Gemini Fusion Layer → final_recommendation.json
        ↓
Touristy (highest scenic index) | Time Crunch (shortest duration)
        ↓
Final recommendation + narration
```

**Key design principle:** every agent is independent. No agent calls another directly — all communication goes through JSON contracts managed by the orchestrator, so any agent can be swapped, upgraded or scaled without touching the rest of the system.

---

## The agents

### 🛣️ Gully Detection Agent — *how wide is this road, really?*
Fetches satellite imagery for ten checkpoints along each route, builds a road graph, extracts the road centreline via skeletonization, and measures width in real-world metres using a distance transform.

*Maps Tiles API · OSMnx · OpenCV · skeletonization* → **`widths.json`** — segment-by-segment width profile per route

### 🚙 Vehicle Suitability Agent — *can this vehicle physically use this road?*
Uses Gemini on Vertex AI to reason over measured road widths against vehicle dimensions, classifying every segment as **PASSABLE**, **CAUTION** or **NOT PASSABLE** for the specific vehicle type.

*Vertex AI · Gemini* → **`vehicle_profile.json`**

### 🌄 Scenic Route Agent — *how beautiful is this route right now?*
Samples five Street View frames from the scenic window of each route (skipping the urban start section). Each frame is classified by Gemini 2.5 Flash Vision for scenic quality, dominant scenery type and obstructions. The raw score is then adjusted by four contextual weights:

- **Golden hour** — 1.5× within 30 minutes of sunrise/sunset
- **Season** — monsoon boosts water bodies and greenery; winter boosts open sky
- **Weather** — fog in the Ghats scores 1.3×; heavy rain is penalised
- **Nearby nature** — waterfalls, viewpoints and forests within 300 m add up to +0.30

It also raises road-risk flags from weather data (wet road, fog risk, flood-prone segments) that the Vehicle Scoring Agent consumes via the orchestrator.

*Street View Static API · Gemini 2.5 Flash Vision · Open-Meteo · OSM Overpass · Places API* → **`scenic_profile.json`**

### 🛡️ Vehicle Scoring Agent — *how risky is this route for this vehicle?*
A FastAPI microservice that computes a 0–100 risk score from the vehicle profile, road widths and weather flags, with bottleneck analysis. It sets two explicit flags per route — safe for Touristy mode and safe for Time Crunch mode — which gate the final recommendation.

*FastAPI* → **`safety_score.json`**

### 🧠 Gemini Fusion Layer
Reads all four agent outputs plus the user's chosen mode, ranks the routes, and blocks any route that fails the safety gate for that mode.

It then narrates the result with a **PathWise tour-guide personality** — translating raw scores into human terms, describing what the driver will see, flagging risks plainly, and ending with a clear best bet.

*Gemini · Vertex AI* → **`final_recommendation.json`**

---

## Routing modes

| Mode | Optimises for | Safety gate |
|------|---------------|-------------|
| **Touristy** | Highest scenic index | Route must be safe for Touristy mode |
| **Time Crunch** | Shortest duration | Route must be safe for Time Crunch mode |

In Time Crunch mode, scenic scoring is skipped entirely to reduce latency.

---

## Tech stack

- **AI & Agents:** Google ADK · Vertex AI Agent Builder · Gemini 2.5 Flash (text + vision)
- **Maps:** Routes API · Maps Tiles API · Street View Static API · Places API (New) · Roads API
- **Computer vision:** OpenCV · OSMnx · skeletonization
- **Backend:** Python · FastAPI
- **Frontend:** React · TypeScript · Vite · Tailwind CSS
- **Infra:** Google Cloud Run · Google Cloud Storage
- **Open data:** Open-Meteo (weather) · OpenStreetMap Overpass (nature tags)

---

## Design decisions

- **No RAG, by choice.** Each agent queries live data sources (weather, OSM, Places) directly. This removed vector-store maintenance, cut latency and simplified deployment without losing contextual accuracy.
- **Time Crunch skips scenic scoring** entirely to keep responses fast.
- **JSON contracts between agents** make it easy to add new ones — planned extensions include pothole detection, air-quality scoring and EV-charging awareness.

---

## Team

Built at a Google hackathon by the PathWise team, MSRIT Bangalore — **Jhanvi, Sian, Vibha and Aditi**.

---

© All rights reserved. This repository is for portfolio viewing only.
