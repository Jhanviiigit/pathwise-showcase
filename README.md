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

The user enters a destination, a vehicle type, and a mode — **Touristy** or **Time Crunch**. An orchestrator built on Google ADK runs four specialist agents in parallel, and a Gemini fusion layer ranks the routes and explains the recommendation conversationally, like a tour guide.

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
│ PHASE 1 — parallel filtering                            │
│  Gully Detection Agent     → road width profile         │
│  Vehicle Suitability Agent → vehicle profile            │
└─────────────────────────────────────────────────────────┘
        ↓
┌─────────────────────────────────────────────────────────┐
│ PHASE 2 — parallel scoring                              │
│  Scenic Route Agent        → scenic profile             │
│  Vehicle Scoring Agent     → safety score               │
└─────────────────────────────────────────────────────────┘
        ↓
Gemini Fusion Layer → ranked recommendation + narration
```

**Key design principle:** every agent is independent. No agent calls another directly — all communication goes through JSON contracts managed by the orchestrator, so any agent can be swapped, upgraded or scaled without touching the rest of the system.

---

## The agents

### 🛣️ Gully Detection Agent — *how wide is this road, really?*
Pulls satellite imagery at checkpoints along each route, builds a road graph, extracts the road centreline via skeletonization, and measures width in real-world metres using a distance transform.
*OSMnx · OpenCV · scikit-image · Google Maps Tiles API*

### 🚙 Vehicle Suitability Agent — *can this vehicle physically use this road?*
Uses Gemini on Vertex AI to reason over measured road widths against vehicle dimensions, classifying each segment as **Passable**, **Caution** or **Not Passable**.

### 🌄 Scenic Route Agent — *how beautiful is this route right now?*
Samples Street View frames along the scenic section of each route and scores them with Gemini 2.5 Flash Vision. Scores are adjusted for context:
- **Golden hour** boost near sunrise/sunset
- **Season** weighting (monsoon favours greenery and water, winter favours open skies)
- **Weather** (Ghat fog scores higher; heavy rain is penalised)
- **Nearby landmarks** — waterfalls, viewpoints and forests from OpenStreetMap

It also raises road-risk flags (wet road, fog, flood-prone segments) for the safety agent.

### 🛡️ Vehicle Scoring Agent — *how risky is this route for this vehicle?*
A FastAPI microservice that computes a 0–100 risk score from vehicle profile, road widths and weather flags, and decides whether each route is safe for each mode.

### 🧠 Gemini Fusion Layer
Combines all agent outputs with the user's chosen mode — ranking by scenic index (Touristy) or duration (Time Crunch), gated by the safety flags — and narrates the final recommendation in plain language.

---

## Tech stack

- **AI & Agents:** Google ADK · Vertex AI Agent Builder · Gemini 2.5 Flash (text + vision)
- **Maps:** Routes API · Maps Tiles API · Street View Static API · Places API (New) · Roads API
- **Computer vision:** OpenCV · OSMnx · scikit-image
- **Backend:** Python · FastAPI
- **Frontend:** React · TypeScript · Vite · Tailwind CSS
- **Infra:** Google Cloud Run · Google Cloud Storage
- **Open data:** Open-Meteo (weather) · OpenStreetMap Overpass

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
