# TrendSync Brand Factory — Architecture & Technical Documentation

## Platform Overview

TrendSync Brand Factory is an AI-powered fashion design studio that generates complete fashion collections — from trend analysis to product images to tech packs — powered by **DigitalOcean Gradient AI** for the design companion agent, **DO Managed PostgreSQL** for persistence, **DO Spaces** for media storage, **DO Managed Valkey** for caching, and Gemini for image generation and voice streaming.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        FRONTEND (React + Vite)                      │
│                          Port 5173                                  │
│                                                                     │
│  ┌───────────┐  ┌──────────────┐  ┌──────────────┐  ┌───────────┐ │
│  │ Dashboard  │  │ Brand Editor │  │  Collection  │  │  Trends   │ │
│  │           │  │ + Guardian   │  │  + Library   │  │  Insights │ │
│  └───────────┘  └──────────────┘  └──────────────┘  └───────────┘ │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │   Lux Design Companion (DesignAdjustments.tsx)               │  │
│  │   → Single call to POST /adk/design-companion                │  │
│  │   → Agent decides which tool(s) to invoke                    │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                              │                                      │
│                    Supabase Auth + DO PostgreSQL                    │
└──────────────────────────────┼──────────────────────────────────────┘
                               │ HTTP / WebSocket
┌──────────────────────────────┼──────────────────────────────────────┐
│                     MAIN BACKEND (FastAPI)                           │
│                       Port 8000                                     │
│                                                                     │
│  REST Endpoints:                                                    │
│  ├── GET  /health                                                   │
│  ├── POST /trends                    (Gemini + Google Search)       │
│  ├── POST /design/chat               (Gemini Flash — legacy)        │
│  ├── POST /edit-image                (Gemini Flash Image)           │
│  ├── POST /generate-image            (Gemini Flash Image)           │
│  ├── POST /validate                  (Rule-based Brand Guardian)    │
│  ├── POST /generate-techpack         (Gemini Flash)                 │
│  ├── POST /generate-collection       (Background: trends→plan→img) │
│  ├── POST /generate-ad-video         (Background: Veo 3.1)         │
│  ├── POST /adk/pipeline              (Full orchestrator pipeline)   │
│  ├── POST /adk/design-companion      (DO Gradient Agent — Lux)     │
│  └── WS   /ws/voice-companion/:id   (Proxy to voice service)       │
│                                                                     │
│  Shared Modules:                                                    │
│  ├── shared/trend_engine.py          Gemini + Google Search         │
│  ├── shared/image_generator.py       Gemini 3 Pro Image             │
│  ├── shared/brand_guardian.py        Rule-based validation          │
│  ├── shared/collection_engine.py     Gemini 3 Pro planning          │
│  ├── shared/techpack_generator.py    Tech pack generation           │
│  ├── shared/ad_video_engine.py       Veo video generation           │
│  └── shared/pipeline_orchestrator.py Multi-step orchestrator        │
│                                                                     │
│  Infrastructure:                                                    │
│  ├── DO Spaces (boto3)               Media storage (images, PDFs)   │
│  ├── DO Managed PostgreSQL 17        Application database           │
│  └── DO Managed Valkey               Redis-compatible cache         │
└─────────────────────────────────────────────────────────────────────┘
                               │
┌──────────────────────────────┼──────────────────────────────────────┐
│                  VOICE COMPANION (FastAPI + ADK)                     │
│                       Port 8002                                     │
│                                                                     │
│  WebSocket: /ws/voice-companion/:session_id                         │
│  ADK Agent with 7 tools + runner.run_live() for bidi streaming      │
│  Model: gemini-live-2.5-flash-native-audio                          │
└─────────────────────────────────────────────────────────────────────┘
```

---

## AI Companion — Actions (Tools)

### Lux Design Companion (Text — DO Gradient Serverless Inference)

**Endpoint:** `POST /adk/design-companion`
**Agent file:** `trendsync-backend/services/main-backend/design_agent.py`
**Model:** `llama3.3-70b-instruct` (via DO Gradient at `inference.do-ai.run`)
**Framework:** OpenAI-compatible function calling loop
**API:** `POST https://inference.do-ai.run/v1/chat/completions` with `tools` array

| # | Tool | Description | Calls |
|---|------|-------------|-------|
| 1 | `analyze_product_image` | Visual analysis of the product image — gives specific feedback on colors, textures, silhouette, proportions | `genai.Client.models.generate_content()` with multimodal image input (Gemini Pro) |
| 2 | `edit_product_image` | Edit the current product image (color, structure, fabric, proportions) | `shared.image_generator.edit_product_image()` |
| 3 | `make_brand_compliant` | Auto-adjust product image to match brand color palette and guidelines | `shared.image_generator.edit_product_image()` + `shared.brand_guardian.validate_prompt()` |
| 4 | `fetch_trend_data` | Real-time fashion trend data via Gemini + Google Search grounding | `shared.trend_engine.fetch_trends()` |
| 5 | `validate_brand_compliance` | Check product compliance score against brand rules | `shared.brand_guardian.validate_prompt()` + `get_compliance_badge()` |
| 6 | `generate_image_variation` | Generate a completely new product image from a description | `shared.image_generator.generate_product_image()` |
| 7 | `save_design_signal` | Signal to the UI to persist the current design | Returns save action for frontend |

**Session state** (managed in-memory per request):
- `image_base64` — Current product image (stored in external `_IMAGE_STORE`, not in conversation history)
- `brand_style_json` — Brand style configuration
- `brand_id` — Brand UUID
- `product_context` — Product metadata (name, category, colors, materials)
- `text_history` — Last 6 conversation turns (text-only, images excluded to prevent context bloat)

### Voice Design Companion (Audio — ADK Agent)

**Endpoint:** `WS /ws/voice-companion/:session_id`
**Agent file:** `trendsync-backend/services/voice-companion/main.py`
**Model:** `gemini-live-2.5-flash-native-audio`
**Framework:** Google ADK (`Agent`, `Runner`, `LiveRequestQueue`)
**Runner mode:** `runner.run_live()` (bidirectional audio streaming)

| # | Tool | Description | Calls |
|---|------|-------------|-------|
| 1 | `adjust_design` | Edit product image via voice instruction | `POST /edit-image` (HTTP) |
| 2 | `fetch_trend_info` | Query real-time fashion trends | `POST /trends` (HTTP) |
| 3 | `validate_design` | Check brand compliance | `POST /validate` (HTTP) |
| 4 | `generate_variation` | Generate new product image | `POST /generate-image` (HTTP) |
| 5 | `generate_ad_video` | Start Veo video generation | `POST /generate-ad-video` (HTTP) |
| 6 | `navigate_to_page` | Navigate app to a specific page | Returns route for frontend |
| 7 | `start_collection_generation` | Start full collection generation | `POST /generate-collection` (HTTP) |

**Total unique actions across both companions: 14 (7 text + 7 voice)**

---

## Database Schema

**Database:** DigitalOcean Managed PostgreSQL 17
**Tables:** 11
**Auth:** Supabase Auth (JWT tokens, session management)

### 1. `user_profiles`

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid (PK) | User identifier |
| `full_name` | text | Display name |
| `avatar_url` | text | Profile picture URL |
| `role` | text | User role (admin, designer, viewer) |
| `created_at` | timestamptz | Account creation |
| `updated_at` | timestamptz | Last profile update |

### 2. `brands`

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid (PK) | Auto-generated |
| `user_id` | uuid (FK → user_profiles) | Brand owner |
| `name` | text | Brand name |
| `description` | text | Brand description |
| `logo_url` | text | Logo image URL |
| `created_at` | timestamptz | Creation date |
| `updated_at` | timestamptz | Last update |

### 3. `brand_styles`

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid (PK) | Auto-generated |
| `brand_id` | uuid (FK → brands) | Parent brand |
| `version` | integer | Style version number |
| `style_json` | jsonb | Full brand style config |
| `is_active` | boolean | Whether this version is active |
| `created_at` | timestamptz | Creation date |
| `created_by` | uuid (FK → user_profiles) | Who created this version |

**`style_json` structure:**
```json
{
  "colorPalette": [{ "id": "", "name": "", "hex": "", "designation": "primary|secondary|accent|neutral", "pantone": "" }],
  "cameraSettings": { "fovMin": 20, "fovMax": 80, "fovDefault": 50, "angleMin": 0, "angleMax": 90 },
  "lightingConfig": { "keyIntensity": 80, "fillIntensity": 40, "colorTemperature": 5000 },
  "logoRules": { "zone": "", "minSize": 0, "maxSize": 0, "allowedPositions": [], "exclusionZones": [] },
  "materialLibrary": [{ "id": "", "name": "", "category": "sustainable|premium|technical|standard" }],
  "negativePrompts": ["string"],
  "aspectRatios": [{ "width": 1, "height": 1, "name": "Square" }]
}
```

### 4. `collections`

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid (PK) | Auto-generated |
| `brand_id` | uuid (FK → brands) | Parent brand |
| `name` | text | Collection name (unique per brand) |
| `season` | text | Target season |
| `region` | text | Target market region |
| `target_demographic` | text | Target audience |
| `status` | text | draft / generating / validating / complete |
| `collection_plan_json` | jsonb | AI-generated collection plan |
| `trend_insights_json` | jsonb | Cached trend data used for generation |
| `created_at` / `updated_at` | timestamptz | Timestamps |

### 5. `collection_items`

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid (PK) | Auto-generated |
| `collection_id` | uuid (FK → collections) | Parent collection |
| `sku` | text | Product SKU |
| `name` | text | Product name |
| `category` | text | apparel / footwear / accessories |
| `subcategory` | text | Specific type (e.g., "blazer") |
| `design_story` | text | Creative narrative |
| `target_persona` | text | Target customer persona |
| `price_tier` | text | entry / mid / premium / luxury |
| `design_spec_json` | jsonb | Design specifications |
| `fibo_prompt_json` | jsonb | Structured image generation prompt |
| `brand_compliance_score` | float | 0-100 compliance score |
| `status` | text | planned / designing / generating / complete / failed |
| `image_url` | text | Product image URL (DO Spaces) |
| `video_url` | text | Product video URL (DO Spaces) |
| `created_at` / `updated_at` | timestamptz | Timestamps |

### 6. `trend_insights`

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid (PK) | Auto-generated |
| `collection_id` | uuid (FK → collections) | Associated collection |
| `region` | text | Geographic region |
| `season` | text | Fashion season |
| `demographic` | text | Target demographic |
| `insights_json` | jsonb | Full trend data (colors, silhouettes, materials, themes) |
| `source` | text | gemini / manual |
| `created_at` | timestamptz | When fetched |
| `expires_at` | timestamptz | Cache expiry |

### 7. `validations`

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid (PK) | Auto-generated |
| `collection_item_id` | uuid (FK → collection_items) | Validated item |
| `compliance_score` | float | 0-100 score |
| `violations` | jsonb | Array of violation objects |
| `auto_fixes_applied` | jsonb | Array of auto-fix records |
| `original_prompt_json` | jsonb | Prompt before fixes |
| `fixed_prompt_json` | jsonb | Prompt after auto-fixes |
| `validated_at` | timestamptz | Validation timestamp |

### 8. `generated_images`

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid (PK) | Auto-generated |
| `collection_item_id` | uuid (FK → collection_items) | Parent item |
| `image_url` | text | Image URL (DO Spaces) |
| `image_type` | text | product / lifestyle / detail / sketch |
| `view_angle` | text | front / back / side / three-quarter / top |
| `generation_params_json` | jsonb | FIBO prompt used for generation |
| `is_primary` | boolean | Whether this is the main image |
| `created_at` | timestamptz | Generation timestamp |

### 9. `tech_packs`

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid (PK) | Auto-generated |
| `collection_item_id` | uuid (FK → collection_items) | Parent item |
| `version` | integer | Tech pack version |
| `tech_pack_json` | jsonb | Full technical specifications |
| `pdf_url` | text | Exported PDF URL (DO Spaces) |
| `status` | text | draft / review / approved |
| `created_at` / `updated_at` | timestamptz | Timestamps |

### 10. `generation_jobs`

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid (PK) | Auto-generated |
| `user_id` | uuid (FK → user_profiles) | Job owner |
| `job_type` | text | trend_analysis / collection_design / image_generation / validation / tech_pack |
| `status` | text | queued / processing / complete / failed |
| `input_json` | jsonb | Job input parameters |
| `output_json` | jsonb | Job results |
| `error_message` | text | Error details if failed |
| `started_at` / `completed_at` / `created_at` | timestamptz | Timestamps |

### 11. `login_audit`

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid (PK) | Auto-generated |
| `user_id` | uuid (FK → user_profiles) | Who logged in |
| `user_agent` | text | Browser user agent |
| `login_method` | text | password / oauth |
| `success` | boolean | Whether login succeeded |
| `created_at` | timestamptz | Login timestamp |

---

## Entity Relationship Diagram

```
user_profiles
    │
    └──→ brands (1:N)
            │
            ├──→ brand_styles (1:N, versioned)
            │
            └──→ collections (1:N)
                    │
                    ├──→ trend_insights (1:N, cached)
                    │
                    └──→ collection_items (1:N)
                            │
                            ├──→ validations (1:N)
                            ├──→ generated_images (1:N)
                            └──→ tech_packs (1:N, versioned)

    generation_jobs (standalone, linked to user)
    login_audit (standalone, linked to user)
```

---

## Technology Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Frontend | React 18 + TypeScript + Vite | SPA with neumorphic pastel UI |
| Styling | Tailwind CSS | Utility-first CSS with custom neumorphic theme |
| State | React Context + useState | Auth context, local component state |
| Auth | Supabase Auth | JWT authentication, session management |
| Database | DO Managed PostgreSQL 17 | Application data (11 tables, JSONB) |
| Cache | DO Managed Valkey | Redis-compatible caching (24h TTL) |
| Object Storage | DO Spaces | Product images, videos, PDFs (CDN) |
| Backend | FastAPI (Python) | REST API gateway, background tasks |
| AI Design Agent | DO Gradient Serverless Inference | Lux companion (`llama3.3-70b-instruct`, 7 tools) |
| Image Generation | Gemini 3 Pro Image Preview | Product image generation and editing |
| Collection Planning | Gemini 3 Pro Preview | Complex multi-product collection planning |
| Trends | Gemini 2.5 Flash + Google Search | Real-time fashion trend data |
| Voice AI | Gemini Live 2.5 Flash Native Audio | Bidirectional voice streaming |
| Video Generation | Veo 3.1 | Multi-scene ad video generation |
| PDF Generation | Foxit Cloud API | DOCX → PDF conversion, compression, merging |
| Deployment | DO App Platform | Docker-based microservice deployment |

---

## AI Models Used

| Model | Platform | Use Case |
|-------|----------|----------|
| `llama3.3-70b-instruct` | **DO Gradient** | Design companion agent (Lux) — 7-tool function calling |
| `gemini-3-pro-preview` | Google | Collection planning, tech packs, storyboards |
| `gemini-3-pro-image-preview` | Google | Product image generation and editing |
| `gemini-2.5-flash` | Google | Trend engine, art-direction prompts |
| `gemini-live-2.5-flash-native-audio` | Google | Voice companion (bidirectional audio streaming) |
| `veo-3.1-generate-preview` | Google | Ad video generation (5-scene cinematic) |

---

## DigitalOcean Services

| Service | Purpose | Details |
|---------|---------|---------|
| **Gradient Serverless Inference** | AI design companion | OpenAI-compatible API, per-token billing, auto-scaling |
| **Managed PostgreSQL 17** | Application database | 11 tables, JSONB, FK constraints, cascade deletes, SSL |
| **Spaces Object Storage** | Media assets | S3-compatible, integrated CDN, public-read ACL |
| **Managed Valkey** | Caching | Redis-compatible, 24h TTL, zero code changes from Redis |
| **App Platform** | Deployment | 3 Docker services, env var management, auto-scaling |
