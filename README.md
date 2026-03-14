# TrendSync Brand Factory

AI-powered fashion design platform that turns real-time trend intelligence into complete, brand-compliant collections — with AI-generated product imagery, manufacturing-ready tech pack PDFs, ad videos, and an AI design companion — powered by **DigitalOcean Gradient AI**, **DO Managed PostgreSQL**, **DO Spaces**, and **DO Managed Valkey**.

![trendsync Image](https://raw.githubusercontent.com/EimisPachecoHacks/IMAGE-REPO/main/TrendSync1.png)
![trendsync Image](https://raw.githubusercontent.com/EimisPachecoHacks/IMAGE-REPO/main/TrendSync2.png)

## What It Does

TrendSync lets fashion designers go from "what's trending?" to a complete collection with product images, tech packs, and ad videos — in a single session. No manual trend research, no separate image tools, no disconnected workflows.

## DigitalOcean Gradient AI Integration

DigitalOcean's full-stack AI platform powers the core infrastructure:

| DO Service | What It Powers |
|-----------|---------------|
| **Gradient Serverless Inference** | AI design companion agent ("Lux") — `llama3.3-70b-instruct` with 7-tool function calling via OpenAI-compatible API at `inference.do-ai.run` |
| **Managed PostgreSQL 17** | Application database — 11 tables with JSONB for brand styles, collections, trend data, tech packs, and compliance scores |
| **Spaces Object Storage** | All generated media — product images, ad videos, PDFs — served via integrated CDN |
| **Managed Valkey** | Redis-compatible caching with 24h TTL — deduplicates trend queries and image generation requests |
| **App Platform** | Production deployment of 3 FastAPI microservices from Docker containers |

**Why DigitalOcean was ideal:** Fashion design requires a full-stack AI platform, not just an inference endpoint. We needed serverless AI with function calling, managed databases with JSONB for complex style configs, S3-compatible storage with CDN for generated media, and in-memory caching — all under one roof. Gradient's OpenAI-compatible API meant zero vendor lock-in, and Valkey's Redis compatibility meant zero code changes for the caching layer.

## App Sections & Tabs

### Dashboard
Overview page with quick actions — "Create New Collection" and "Try Brand Guardian." Shows the platform workflow and trend source options.

### Trend Intel
Analyze real-time fashion trends across 6 global markets. Two modes:
- **Regional Trends** — Select region (Tokyo, NYC, London, Paris, Seoul, LA), season, and demographic. Returns trending colors, silhouettes, materials, and themes sourced from live web data.
- **Celebrity Trends** — Fetches the top 10 celebrity fashion influencers for your demographic, then generates trend insights based on their signature styles.

### Collections
Create AI-generated fashion collections. Two modes:
- **Regional & Seasonal** — Pick a region, season, demographic, product categories, and count. The platform analyzes trends, plans a collection with deep AI reasoning, then generates product images and validates each against brand guidelines.
- **Celebrity-Inspired** — Same flow but seeded from celebrity trend data instead of regional data.

Each generated product appears in a gallery with compliance scores and can be clicked for detail.

### Product Detail (6 tabs per product)
- **Overview** — Product image, name, compliance score, design story, color palette
- **FIBO JSON** — Full image generation prompt with all parameters
- **Validation** — Brand compliance breakdown (color, camera, lighting, negative prompts)
- **Tech Pack** — AI-generated manufacturing specs (materials, measurements, construction). Download as PDF via Foxit, or email directly to manufacturers
- **Adjust Design** — Chat with "Lux" (the AI design companion on DO Gradient) to edit images, check trends, validate compliance, or generate variations
- **Ad Video** — Generate a 5-scene cinematic ad video from the product

### Brand Style Editor
Configure your brand's visual rules — color palette (hex + Pantone), camera settings (FOV, angle ranges), lighting (color temperature, intensity), material library, logo placement rules, negative prompts, and aspect ratios. All stored as JSONB in DO Managed PostgreSQL.

### Brand Guardian
Real-time compliance scoring. Tests any design against your brand rules:
```
score = 100 - (critical x 25) - (warning x 10) - (suggestion x 3)
```

### Collection Library
Browse and manage saved collections.

### Voice Companion
Hands-free design assistant via bidirectional audio streaming. Say "make the jacket terracotta" or "what's trending in Seoul?" — same tools as the typing companion.

## Architecture

```
React (Vercel) → DO App Platform: Main Backend (:8080)
                    ├── DO Gradient Inference (Llama 3.3 70B) ← Design Companion
                    ├── Gemini 3 Pro / Flash / Image ← Collection + Image Gen
                    ├── Veo 3.1 ← Video Generation
                    ├── Foxit Cloud ← PDF Generation
                    ├── DO App Platform: Video Service (:8080)
                    └── DO App Platform: Voice Service (:8080)
                          ↕
          DO Managed PostgreSQL 17 (11 tables)
          DO Managed Valkey (Redis-compatible cache)
          DO Spaces Object Storage (images, videos, PDFs)
```

## Prerequisites

- **Node.js** >= 18
- **Python** >= 3.11
- **DigitalOcean account** with:
  - Spaces Object Storage bucket
  - Managed PostgreSQL cluster
  - Gradient AI model access key
  - (Optional) Managed Valkey cluster
- **Google Cloud** credentials (for Gemini image gen, Veo video, voice companion)
- **Foxit Cloud** API credentials (for PDF generation)

## Installation

### 1. Clone the repo

```bash
git clone https://github.com/EimisPachecoHacks/TrendSync-Brand-Factory-Live.git
cd TrendSync-Brand-Factory-Live
```

### 2. Frontend setup

```bash
npm install
```

Create `.env` at the project root:
```bash
VITE_SUPABASE_URL=https://your-project.supabase.co
VITE_SUPABASE_ANON_KEY=your-anon-key
VITE_API_BASE_URL=http://localhost:8000
```

### 3. Backend setup

```bash
cd trendsync-backend
pip install -r requirements.txt
```

Create `trendsync-backend/.env`:
```bash
# DigitalOcean Spaces
DO_SPACES_KEY=your-spaces-access-key
DO_SPACES_SECRET=your-spaces-secret-key
DO_SPACES_BUCKET=trendsync-media
DO_SPACES_REGION=sfo3

# DigitalOcean Gradient AI
DO_MODEL_ACCESS_KEY=your-gradient-model-key
DO_CHAT_MODEL=llama3.3-70b-instruct

# Google Cloud (for image gen, video, voice, trends)
GOOGLE_APPLICATION_CREDENTIALS=/path/to/service-account.json
GOOGLE_CLOUD_PROJECT=your-project-id
GOOGLE_CLOUD_LOCATION=global
GOOGLE_GENAI_USE_VERTEXAI=TRUE

# Gemini models
GEMINI_PRO_MODEL=gemini-3-pro-preview
GEMINI_FLASH_MODEL=gemini-2.5-flash
GEMINI_FLASH_IMAGE_MODEL=gemini-3-pro-image-preview
VEO_MODEL=veo-3.1-generate-preview
VOICE_MODEL=gemini-live-2.5-flash-native-audio

# DO Managed Valkey / Redis cache
REDIS_URL=rediss://default:password@your-valkey-host:25061

# Foxit PDF API
FOXIT_CLIENT_ID=your-foxit-id
FOXIT_CLIENT_SECRET=your-foxit-secret
```

### 4. Database setup

Create a DO Managed PostgreSQL cluster, then run the schema:

```bash
psql "postgresql://doadmin:PASSWORD@your-db-host:25060/defaultdb?sslmode=require" \
  -f trendsync-backend/do-postgres-migration.sql
```

### 5. Start the services

```bash
# Terminal 1 — Frontend
npm run dev

# Terminal 2 — Main backend
cd trendsync-backend
uvicorn services.main-backend.main:app --host 0.0.0.0 --port 8000 --reload

# Terminal 3 — Video service (optional)
cd trendsync-backend
uvicorn services.video-gen-service.main:app --host 0.0.0.0 --port 8001 --reload

# Terminal 4 — Voice companion (optional)
cd trendsync-backend
python -m services.voice-companion.main
```

The app will be available at `http://localhost:5173`.

## Testing the App

### Quick Test: Trend Analysis
1. Open the app → click **Trend Intel** in the sidebar
2. Select "Tokyo" region, "Spring 2026" season, "Gen-Z" demographic
3. Click **Analyze Trends** — you'll see trending colors, silhouettes, materials, and themes

### Full Test: Generate a Collection
1. Go to **Collections** → **Regional Trends** tab
2. Set region, season, demographic, categories, and product count (4 recommended)
3. Click **Generate Collection**
4. Watch the pipeline: fetching trends → planning collection → generating images
5. Click any product → explore the 6 tabs (overview, FIBO, validation, tech pack, design, video)

### Test: Celebrity-Inspired Collection
1. Go to **Collections** → **Celebrity Trends** tab
2. Select demographic → view fetched celebrities
3. Click **Generate Collection** — collection will be inspired by celebrity fashion

### Test: AI Design Companion (DO Gradient)
1. Generate a collection, click a product, go to **Adjust Design** tab
2. Type: "Change the color to terracotta" → Lux calls the edit tool on DO Gradient
3. Type: "What's trending in Seoul?" → Lux fetches live trend data
4. Type: "Check brand compliance" → Lux validates against your brand rules

### Test: Tech Pack PDF
1. Click a product → **Tech Pack** tab → **Generate Tech Pack**
2. Click **Download PDF** — Foxit converts the specs to a professional PDF

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | React 18, TypeScript, Vite, Tailwind CSS |
| Backend | Python, FastAPI (3 microservices) |
| AI Agent | DO Gradient Serverless Inference (Llama 3.3 70B) |
| Image Gen | Gemini 3 Pro Image Preview |
| Video Gen | Veo 3.1 |
| Voice | Gemini Live 2.5 Flash Native Audio |
| Trends | Gemini 2.5 Flash + Google Search Grounding |
| Database | DO Managed PostgreSQL 17 |
| Storage | DO Spaces Object Storage (S3-compatible, CDN) |
| Cache | DO Managed Valkey (Redis-compatible) |
| Deployment | DO App Platform (Docker) |
| PDFs | Foxit Cloud API |
| Auth | Supabase Auth |

## License

MIT — see [LICENSE](LICENSE)
