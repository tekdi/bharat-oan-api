# Glossary

Domain and technical terms that appear in the `bharat-oan-api` source. Each entry names where the term is evidenced. Expansions are given only when the code, prompts or docs state them.

Related: [Business Overview](../business/business-overview.md) | [Third-party integrations](../integrations/third-party-integrations.md) | [AI/LLM integration](../integrations/ai-llm.md) | [References](references.md) | [Repository Map](../repository-map.md)

## Platform and organisation

| Term | Meaning | Where used |
|------|---------|------------|
| OAN / OpenAgriNet | OAN is the short form used in the repository name `bharat-oan-api`; OpenAgriNet is the initiative named in the MIT license, which credits "Centre for Open Societal Systems (COSS), as part of the OpenAgriNet initiative" | `LICENSE`, `README.md`, git remotes |
| Bharat Vistaar / BharatVistaar | The farmer-facing product this API serves. Default channel name `BharatVistaar`, Langfuse trace name `bharat-vistaar-chat`, app name "Bharatvistaar AI API" | `app/config.py`, `app/auth/jwt_auth.py` (`DEFAULT_AUTH_CHANNEL`), `app/services/chat.py` |
| Vistaar network | Beckn-based network of agricultural service providers that most tools call through a BAP endpoint | `agents/tools/*.py`, [Third-party integrations](../integrations/third-party-integrations.md) |
| Agrinet / agrinet agent | The main conversational LLM agent ("Vistaar Agent"), its prompts `assets/prompts/agrinet_<lang>.md`, and the model pair `AGRINET_MODEL` | `agents/agrinet.py`, `agents/models.py` |
| Moderation agent | LLM classifier run before the agrinet agent. Categories: `valid_agricultural`, `invalid_non_agricultural`, `invalid_external_reference`, `invalid_compound_mixed`, `invalid_language`, `unsafe_illegal`, `political_controversial`, `role_obfuscation` | `agents/moderation.py` |
| Channel | Calling client identity carried in the JWT (`channel`, `client_code`); drives telemetry and Langfuse tags | `app/auth/jwt_auth.py`, `app/routers/token.py` |
| Client code | Per-partner identifier selecting API key / Play Integrity configuration; normalised to upper case with `_` | `app/routers/token.py` |
| QID | Question ID for correlating a chat turn across the `X-QID` header, telemetry and Langfuse | `app/routers/chat.py`, `app/services/chat_turn_map.py` |
| Session ID | Conversation identifier used as the Redis history key | `app/routers/chat_query_utils.py`, `app/utils.py` |
| FarmerContext | Pydantic dependency passed to the agent with query, language, coordinates and NPSS state | `agents/deps.py` |
| Guest token | JWT issued by `POST /api/token` for unauthenticated clients, carrying fingerprint context | `app/routers/token.py`, `tests/test_backend_owned_telemetry.py` |

## Beckn and network terms

| Term | Meaning | Where used |
|------|---------|------------|
| Beckn | Open protocol used for `/search`, `/init`, `/status` style calls to providers | `agents/tools/gfr.py`, `mandi.py`, `pmfby_scheme_status.py`, ... |
| BAP / BPP | Beckn Application Platform (this side, `BAP_ID`, `BAP_URI`, `BAP_ENDPOINT`) and Beckn Provider Platform (`BPP_ID`, `BPP_URI`) | `agents/tools/*.py` |
| N-N pattern | Network-to-network cross-network search ("BH to MH") | `agents/tools/maha_vistaar.py` docstring |
| MahaVistaar | The Maharashtra counterpart network; tool `call_maha_vistaar_network` handles NDKSP drip irrigation, farm pond lining and AIF queries | `agents/tools/maha_vistaar.py` |
| NDKSP | Scheme-code prefix (`ndksp-drip-irrigation`, `ndksp-farm-pond-lining`) used with MahaVistaar; expansion is not given in code | `agents/tools/maha_vistaar.py` |
| AmulVistaar / Amul union | Cross-network search for Amul union schemes, Beckn domain `schemes:amul-union` | `agents/tools/amul_vistaar.py` |
| Master catalog | Postgres-owned catalogue of schemes published by the docs-pipeline as a Redis snapshot `master-catalog:<tier>:snapshot`; tier is `dev` or `live` | `helpers/master_catalog.py` |
| docs-pipeline | External ingestion pipeline that writes the master catalog and Qdrant data | `helpers/master_catalog.py` docstring |

## Schemes and services

| Term | Meaning (as stated in code, prompts, or tool labels) | Where used |
|------|---------------------------------------------------|------------|
| PM-KISAN | Pradhan Mantri Kisan Samman Nidhi scheme; status check and grievance flows with OTP | `agents/tools/pmkisan_scheme_status.py`, `pmkisan_grievance.py`, `scheme_info.py` |
| PMFBY | Pradhan Mantri Fasal Bima Yojana (crop insurance); status and grievance flows with OTP | `agents/tools/pmfby_scheme_status.py`, `pmfby_grievance.py` |
| SHC | Soil Health Card scheme; status tool renders an HTML card cached in Redis and served at `/api/file/{hash}` | `agents/tools/shc_scheme_status.py`, `app/routers/file.py` |
| SMAM | Sub-Mission on Agricultural Mechanization; application/beneficiary status | `agents/tools/smam_scheme_status.py` |
| SATHI | Seed Authentication, Traceability & Holistic Inventory; seed availability and dealer lookup | `agents/tools/sathi_seed.py` |
| AIF | Agriculture Infrastructure Fund; OTP-verified loan and grievance status | `agents/tools/aif.py` |
| GFR | Government Fertilizer Recommendation; crop registry and fertilizer advice | `agents/tools/gfr.py` |
| KCC | Kisan Credit Card scheme | `agents/tools/scheme_info.py` (`_SCHEME_LABELS`) |
| PMKSY | Pradhan Mantri Krishi Sinchayee Yojana | `agents/tools/scheme_info.py` |
| PM-ASHA (`pmasha`) | Pradhan Mantri Annadata Aay Sanrakshan Abhiyan | `agents/tools/scheme_info.py` |
| PDMC | Per Drop More Crop scheme | `agents/tools/scheme_info.py` |
| PKVY | Paramparagat Krishi Vikas Yojana | `agents/tools/scheme_info.py` |
| NFSM | National Food Security Mission | `agents/tools/scheme_info.py` |
| RAD | Rainfed Area Development | `agents/tools/scheme_info.py` |
| FFS | Labelled "Framework for Fertilizer Sales" in the scheme label table | `agents/tools/scheme_info.py` |
| NBM / NBHM | National Bamboo Mission / National Beekeeping & Honey Mission | `agents/tools/scheme_info.py` |
| MIF | Micro Irrigation Fund (named in the moderation prompt; vector-indexed scheme per config comment) | `assets/prompts/moderation_system.md`, `app/config.py` |
| PM-KMY / PMKMY | Kisan Maandhan (per moderation prompt) | `assets/prompts/moderation_system.md` |
| e-NAM | Electronic National Agriculture Market; training video retrieval | `assets/prompts/moderation_system.md`, `helpers/video_qdrant_search.py` |
| RWBCIS, CDP, NMEO | Further scheme acronyms the moderation prompt treats as valid farmer queries (Restructured Weather Based Crop Insurance Scheme, Crop Diversification Programme, NMEO / NMEO-OS) | `assets/prompts/moderation_system.md` |
| NPSS | National Pest Surveillance System; crop image analysis API at `NPSS_BASE_URL`, with attribution/translation post-processing | `agents/tools/npss.py`, `app/core/npss_geocodes.py`, `app/services/npss_response.py` |
| KVK | Krishi Vigyan Kendra; NPSS location fallback is the KVK in Ujwa, Delhi when coordinates are unavailable | `agents/tools/npss.py`, `app/services/chat.py` |
| Mandi | Agricultural market; `get_mandi_prices` fetches nearby mandi prices via the Vistaar API | `agents/tools/mandi.py` |
| AGMKT | Source of the commodity codes used by `search_commodity` | `agents/tools/commodity.py`, `assets/commodity_codes.json` |
| IMD / Mausamgram | India Meteorological Department weather source; the Beckn item is named "Weather-Forecast-Mausamgram" | `agents/tools/weather.py` |
| Kharif / Rabi / Zaid | Indian cropping seasons computed by `get_crop_season` (Jun-Sep, Oct-Feb, Mar-May) and injected into the prompt | `helpers/utils.py` |

## AI, search and infrastructure terms

| Term | Meaning | Where used |
|------|---------|------------|
| pydantic-ai | Agent framework (`Agent`, `Tool`, `RunContext`, `ModelRetry`); pinned `pydantic-ai==1.2.1` | `agents/*`, `requirements.txt` |
| Route `gpt41` / `gemma` | Weighted model routing between the default model (`AGRINET_MODEL`) and a Gemma vLLM model; sticky per session | `agents/models.py`, `app/services/agrinet_routing.py` |
| Capacity gate | Deflects requests from Gemma to the default route when vLLM concurrency reaches `AGRINET_GEMMA_MAX_CONCURRENCY` | `app/services/agrinet_routing.py` |
| vLLM | OpenAI-compatible inference server used for self-hosted models | `agents/models.py` |
| Langfuse | LLM tracing and scoring platform (`@observe`, `create_score`) | `helpers/langfuse_*.py`, `app/services/chat.py` |
| Telemetry events | Sunbird-style events (`OE_START`, `OE_ITEM_RESPONSE`, `OE_INTERACT`, ...) relayed to `TELEMETRY_API_URL` | `helpers/telemetry.py`, `app/tasks/telemetry.py` |
| Marqo | Vector search engine used by `search_documents` / `search_pests_diseases` | `agents/tools/search.py` |
| Qdrant | Vector database for scheme documents and videos | `helpers/scheme_qdrant_search.py`, `helpers/video_qdrant_search.py` |
| E5 (`intfloat/multilingual-e5-large`) | Multilingual embedding model baked into the image | `Dockerfile`, `helpers/scheme_qdrant_search.py` |
| Glossary / TermPair | Multilingual agricultural term pairs searched with RapidFuzz by `search_terms`; `transliteration` is a Hindi-in-Latin field | `agents/tools/terms.py`, `assets/glossary_terms.json` |
| Bhashini / Dhruva | Government speech and translation APIs (`dhruva-api.bhashini.gov.in`), authenticated by `MEITY_API_KEY_VALUE` | `helpers/transcription.py`, `helpers/tts.py`, `helpers/translation.py` |
| Whisper | OpenAI speech-to-text alternative selected by `service_type` | `helpers/transcription.py`, `app/models/requests.py` |
| Photon | OSM-based geocoder used by `forward_geocode` / `reverse_geocode` and NPSS location resolution; HTTPS URL required | `agents/tools/maps.py`, `app/core/npss_geocodes.py` |
| Play Integrity | Google Play app-attestation flow used to issue tokens for Android clients | `app/routers/token.py` |
| SSE | Server-Sent Events: chat responses stream as `text/event-stream` | `app/routers/chat.py` |
| Supervisord | Process manager that runs uvicorn in the container | `supervisord.conf` |
| svanetwork | External Docker network shared by the app, Redis and Portainer ("SVA" prefix also appears in `sva-cache-` and `_SVA` key names) | `docker-compose.yml`, `app/config.py`, `app/utils.py` |
| Redis Stack | Redis distribution used for cache and history | `docs/setup/redis.md` |
| OTP flow | Two-step tool pattern (`initiate_*` then `check_*_with_otp`) for personal scheme data | `agents/tools/pmkisan_scheme_status.py`, `pmfby_scheme_status.py`, `aif.py` |

## Source Files

`LICENSE`, `agents/tools/*.py`, `agents/moderation.py`, `assets/prompts/moderation_system.md`, `helpers/*.py`, `app/config.py`, `app/services/*.py`, `app/routers/*.py`, `docker-compose.yml`.
