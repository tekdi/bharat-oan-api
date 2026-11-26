# Business Rules

Rules the service applies, traced to source. Status labels: **Implemented (code)** = enforced by Python code; **Implemented (prompt)** = enforced by instructions in `assets/prompts/*.md` that the LLM is expected to follow (not a hard code gate); **Inferred from code** = interpretation of behaviour that is not explicitly documented.

Related pages: [Business Overview](./business-overview.md) | [Workflows](./workflows.md) | [Users and Personas](./users-and-personas.md) | [AI / LLM Integrations](../integrations/ai-llm.md) | [Third-Party Integrations](../integrations/third-party-integrations.md) | [Configuration](../architecture/configuration.md) | [Technical Debt](../knowledge/technical-debt.md)

---

## Moderation Rules

Source: `assets/prompts/moderation_system.md`, `agents/moderation.py`, `assets/prompts/agrinet_en.md` ("Moderation Categories"). Enforcement: classification by the moderation LLM (**Implemented, prompt**); the verdict is passed to the main agent, which is told to answer with a fixed template and never override it (**Implemented, prompt**). There is no code branch that skips the main agent for refused categories.

| Category | Meaning (from policy) | Response to farmer (English template, translated per language) |
|----------|----------------------|----------------------------------------------------------------|
| `valid_agricultural` | Crops, livestock, soil, inputs, irrigation, pests, weather, mandi, schemes, machinery, welfare, follow-ups; weather and mandi queries need no explicit farming context | Processed normally with tools |
| `invalid_non_agricultural` | No farming or farmer-welfare link | "Friend, I'm here specifically to help with farming and agriculture questions..." |
| `invalid_external_reference` | Question grounded primarily in fiction, mythology, movies, social media or religious texts (test: does it still make sense without the fictional source?) | "I work with only trusted agricultural sources..." |
| `invalid_compound_mixed` | Current message has a separable agricultural and non-agricultural request (history does not count) | "I focus only on farming and agricultural matters..." |
| `invalid_language` | Explicit request to respond in a non-Indian language (e.g. German); queries may arrive in any language | Lists the supported languages and asks to continue in one of them |
| `unsafe_illegal` | Banned or restricted agrochemicals, illegal activity, fraud, tax evasion, harm, insurance fraud, gaming schemes, dual registration - even in farming context | "I share only safe and legal farming practices..." |
| `political_controversial` | Partisan comparison, voting advice, caste-based discrimination claims, regional fairness debates about policy | "I provide farming information without getting into politics..." |
| `role_obfuscation` | Prompt injection, instruction override, system-prompt extraction, persona switching, emotional manipulation or bribery, encoded text, requests for API keys/internals | "I'm here specifically for agricultural and farming assistance..." |

| Moderation rule | Detail | Source |
|-----------------|--------|--------|
| Priority when several apply | unsafe_illegal > role_obfuscation > political_controversial > invalid_compound_mixed > invalid_external_reference > invalid_non_agricultural > invalid_language > valid_agricultural | `moderation_system.md` |
| Default stance | "When in doubt, decline rather than allow" (government project) | `moderation_system.md` |
| Scheme acronyms | Bare acronyms or short codes (MIF, PM-KMY, CDP, NMEO-OS, Makhana, e-NAM, RWBCIS, KCC, PMFBY, PM-KISAN, SHC, FFS, NFSM, SMAM, AIF, PDMC, PKVY, PMKSY, SATHI, PMASHA, RAD, NBHM, etc.) are valid agricultural | `moderation_system.md` |
| "Policy status", "claim status", "scheme status" without a scheme | Valid agricultural (the assistant then asks which scheme) | `moderation_system.md` |
| Short replies ("Yes", "Haan", "और बताओ") | Valid when the prior conversation is agricultural; without history they have no intent | `moderation_system.md` |
| Each message judged independently | Prior agricultural history does not make an unsafe message safe | `moderation_system.md` |
| Output format | JSON `{"category": ..., "action": ...}`; action text always English | `QueryModerationResult` in `agents/moderation.py` |
| Determinism | temperature 0.0, timeout 20 s (`LLM_MODERATION_TIMEOUT_SECONDS`), 2 retries | `agents/moderation.py` |
| Context given to classifier | Conversation prefix (last 3 pairs) + query + selected language | `app/services/chat.py` |
| Moderation failure | Not caught: the chat turn errors (no default verdict) | Inferred from code (`_run_moderation` has no try/except) |
| Evaluation fixtures | `tests/fixtures/moderation_single_turn.json`, `moderation_multi_turn.json` (live-LLM script tests) | `tests/test_moderation_*.py` |

---

## Language Support

| Rule | Detail | Source |
|------|--------|--------|
| Supported response languages (13) | English `en`, Hindi `hi`, Assamese `as`, Bengali `bn`, Gujarati `gu`, Kannada `kn`, Maithili `mai`, Malayalam `ml`, Marathi `mr`, Odia `or`, Punjabi `pa`, Tamil `ta`, Telugu `te` | `assets/prompts/agrinet_*.md`, `SUPPORTED_LANGS` in `agents/tools/terms.py` |
| Newest additions | Odia (`or`), Punjabi (`pa`), Maithili (`mai`) - prompts, glossary fields, NPSS labels and script ranges, tests | `assets/prompts/agrinet_{or,pa,mai}.md`, `agents/tools/terms.py`, `app/services/npss_response.py`, `tests/test_languages.py` |
| ISO code note | Odia uses ISO `or` in this repo (source comment: another service uses `od`) | `agents/tools/terms.py` |
| Language of the answer | The `target_lang` request parameter selects the prompt file and the "Selected Language" line; default `hi`. `source_lang` is only recorded as Langfuse metadata | `agents/agrinet.py`, `agents/deps.py`, `app/services/chat.py` |
| One language per answer | No mixing of languages mid-response; tool calls always in English | Prompt (**Implemented, prompt**) |
| Numerals | Always Roman/Arabic digits 0-9 in answers, never regional-script numerals; farmer-supplied native digits are converted to ASCII before tool calls | Prompt; `helpers/utils.py::to_ascii_digits` (**code**) |
| Source citation language | Entire citation including the word "Source" is translated, except network-provided scheme source names from `search_schemes`, where only the word "Source" is translated | Prompt |
| Search language | Tool queries are English (2-5 words); `search_terms` and `search_commodity` accept native-language terms | Prompt; `terms.py`, `commodity.py` |
| Unsupported `target_lang` | No validation; a language without a prompt file raises a template error when the system prompt renders | Inferred from code (`get_prompt(f'agrinet_{lang_code}')`) |
| Voice | ASR passes `lang_code` to Bhashini unchanged (no allow-list in code); TTS passes `target_lang` unchanged | `helpers/transcription.py`, `helpers/tts.py` |
| Prompt parity | Only `en, mai, or, pa` render the live vector-scheme list; other languages carry a static list | `assets/prompts/*.md` |

---

## Tool Selection Rules

Source: "Tool Selection Guide" and related sections of `assets/prompts/agrinet_en.md` (translated equivalents in other prompts). **Implemented (prompt)** unless stated.

| Query type | Tool chain | Source label shown | Notes |
|------------|------------|--------------------|-------|
| Crop / seed information | `search_documents` | Name from tool response | Primary info source |
| Livestock diseases and issues | `search_documents` | Name from tool response | Not `search_pests_diseases` |
| Crop pests and diseases | `search_pests_diseases` | Name from tool response | Crop only |
| Crop image upload | `analyze_crop_image` | NPSS (fixed by code) | Do not call `search_pests_diseases` automatically |
| eNAM workflows / training | `search_schemes` then `search_video` | Scheme source + video titles | Do not also call `search_documents` |
| Weather | `forward_geocode` then `weather_forecast` | India Meteorological Department | Geocode place names first |
| Mandi prices | `forward_geocode` -> `search_commodity` -> `get_mandi_prices` | Mandi Prices | Date intent first, then location |
| Legacy scheme info (16 codes) | `get_scheme_info(code)` | Government Scheme Information | Exact code only; never auto-complete (`nbm` is not `nbhm`, `ffs` is not `nfsm`) |
| Vector-indexed schemes | `search_schemes` | As returned by the tool | PKVY always here; NBM always `get_scheme_info("nbm")` |
| MahaVistaar schemes | `call_maha_vistaar_network` | Government Scheme Information | Only `ndksp-drip-irrigation`, `ndksp-farm-pond-lining`, `aif` |
| Amul union schemes | `call_amul_vistaar_network` | Government Scheme Information | Unions `banas, kutch, sumul, surendranagar` |
| Bare "drip irrigation" | Ask which scheme (PDMC, NDKSP, AIF) before any tool | - | Never default to `search_documents` |
| Scheme not covered by any tool | `search_terms` (if regional language) then `search_documents` | - | Only then say unavailable |
| Status: PMFBY / PM-KISAN / SHC / SMAM / AIF | See [Workflows W2](./workflows.md#w2-streaming-chat-turn) | Portal names | Offer status checks only after scheme info or on request; never for KCC, PMKSY, SATHI, PMASHA, PDMC, FFS, NBHM |
| Official fertilizer dose | `forward_geocode` -> `gfr_get_crop_registries` -> `gfr_get_recommendations` | GFR Crop Recommendation | Skip `search_terms` |
| Seed availability | SATHI chain | SATHI | Maharashtra only |
| Term lookup | `search_terms` | none | Only before crop/pest/knowledge searches; skip for weather, mandi, scheme, status, grievance, GFR, SATHI |

General tool rules: always use tools and never answer from memory; no redundant calls with identical parameters; on empty/failed results say so plainly, do not add general tips or external links; reuse already-provided crop, location, date, scheme, phone or reg-no within the conversation; call the scheme tool again on every scheme follow-up turn; never output raw JSON, tool syntax or LaTeX.

Code-level tool constraints (**Implemented, code**):

| Tool | Constraint | Source |
|------|------------|--------|
| `get_scheme_info` | `scheme_name` is a `Literal` of the 16 codes | `scheme_info.py` |
| `search_schemes` | Query naming an unknown scheme returns "Scheme not available right now"; empty hits for a known scheme return "Could not find this information right now" | `helpers/scheme_qdrant_search.py` |
| `search_terms` | threshold must be in [0, 1]; default 0.7 (prompt asks for 0.5) | `terms.py` |
| `search_commodity` | default threshold 0.65, max 5 results | `commodity.py` |
| `forward_geocode` | Restricted to India bounding box [68, 6, 98, 36] | `maps.py` |
| `gfr_get_recommendations` | state id length >= 6; 1-6 crop ids (extra ignored); phone normalized to 10 digits (+91); cycle required | `gfr.py` |
| `gfr_get_crop_registries` | `limit` clamped to 1-50 | `gfr.py` |
| SATHI search | Max 60 dealers per result, 15 varieties per dealer in the formatter | `sathi_seed.py` |

---

## Scheme Eligibility and Exclusion Rules

| Rule | Detail | Status / source |
|------|--------|-----------------|
| Eligibility questions | Two labelled sections: "Who is eligible" (Eligibility chunks) then "Who is not eligible" (Exclusion chunks, whenever exclusion data is present) | Prompt |
| Exclusion-only questions | One section only; no eligibility content | Prompt |
| Never merge | Eligibility and exclusion bullets are never combined; exclusion never inferred from eligibility text | Prompt |
| Missing exclusion | State that exclusion criteria could not be found; do not infer | Prompt |
| Chunk labelling | Scheme chunks are classified into `eligibility`, `exclusion`, `application`, `support`, `other` by heading/keyword patterns | `helpers/scheme_qdrant_search.py::classify_chunk_section` (**code**) |
| Query focus detection | `eligibility_with_exclusion` when the query has eligibility terms; `exclusion_only` when it has exclusion terms without eligibility terms; the search expands the query and tops up the missing section | `classify_scheme_section_focus`, `_merge_supplemental_results` (**code**) |
| Result balancing | Eligibility+exclusion queries return a balanced mix of both sections | `_take_balanced` (**code**) |
| Intent-based reranking | `application` and `support` intents boost matching chunks and penalise institutional/table noise | `rerank_results` (**code**) |
| Context carry-over | Follow-ups ("how to apply?") refer to the scheme already discussed; never reset scheme context | Prompt |
| State what the tool returns | No facts from memory about scheme rules | Prompt |
| PM-KISAN 23rd instalment | On or before 20 June 2026 answer in future tense, from 21 June 2026 in past tense, using the pre-formatted English/Hindi sentence from the tool | `helpers/pmkisan_installment_release.py` (**code**) and prompt. Time-bound content |
| Scheme eligibility itself (income limits, land limits, etc.) | Not encoded in this service; comes from provider data via the network | Inferred from code |

---

## Authentication and Session Rules

| Rule | Value | Source |
|------|-------|--------|
| JWT algorithm | RS256 | `settings.jwt_algorithm` |
| Guest / API-key / Play-Integrity token lifetime | `JWT_EXPIRY_MINUTES`, default 15 | `app/config.py` |
| Audience / issuer verification | Disabled | `app/auth/jwt_auth.py` |
| Play Integrity freshness | 120 s (`PLAY_INTEGRITY_FRESHNESS_SECONDS`), future skew 10 s | `app/routers/token.py` |
| Play Integrity verdicts required | `PLAY_RECOGNIZED` app, `MEETS_DEVICE_INTEGRITY` device; licensing verdict only logged | `_validate_play_integrity_payload` |
| Nonce replay protection | Redis key `integrity_nonce:<nonce>`, TTL = freshness window | `_ensure_nonce_not_reused` |
| API key comparison | Constant-time (`hmac.compare_digest`) | `create_auth_token_with_api_key` |
| Startup validation | Empty `API_KEY_AUTH_TOKEN_*`, `PLAY_INTEGRITY_PACKAGE_NAME_*`, `PLAY_INTEGRITY_PRIVATE_KEY_*` variables abort startup | `validate_multi_provider_auth_config` |
| Role-based access | None enforced | See [Users and Personas](./users-and-personas.md#24-authorization-roles) |
| Rate limiting | Not implemented (setting `rate_limit_requests_per_minute=1000` unused) | `app/config.py` |

## Conversation and Session Rules

| Rule | Value | Source |
|------|-------|--------|
| History storage | Redis key `<session_id>__SVA`, TTL 2 hours | `app/utils.py` |
| Context given to the model | Last 3 user/assistant pairs prepended to the user message plus trimmed message history up to 64,000 tokens | `app/services/chat.py`, `trim_history` |
| First conversation always retained when trimming | Yes (newest-first fill thereafter) | `trim_history` |
| Thinking content | Never stored or replayed | `filter_thinking_from_history` |
| New session id | UUID4 when none supplied | `ensure_session_id` |
| Question id | Client `qid` or UUID4; returned in `X-QID` | `app/routers/chat.py` |
| Model route pin | Per session, TTL `AGRINET_ROUTE_TTL_SECONDS` (7200), refreshed after each turn | `app/services/agrinet_routing.py` |
| Run time limit | 45 s per agent run | `AGRINET_MODEL_TIMEOUT_SECONDS` |
| Failover | Once, only if no text was streamed, only when routing is enabled | `app/services/chat.py` |
| Feedback to Langfuse | Only while `chat_turn:<qid>` exists (2 h) and the session id matches | `app/routers/telemetry.py` |

---

## Status Checks, OTP and Identifier Rules

| Scheme | Rule | Source |
|--------|------|--------|
| All OTP/ID inputs | Native-script digits converted to ASCII; never use placeholder numbers; never echo OTP digits (AIF: never repeat OTP) | `to_ascii_digits`; prompt |
| PM-KISAN status | Registration number (usually 11 digits) or 10-digit mobile; OTP exactly 4 digits; OTP sent to registered mobile automatically | `pmkisan_scheme_status.py`; prompt |
| PM-KISAN grievance | Registration number required; 4-digit OTP; grievance type must be one of 10 codes `G001`-`G010` (e.g. installment not received `G003`, payment related `G007`, eKYC problems `G008`-`G010`) | `pmkisan_grievance.py`, `assets/grievance_types.json` |
| PMFBY status | OTP exactly 6 digits; `inquiry_type` in {policy_status, claim_status}; season in {Kharif, Rabi, Summer}; year required; phone normalized | `pmfby_scheme_status.py` |
| PMFBY grievance | 10-digit registered mobile first, then 6-digit OTP, only then application number, season/year, complaint; receipt source id fixed by system; tracking needs both mobile and ticket number (10 digits = mobile, longer = ticket) | `pmfby_grievance.py`; prompt |
| Soil Health Card | Phone + cycle year; HTML report cached 10 minutes | `shc_scheme_status.py` |
| SMAM | Search by `mobile` (10 digits, `+91`/`0` stripped) or `application_no`; Aadhaar must not be used | `smam_scheme_status.py`; prompt |
| AIF | Beneficiary ID, 6-digit OTP via mandatory `initiate_aif_otp` and `verify_aif_otp`; verify once per conversation; one number requested at a time; "no open grievances" is a normal result; never ask for a ticket number; cite "AIF Portal" only for results, not OTP steps | `aif.py`; prompt |
| Payment issues | If a claim is approved but unpaid, share UTR (12-digit reference) and guide the farmer to the bank | Prompt |
| PII in logs | PMFBY masks selected fields; AIF masks the OTP; other tools log raw responses | `pmfby_scheme_status.py::mask_pii_value`, `aif.py::_loggable` |

## Mandi Rules

| Rule | Detail | Source |
|------|--------|--------|
| Date first | No tool call until the farmer gives a valid date, asks for today, or chooses latest; "latest"/"current" alone do not count as today | Prompt |
| Date validity | Must be a real calendar date and not in the future; never guess or clamp; format DD-MM-YYYY | Prompt; `MandiRequest.price_date` |
| Date ranges | A range already counts as date intent | Prompt |
| Location | At least district/city; ask state only when ambiguous; unambiguous metros and city-states proceed directly | Prompt |
| No data | Say so; never substitute another date | Prompt |
| Carry-forward | Reuse stated crop, location and date for follow-ups | Prompt |

## Response Style Rules (Implemented, prompt)

| Rule | Detail |
|------|--------|
| Length | 2-4 sentences simple, 6-8 complex, hard max 10 |
| Order | Answer, then bold source line on its own line, then one follow-up question within agricultural scope and tool capability |
| Abbreviations | Written with full stops (e.g. P.M.F.B.Y.) in English |
| Language of dosages | Plain farmer-friendly wording, local units where possible, no chemical formulas |
| Zero fabrication | No advice from memory; "no verified source found" stated if a tool gives no source |
| Citations | Mandatory for tool-derived facts; not for errors or empty results |
| Empathy | Acknowledge frustration before grievance processes |

## Image and Upload Rules

| Rule | Value | Source |
|------|-------|--------|
| Upload types | `image/jpeg`, `image/jpg`, `image/png` | `app/routers/image.py` |
| Max upload | 10 MB (`MAX_IMAGE_UPLOAD_SIZE_MB`), HTTP 413 beyond | `app/core/image_storage.py` |
| NPSS accepted formats | JPEG, PNG (magic-byte detected); WebP rejected; max 10 MB | `agents/tools/npss.py` |
| Location for NPSS | Browser coordinates when present (resolved through Photon + NPSS masters, strict verified matching); else Krishi Vigyan Kendra Delhi fallback; farmer is never asked for location; a (0,0) pair is not treated as real | `npss.py`, `app/core/npss_geocodes.py` |
| Result presentation | Pest, Crop, Cause card plus short summary; no treatment, prevention or follow-up question | Prompt wrapper in `chat.py`; `npss_response.py` |
| Attribution | Fixed localized `Source: NPSS`; model-written sources removed | `npss_response.py` (**code**) |
| Image retention | Deleted (or moved to GCS mount) after successful NPSS call; unprocessed images are not auto-cleaned | `image_storage.py` |
| Image URL | UUID-addressed, served without JWT | `app/routers/image.py` |

---

## Product Non-Rules and Reconciliation Notes

| Topic | Observation |
|-------|-------------|
| Hard moderation gate | Not implemented; refusal depends on the main model following the prompt |
| Role-based permissions | Not implemented (see [Users and Personas](./users-and-personas.md#24-authorization-roles)) |
| Rate limit of 1000 requests/min (`API_DOCUMENTATION.md`) | Not enforced |
| 30-day token validity (`API_DOCUMENTATION.md`) | Code default is 15 minutes |
| Suggestions generation | Rule set exists in `assets/prompts/suggestions_system.md` (3-5 questions, English/Hindi/Marathi only, farmer's perspective) but the feature is disabled |
| Supported-language list in `suggestions_system.md` | Covers 3 languages only, narrower than the 13 chat languages |

## Source Files

`assets/prompts/moderation_system.md`, `assets/prompts/agrinet_en.md`, `agents/moderation.py`, `agents/tools/*.py`, `helpers/scheme_qdrant_search.py`, `helpers/pmkisan_installment_release.py`, `helpers/utils.py`, `app/services/chat.py`, `app/services/agrinet_routing.py`, `app/utils.py`, `app/routers/token.py`, `app/auth/jwt_auth.py`, `app/core/image_storage.py`, `app/config.py`, `tests/test_languages.py`.
