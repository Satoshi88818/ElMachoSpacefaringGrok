🚀 SpacefaringGrok v5.2

Multi-Agent AI Assistant for Space Intelligence FastAPI · PostgreSQL + pgvector · Grok LLM · SGP4 Orbital Mechanics

Table of Contents

Overview

System Requirements

Architecture

Configuration

Deployment

API Reference

Key Functionalities

Security Considerations

Known Limitations

Future Enhancements

Version History

Glossary

1. Overview

SpacefaringGrok is a production-grade, multi-agent AI assistant purpose-built for space intelligence. It combines large-language-model reasoning with real-time orbital mechanics, live launch data, semantic long-term memory, and multimodal input (text, images, audio) behind a single FastAPI HTTP interface.

Version 5.2 is the first release to fully eliminate module-level global state, enforce pre-flight budget checks, and ship an idempotent tone-adjustment pipeline.

PropertyValueCodenameSpacefaringGrok v5.2Codebase styleModularised monolith — three clear layers (core / agents / api)Primary AIGrok LLM (xAI) via internal grok_client wrapperDatabasePostgreSQL with pgvector extension for semantic searchFrameworkFastAPI (async throughout) + asyncpg connection poolingAuthJWT / OAuth2 — passlib bcrypt, python-jose 

2. System Requirements

2.1 Runtime Environment

RequirementDetailPython3.11 or higher (async/await, type unions)PostgreSQL14+ with pgvector extension installedOSLinux / macOS / Windows (WSL2 recommended)RAMMinimum 2 GB; 4 GB+ recommended under loadNetworkOutbound HTTPS to xAI API, CelesTrak, Launch Library 2 

2.2 Python Dependencies

PackageVersionPurposefastapi>=0.111Async HTTP frameworkuvicorn[standard]>=0.29ASGI serverasyncpg>=0.29Async PostgreSQL driverpgvector>=0.2Vector similarity searchpython-jose[cryptography]>=3.3JWT encoding/decodingpasslib[bcrypt]>=1.7Password hashing (CryptContext)aiohttp>=3.9Async HTTP client (TLE fetches, launch data)sgp4>=2.22SGP4/SDP4 orbital propagationtenacity>=8.2Retry logic with exponential back-off and jittercachetools>=5.3TTL in-memory embedding cacheslowapi>=0.1Rate limiting middlewareprometheus-client>=0.20Metrics exposurepydantic>=2.0Request/response validationpython-multipart>=0.0.9Multipart form upload handling 

2.3 External Services

Grok API (xAI) — all LLM inference: chat completions, JSON-mode intent detection, embeddings, image generation, and voice transcription.

CelesTrak — live Two-Line Element (TLE) data for SGP4 orbital propagation. Public HTTPS endpoint; no API key required.

Launch Library 2 — upcoming launch data from https://ll.thespacedevs.com. Public endpoint; no API key required.

Prometheus / Grafana (optional) — metrics emitted by the application; scrape endpoint at /metrics.

3. Architecture

3.1 Layer Overview

LayerPackageResponsibilityInfrastructurecore/Grok API client, circuit breaker, rate limiter, Prometheus metricsBusiness Logicagents/All AI agents plus orchestration façadeHTTP Interfaceapi/FastAPI routes, JWT auth, dependency injectionDatabasedb/Connection pool, token budget managementConfigurationconfig.pyCentralised settings loaded from environment variables 

3.2 Request Lifecycle

Every call to POST /process follows this sequence:

1. Rate limiter → 60 requests / minute per IP (slowapi) 2. JWT verification → extract user_id from Bearer token 3. Pre-flight budget → reject over-quota users before any LLM spend 4. Input sanitisation → image/audio type+size checks, thread_id cleaning 5. Intent detection → EmpathyMemoryAgent calls Grok JSON mode (sequential) 6. Memory retrieval → thread context (recency) + pgvector semantic search 7. Parallel fan-out → asyncio with per-agent timeouts: a) VisionAgent — image analysis if image_base64 present b) ReasoningAgent — deep contextual reasoning c) ActionAgent — command routing or LLM insight 8. Image generation → ImageGenAgent if trigger keywords detected (optional) 9. Commander synthesis → single Grok call assembles all agent outputs 10. commit_token_usage → total tokens written to DB (post-flight) 11. Response returned → includes follow-up suggestions and metrics 

3.3 Agent Descriptions

AgentModuleKey BehaviourEmpathyMemoryAgentagents/empathy.pyDetects mood/intent (JSON mode), stores embeddings, retrieves semantic history, adjusts response toneVisionAgentagents/vision.pySends base64 image + query to Grok vision endpoint; returns text analysisReasoningAgentagents/reasoning.pyDeep contextual reasoning over full thread + long-term memoryActionAgentagents/action.pyRoutes command intents to CommandModule; otherwise queries LLM directlyCommandModuleagents/commands.pySGP4 orbit prediction, live launch data, delta-v, exoplanet stubs, helpImageGenAgentagents/image_gen.pyKeyword-triggered image generation via Grok image endpoint ("Aurora")VoiceAgentagents/voice.pyTranscribes audio uploads (webm/wav/mp3) via Grok audio endpointGrok420Orchestratoragents/orchestrator.pyFans out to all agents, accumulates tokens, synthesises final responseSpacefaringGrokagents/orchestrator.pyTop-level façade stored on app.state; entry point for the API layer 

3.4 Directory Structure

spacefaring-grok/ ├── config.py # Centralised env-var config ├── main.py # FastAPI app factory + lifespan events ├── requirements.txt ├── .env.example │ ├── core/ │ ├── circuit_breaker.py # Prevents cascading API failures │ ├── grok_client.py # Async wrapper for all Grok endpoints │ ├── metrics.py # Prometheus counters + histograms │ └── rate_limit.py # slowapi rate limiter instance │ ├── db/ │ ├── pool.py # asyncpg pool creation + get_db_pool dependency │ └── budget.py # preflight_budget_check + commit_token_usage │ ├── agents/ │ ├── empathy.py # EmpathyMemoryAgent │ ├── vision.py # VisionAgent │ ├── reasoning.py # ReasoningAgent │ ├── action.py # ActionAgent │ ├── commands.py # CommandModule (SGP4, launches, etc.) │ ├── image_gen.py # ImageGenAgent ("Aurora") │ ├── voice.py # VoiceAgent │ └── orchestrator.py # Grok420Orchestrator + SpacefaringGrok façade │ └── api/ ├── dependencies.py # get_grok + verify_token FastAPI deps ├── auth.py # /register + /token endpoints └── process.py # /process endpoint 

3.5 Core Infrastructure

Circuit Breaker (core/circuit_breaker.py) — stops sending requests to the Grok API after a threshold of consecutive failures, preventing cascading degradation.

Rate Limiter (core/rate_limit.py) — slowapi-based; 60 requests/minute per IP on /process.

Metrics (core/metrics.py) — Prometheus counters and histograms for request count, endpoint latency, and per-agent latency. Labelled by endpoint and agent name.

Grok Client (core/grok_client.py) — single async wrapper for all Grok endpoints (chat, embeddings, image generation, audio transcription). All methods return (content, token_count) tuples so the orchestrator can accumulate spend in one place.

4. Configuration

All settings are loaded from environment variables. Copy .env.example to .env and populate the values before starting the application.

4.1 Required Variables

VariableExamplePurposeGROK_API_KEYxai-xxxxxxxxAPI key for xAI Grok endpointsDATABASE_URLpostgresql://user:pw@host/dbasyncpg connection stringJWT_SECRETa-long-random-stringHMAC secret for JWT signing 

4.2 Optional Variables (with defaults)

VariableDefaultPurposeGROK_MODELgrok-3Default chat / reasoning modelIMAGE_MODELgrok-2-imageImage generation modelJWT_ALGORITHMHS256JWT signing algorithmJWT_EXPIRE_HOURS24Token lifetime in hoursMAX_TOKENS_PER_RESPONSE1000Cap on synthesis output tokensVISION_TIMEOUT15.0Per-agent timeout in secondsREASONING_TIMEOUT20.0Per-agent timeout in secondsACTION_TIMEOUT12.0Per-agent timeout in secondsMAX_CONTEXT_MESSAGES10Recent messages kept in thread contextMAX_CONTEXT_WORDS600Word-count cap on combined context stringWARP_QUIP_ENABLEDtrueEnable / disable the random warp quip suffixWARP_QUIP_PROBABILITY0.1Probability (0–1) of appending the quipDEFAULT_CONFIDENCE0.5Intent confidence fallback when JSON parse fails 

5. Deployment

5.1 Local Development

# 1. Clone and create virtual environment git clone <repo-url> && cd spacefaring-grok python -m venv .venv && source .venv/bin/activate # 2. Install dependencies pip install -r requirements.txt # 3. Configure environment cp .env.example .env # Edit .env — fill in GROK_API_KEY, DATABASE_URL, JWT_SECRET # 4. Initialise the database psql $DATABASE_URL -f db/schema.sql # 5. Start the development server uvicorn main:app --reload --host 0.0.0.0 --port 8000 

5.2 Database Schema

Three tables are required. The embedding column needs the pgvector extension.

CREATE EXTENSION IF NOT EXISTS vector; CREATE TABLE users ( id UUID PRIMARY KEY DEFAULT gen_random_uuid(), username TEXT UNIQUE NOT NULL, hashed_password TEXT NOT NULL ); CREATE TABLE user_history ( id BIGSERIAL PRIMARY KEY, user_id TEXT NOT NULL, thread_id TEXT NOT NULL, text TEXT NOT NULL, sentiment FLOAT, embedding VECTOR(1536), timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW() ); CREATE INDEX ON user_history USING ivfflat (embedding vector_cosine_ops); CREATE TABLE user_budgets ( user_id TEXT PRIMARY KEY, tokens_used BIGINT NOT NULL DEFAULT 0, tokens_limit BIGINT NOT NULL DEFAULT 1000000, updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW() ); 

5.3 Production

Use Gunicorn with Uvicorn workers behind a reverse proxy (nginx / Caddy):

gunicorn main:app \ -k uvicorn.workers.UvicornWorker \ -w 4 \ --bind 0.0.0.0:8000 \ --access-logfile - \ --error-logfile - 

Key production considerations:

Worker count — set to (2 × CPU cores) + 1. Each worker maintains its own asyncpg pool.

DB pool size — tune MAX_POOL_SIZE in db/pool.py; total connections must stay below PostgreSQL max_connections.

TLS — terminate at the reverse proxy, not inside the application.

Secrets — use a secrets manager (AWS Secrets Manager, HashiCorp Vault) rather than .env files.

Metrics — expose /metrics on an internal port only; scrape with Prometheus.

5.4 Docker

FROM python:3.12-slim WORKDIR /app COPY requirements.txt . RUN pip install --no-cache-dir -r requirements.txt COPY . . CMD ["gunicorn", "main:app", \ "-k", "uvicorn.workers.UvicornWorker", \ "-w", "4", \ "--bind", "0.0.0.0:8000"] # Build and run docker build -t spacefaring-grok . docker run -p 8000:8000 --env-file .env spacefaring-grok 

6. API Reference

6.1 Authentication

POST /register

Register a new user account.

Request body (JSON):

{ "username": "cosmonaut", "password": "s3cur3p@ss" } 

Response: 201 Created

{ "message": "User registered successfully." } 

POST /token

Authenticate and receive a JWT Bearer token.

Request body (JSON):

{ "username": "cosmonaut", "password": "s3cur3p@ss" } 

Response: 200 OK

{ "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...", "token_type": "bearer", "user_id": "cosmonaut" } 

6.2 Core Endpoint

POST /process

Submit a query through the full multi-agent pipeline.

Authentication: Authorization: Bearer <token>

Request — multipart/form-data:

FieldTypeRequiredDescriptionquerystringNoText query (defaults to "Hello")imagefileNoJPEG / PNG / WebP, max 10 MBaudiofileNoWebM / WAV / MP3, max 10 MBthread_idstringNoConversation thread ID (auto-generated if omitted) 

Response: 200 OK

{ "response": "Thrusters firing — The ISS is currently at X=3204.1 Y=-5612.3 Z=1847.9 km.", "image_url": null, "thread_id": "th-abc123xyz", "follow_ups": [ "Orbit 25544 30", "Next launch", "Generate image of TRAPPIST-1e", "Status report" ], "metrics": { "confidence": 0.92, "agents": 6, "model": "grok-3", "total_time_s": 3.847, "tokens_used": 1243 } } 

Error responses:

CodeCondition401Missing or invalid JWT token402 / 429User has exceeded their token budget413Uploaded file exceeds 10 MB415Unsupported file type503Application still starting up (app.state.grok not yet set) 

7. Key Functionalities

7.1 Multi-Agent Orchestration

Grok420Orchestrator fans out to six agents in a controlled sequence. Intent detection runs first because downstream agents depend on mood and intent classification. Vision, reasoning, and action then run concurrently under individual asyncio timeouts, so a slow or failing agent cannot block the response. All token costs are accumulated into a single counter and written to the database in one call at the end.

The _safe_agent wrapper provides a consistent fallback contract: any agent that times out or raises an unexpected exception returns a (fallback_string, 0) tuple rather than propagating the error upward.

7.2 Semantic Long-Term Memory

Every user message is embedded and stored in PostgreSQL via pgvector. On each request the system retrieves two memory streams and merges them:

Thread context — the most recent MAX_CONTEXT_MESSAGES messages for the current thread, ordered by recency.

Long-term semantic memory — the top-6 most semantically similar historical messages across all threads, using the pgvector <=> cosine distance operator.

The merged string is then word-count-truncated to MAX_CONTEXT_WORDS before being passed to the reasoning agent, preventing context-window overflow without discarding the most relevant history. Embeddings for identical strings are cached in a 5-minute TTL cachetools.TTLCache to avoid redundant API calls.

7.3 Real-Time Orbital Mechanics

CommandModule.plot_orbit fetches live Two-Line Element (TLE) data from CelesTrak for any NORAD catalogue number and propagates the satellite's position forward by a configurable number of minutes using the sgp4 library (Satrec.twoline2rv + sat.sgp4). The implementation includes:

TLE line-count validation before parsing — prevents malformed data from crashing the propagator.

SGP4 error code checking — non-zero error codes are surfaced as descriptive messages.

Tenacity retry with exponential back-off and jitter (2 attempts, 0.5–4 s wait) wrapping the CelesTrak HTTP fetch.

Automatic fallback to NORAD 25544 (ISS) if the requested satellite ID fails.

7.4 Live Launch Data

Upcoming launch information is fetched in real time from the Launch Library 2 public API (https://ll.thespacedevs.com/2.2.0/launch/upcoming/). The response is parsed to extract the mission name, agency, launch pad, and NET (no-earlier-than) timestamp. A human-readable countdown (T-Xh Ym) is computed from the NET, with graceful fallback to a static message on network or parse errors.

7.5 Mood-Aware Responses

Intent detection classifies each message along four dimensions using Grok in JSON mode:

FieldValuesmoodstressed / neutral / excitedintentcommand / seek_help / predict / seek_insight / anomalysentimentfloat −1.0 to +1.0confidencefloat 0.0 to 1.0 

The synthesised response is prefixed with a mood-appropriate tone marker (adjust_tone), and the synthesis prompt instructs the commander agent to apply cosmic humour when mood is excited. adjust_tone is idempotent — it checks for the prefix before prepending, so calling it twice on the same string has no effect.

7.6 Multimodal Input

Images — JPEG, PNG, and WebP files up to 10 MB are accepted, base64-encoded, and forwarded to the Grok vision endpoint alongside the text query.

Audio — WebM, WAV, and MP3 uploads are transcribed by VoiceAgent. The transcript replaces the text query if transcription succeeds; the original query is used as fallback.

7.7 Token Budget Enforcement

Each user has a token budget row in user_budgets. preflight_budget_check runs before any LLM call is made, returning an error response immediately if the user is over quota. This prevents costs from accruing for over-budget users. After the pipeline completes, commit_token_usage writes the actual token spend to the database in a single update.

7.8 Image Generation ("Aurora")

ImageGenAgent monitors the query for trigger keywords (generate image, show, visualize, aurora) and calls the Grok image generation endpoint if matched. The prompt is appended with , cinematic space style, ultra-realistic, high detail before submission. Generation runs after the main agent fan-out and does not contribute to the tracked token total.

8. Security Considerations

8.1 Authentication & Authorisation

JWT tokens are signed with HMAC-SHA256; expiry is configurable via JWT_EXPIRE_HOURS (default 24 h).

Passwords are hashed with bcrypt via passlib.CryptContext.

Username sanitisation (re.sub(r"[^a-zA-Z0-9_-]", "", ...)) strips non-safe characters before the value is used as user_id in the database.

All endpoints except /register and /token require a valid Bearer token.

8.2 Input Validation

Image uploads: content-type whitelist (image/jpeg, image/png, image/webp) + 10 MB hard cap.

Audio uploads: content-type whitelist (audio/webm, audio/wav, audio/mpeg) + 10 MB hard cap.

thread_id: regex sanitisation and 64-character truncation before use.

Rate limiting: 60 requests/minute per IP enforced before any business logic runs.

8.3 Known Security Gaps

AreaRiskPrompt injectionthread_context and agent outputs are injected into the synthesis prompt without sanitisation. A crafted user message could manipulate the commander agent's behaviour.JWT secret rotationRotating JWT_SECRET immediately invalidates all active sessions. A versioned key scheme is recommended for production.No refresh tokensSessions expire abruptly; users must re-authenticate after JWT_EXPIRE_HOURS. 

9. Known Limitations

AreaLimitationImpactVision MIME typeimage/jpeg is hardcoded in the base64 data URI regardless of actual upload typeMay fail with strict vision API implementations for PNG/WebP inputsImage trigger words"show" alone triggers image generation — e.g. "show ISS orbit" fires Aurora unnecessarilyUnexpected API calls and added latencyDuplicate DB connectionslogin_for_access_token acquires the pool twice sequentially for password check and budget insertMinor inefficiency; holds an extra connection during loginHardcoded command stubsDelta-v and TRAPPIST-1e responses are static stringsNo real scientific computation for these commandsNo test suiteNo unit or integration tests visible in the codebaseRegression risk; retry logic and budget enforcement paths untestedPrompt injectionRaw user memory injected into synthesis prompt without sanitisationPotential for adversarial prompt manipulationBudget race conditionToken budget is checked pre-flight but not locked — no SELECT FOR UPDATEUsers can exceed budget slightly under high-concurrency burstNo streamingAll responses are buffered; large synthesis outputs cause noticeable latencyPoor UX for long responsesAudio read orderingaudio.read() is called before audio.seek(0) — bytes may be consumed before transcriptionTranscription could silently receive empty bytes in some execution pathsNo refresh tokensJWTs cannot be refreshed without re-authenticatingSessions expire abruptly after JWT_EXPIRE_HOURS 

10. Future Enhancements

Short Term

Fix VisionAgent MIME type — detect actual content type from the upload and set the correct data URI prefix (image/png, image/webp, etc.).

Refine image trigger logic — replace substring matching with intent-based routing; only call Aurora when intent == "visualize".

Consolidate auth DB connections — merge the two pool.acquire() calls in login_for_access_token into a single transaction.

Add a test suite — unit tests for CommandModule (mock aiohttp), EmpathyMemoryAgent (mock DB), and budget enforcement; integration tests for /process.

Fix audio consumption order — call audio.seek(0) before audio.read() to ensure the transcription endpoint always receives full bytes.

Medium Term

Streaming responses — implement SSE or WebSocket streaming for the synthesis step to reduce perceived latency on long outputs.

Prompt injection mitigation — sanitise agent outputs before injecting into the synthesis prompt; consider a dedicated content-safety classifier stage.

Real scientific commands — replace hardcoded delta-v and exoplanet stubs with calls to NASA APIs (JPL Horizons, NASA Exoplanet Archive).

Budget locking — use SELECT ... FOR UPDATE or a Redis atomic counter to prevent budget race conditions under concurrency.

Distributed rate limiting — replace in-process slowapi with a Redis-backed limit to support multi-instance deployments.

Long Term

Multi-model routing — allow the orchestrator to select between LLM providers (Grok, OpenAI, Gemini) based on task type and cost profile.

Agent plugin system — define a formal BaseAgent interface and allow new agents to be registered without modifying the orchestrator.

Real-time telemetry — subscribe to live satellite telemetry streams (NOAA, amateur radio networks) for richer orbital data beyond TLE propagation.

WebSocket interface — replace REST polling with a persistent WebSocket connection for an interactive mission-control-style UX.

Fine-tuned space model — fine-tune a domain-specific variant of the base Grok model on astrophysics, mission logs, and space operations corpora.

Multi-tenant isolation — namespace DB tables per organisation; add role-based access control for team deployments.

11. Version History

VersionSummaryStatusv5.2Eliminated module-level globals; pre-flight budget check; idempotent adjust_tone; live launch API replacing hardcoded string; CryptContext typo fix (was CryptoContext); word-count context truncation; except Exception replacing bare except:; WARP_QUIP_ENABLED flagCurrentv5.1thread_id sanitisation; circuit breaker; Prometheus metrics; embedding TTL cacheSupersededv5.0Initial modularised monolith; parallel agent fan-out architectureSuperseded 

12. Glossary

TermDefinitionSGP4Simplified General Perturbations model 4 — standard algorithm for predicting low-Earth-orbit satellite positions from TLE dataTLETwo-Line Element set — standardised format for satellite orbital parameters, published by CelesTrak and Space-TrackpgvectorPostgreSQL extension adding a vector column type and cosine / L2 distance operators for semantic similarity searchNORAD IDUnique catalogue number assigned by NORAD to every tracked Earth-orbiting objectNETNo Earlier Than — the earliest possible launch time published by launch providersapp.stateFastAPI mechanism for storing application-wide objects that survive across requests without relying on module-level globalsFan-outDispatching multiple coroutines concurrently with asyncio.gather and collecting all results before proceedingCircuit BreakerDesign pattern that stops forwarding requests to a failing dependency after a threshold of errors, preventing cascading failuresWarp QuipConfig-controlled Easter egg — a random-probability suffix appended to responses for cosmic flavourCommander AgentThe final synthesis step in the orchestrator; receives all agent outputs and produces the user-facing response 

SpacefaringGrok v5.2 — Space bends time, ready to warp? 🚀

