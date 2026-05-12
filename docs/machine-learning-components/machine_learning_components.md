# 5. Machine Learning Components

This section documents the machine learning subsystem of OMVision, which classifies ingested companies based on their relevance to OMVC's investment thesis. The scoring pipeline transforms enriched company data into structured relevance scores, producing ranked outputs that enable prioritized deal evaluation.

The classification system operates as part of the `classify_ingested_companies` Dagster job (`app/jobs/classify_ingested_companies.py`), which runs after companies have been ingested and enriched from various data sources. The pipeline outputs a composite relevance score (0–100) and a discrete rank (0–3) for each company, stored in the `Company.llm_score` and `Company.rank` fields and used by the frontend for sorting, filtering, and analyst review.

> **Historical note:** Prior to May 2026 this pipeline used a LightGBM ordinal classifier wrapped in an `MLResource`. That approach was replaced entirely by the GPT-4o LLM scoring pipeline described in this section. The `MLResource` class and all `lightgbm_*.pkl` model artifacts are no longer part of the production codebase.

```mermaid
flowchart TB
    subgraph Input["Data Input"]
        DB[(PostgreSQL<br/>Unclassified Companies)]
        DB --> Fetch[fetch_companies_for_scoring]
        Fetch --> Companies[list[CompanyForScoring]]
    end

    subgraph Enrich["Enrichment"]
        Companies --> Enrich_op[enrich_company_data]
        Harmonic[(Harmonic<br/>GraphQL API)] --> Enrich_op
        Enrich_op --> EnrichedList[list[EnrichedCompanyData]]
    end

    subgraph Score["LLM Scoring"]
        EnrichedList --> Score_op[score_company_with_llm]
        OpenAI[(OpenAI Responses API<br/>gpt-4o + web_search_preview)] --> Score_op
        Score_op --> Results[list[CompanyScoreResult]]
    end

    subgraph Write["Database Persistence"]
        Results --> Write_op[write_scores_to_db]
        Write_op --> DB2[(PostgreSQL<br/>llm_score · rank · llm_reasoning<br/>llm_justification · analyst_feedback)]
    end

    style Input fill:#e1f5ff
    style Enrich fill:#fff4e1
    style Score fill:#f0e1ff
    style Write fill:#e1ffe6
```

**Key Design Principles:**

- **End-to-End LLM Pipeline**: A single GPT-4o call per company replaces the multi-step feature-extraction → ML-inference chain, enabling richer reasoning and human-readable justifications alongside the numerical score
- **Structured Output with Strict JSON Schema**: The Responses API returns a validated `LLMScoringOutput` object. OpenAI strict mode is enforced by post-processing the Pydantic schema to inject `additionalProperties: false` on every object node (see §5.3)
- **Grounded Web Search**: The `web_search_preview` tool is available to the model per call, enabling it to verify company information against live sources when Harmonic data is sparse
- **Harmonic Founder Enrichment**: Founder profiles are resolved from Harmonic GraphQL before scoring, giving the model high-quality team signal that is not present in the base company record
- **Analyst Feedback Scaffold**: Every scored company receives a pre-populated `analyst_feedback` JSONB skeleton, ready for human override annotations on any scoring dimension

---

## 5.1 Classification Pipeline

The classification pipeline executes as a Dagster job with four sequential operations that transform unclassified companies into scored and ranked records. The job is scheduled to run daily after ingestion jobs complete, processing all companies where `Company.rank IS NULL` that were created today (UTC).

**Job Definition** (`app/jobs/classify_ingested_companies.py`):

```python
@job
def classify_ingested_companies():
    companies = fetch_companies_for_scoring()
    enriched = enrich_company_data(companies)
    scores = score_company_with_llm(enriched)
    write_scores_to_db(scores)
```

**Resources used by this job:**

| Resource | Class | Purpose |
|----------|-------|---------|
| `db` | `DatabaseResource` | Fetch unclassified companies; bulk-update scores |
| `harmonic` | `HarmonicResource` | GraphQL query for founder profiles |
| `openai` | `OpenAIResource` | LLM scoring via Responses API |

### 5.1.1 Fetch Companies for Scoring

The `fetch_companies_for_scoring` operation retrieves companies that have not yet been classified and constructs typed input objects for the downstream ops.

**Operation** (`fetch_companies_for_scoring`):

```python
@op
def fetch_companies_for_scoring(
    context,
    db: DatabaseResource,
) -> list[CompanyForScoring]:
    rows = db.fetch_unclassified_companies()
    companies = [CompanyForScoring.from_db_row(row) for row in rows]
    context.log.info(f"Companies to score today: {len(companies)}")
    return companies
```

**`db.fetch_unclassified_companies()` behavior:**

- Selects rows where `Company.rank IS NULL`
- Deduplicates by `source_company_id`, retaining `MAX(id)` per group
- Applies a `created_at` filter scoped to today (UTC)
- Returns joined `Company` + `CompanyMetric` rows

**`CompanyForScoring` schema** (`app/schemas/llm_scoring.py`):

| Field | Type | Source |
|-------|------|--------|
| `id` | `int` | `Company.id` |
| `name` | `Optional[str]` | `Company.name` |
| `description` | `Optional[str]` | `Company.description` |
| `tags` | `list` | `Company.tags` |
| `location` | `dict` | `Company.location` |
| `founding_date` | `dict` | `Company.founding_date` |
| `highlights` | `list` | `CompanyMetric.highlights` |
| `employee_highlights` | `list` | `CompanyMetric.employee_highlights` |
| `headcount` | `Optional[int]` | `CompanyMetric.headcount` |
| `funding` | `dict` | `CompanyMetric.funding` |
| `stage` | `Optional[str]` | `CompanyMetric.stage` |
| `traction_metrics` | `dict` | `CompanyMetric.traction_metrics` |

The `from_db_row(row)` classmethod constructs a `CompanyForScoring` from a joined ORM row, defaulting all JSON/list fields to empty containers rather than `None`.

### 5.1.2 Enrich Company Data

The `enrich_company_data` operation queries the Harmonic GraphQL API to resolve founder profiles for each company before scoring. Richer founder data improves the model's ability to evaluate team strength (STEP 5 of the evaluation framework).

**Operation** (`enrich_company_data`):

```python
@op
def enrich_company_data(
    context,
    companies: list[CompanyForScoring],
    harmonic: HarmonicResource,
) -> list[EnrichedCompanyData]:
    enriched = []
    for company in companies:
        enrichment_used: list[str] = []
        founder_profiles: list[FounderProfile] = []

        try:
            founder_profiles = fetch_founder_profiles(company, harmonic, context)
            if founder_profiles:
                enrichment_used.append("harmonic_founders")
        except Exception as exc:
            context.log.error(f"[{company.name}] Harmonic enrichment failed: {exc}")

        enriched.append(
            EnrichedCompanyData(
                company=company,
                founder_profiles=founder_profiles,
                enrichment_used=enrichment_used,
            )
        )

    context.log.info(f"Enriched {len(enriched)} companies")
    return enriched
```

**Founder profile resolution** (`fetch_founder_profiles` helper):

1. Scans `company.employee_highlights` for entries whose `title` field contains any of: `"founder"`, `"co-founder"`, `"ceo"`, `"cto"` (case-insensitive)
2. Collects the `person_urn` values from matching entries
3. Fires a single Harmonic GraphQL query (`getPersonsByUrns`) for all URNs in batch
4. Maps the response into `FounderProfile` objects

**Harmonic GraphQL query used:**

```graphql
query GetPersonsByUrns($urns: [String!]!) {
  getPersonsByUrns(urns: $urns) {
    fullName
    linkedinUrl
    title
    education {
      school { name }
      degree
      fieldOfStudy
    }
    experience {
      company { name }
      title
      isCurrent
    }
    highlights {
      text
    }
  }
}
```

**`FounderProfile` schema** (`app/schemas/llm_scoring.py`):

| Field | Type | Description |
|-------|------|-------------|
| `name` | `str` | Full name from Harmonic |
| `title` | `Optional[str]` | Current title |
| `linkedin_url` | `Optional[str]` | LinkedIn profile URL |
| `education` | `list[dict]` | School name, degree, field of study |
| `previous_companies` | `list[dict]` | Company name, title, is_current flag |
| `highlights` | `list[str]` | Harmonic highlight text entries |

**`EnrichedCompanyData` schema:**

```python
class EnrichedCompanyData(BaseModel):
    company: CompanyForScoring
    founder_profiles: list[FounderProfile] = Field(default_factory=list)
    enrichment_used: list[str] = Field(default_factory=list)
```

The `enrichment_used` list records which enrichment sources were successfully applied (e.g., `["harmonic_founders"]`). If Harmonic enrichment fails or returns no matching profiles, the list remains empty and scoring proceeds with base company data only — the op does not raise on enrichment failures.

### 5.1.3 Score Company with LLM

The `score_company_with_llm` operation calls `OpenAIResource.score_company_relevance()` for each enriched company, collecting structured scoring results.

**Operation** (`score_company_with_llm`):

```python
@op
def score_company_with_llm(
    context,
    enriched_companies: list[EnrichedCompanyData],
    openai: OpenAIResource,
) -> list[CompanyScoreResult]:
    results: list[CompanyScoreResult] = []
    for enriched in enriched_companies:
        if (
            not (enriched.company.name or "").strip()
            and not (enriched.company.description or "").strip()
        ):
            context.log.warning(
                f"[id={enriched.company.id}] Skipping — no name or description"
            )
            continue
        try:
            result = openai.score_company_relevance(enriched)
            results.append(result)
            context.log.info(
                f"[{enriched.company.name}] Score: {result.llm_score} "
                f"(industry={result.industry_fit.score if result.industry_fit else 'N/A'}, "
                f"stage={result.stage_fit.score if result.stage_fit else 'N/A'}, "
                f"biz_model={result.business_model.score if result.business_model else 'N/A'}, "
                f"web_search={'yes' if result.web_search_used else 'no'})"
            )
        except Exception as exc:
            context.log.error(f"[{enriched.company.name}] Scoring failed: {exc}")

    context.log.info(f"Scored {len(results)}/{len(enriched_companies)} companies")
    return results
```

**Skip condition:** Companies with no `name` AND no `description` are skipped entirely — the model cannot produce a meaningful score without any textual input.

**Per-company log line:** The op logs score, `industry_fit.score`, `stage_fit.score`, `business_model.score`, and whether web search was invoked, providing a compact audit trail in Dagit.

**Error isolation:** Scoring failures (e.g., OpenAI API errors, JSON parse failures) are caught per-company and logged as errors. The rest of the batch continues — a single company failure does not abort the job.

### 5.1.4 Write Scores to Database

The `write_scores_to_db` operation assembles update dicts from `CompanyScoreResult` objects and persists them in a single bulk operation.

**Operation** (`write_scores_to_db`):

```python
@op
def write_scores_to_db(
    context,
    db: DatabaseResource,
    score_results: list[CompanyScoreResult],
) -> None:
    if not score_results:
        context.log.info("No scores to write.")
        return
    updates = [_build_company_update(r) for r in score_results]
    db.bulk_update_llm_scores(updates)
    context.log.info(f"Wrote scores for {len(updates)} companies")
```

**`_build_company_update()` output structure:**

```python
{
    "id": result.company_id,
    "llm_score": result.llm_score,           # Integer 0-100
    "rank": float(derive_rank(result.llm_score)),  # 0.0-3.0
    "llm_justification": result.llm_justification, # list[str], ≤4 bullets
    "llm_reasoning": {
        "chain_of_thought": result.chain_of_thought,
        "dimensions": {
            k: {"score": v.score, "reasoning": v.reasoning}
            for k, v in _dimensions.items()
            if v is not None
        },
        "enrichment_used": result.enrichment_used,
        "model": "gpt-4o",
        "prompt_version": "v2.0",
        "scored_at": "<ISO 8601 UTC datetime>",
    },
    "analyst_feedback": ANALYST_FEEDBACK_SKELETON,  # deep copy per company
}
```

The `_dimensions` dict contains the seven scoring dimensions: `industry_fit`, `stage_fit`, `business_model`, `founder_strength`, `team_strength`, `investor_strength`, `highlights`. Dimensions that are `None` (missing data) are excluded from the `dimensions` sub-dict — the model's null signal is preserved rather than stored as a zero.

**`db.bulk_update_llm_scores(updates: list[dict])` behavior:**

Executes a bulk `UPDATE` on the `company` table from the list of update dicts, writing `llm_score`, `rank`, `llm_justification`, `llm_reasoning`, and `analyst_feedback` in a single transaction.

**Fields written to `Company`:**

| Column | Type | Description |
|--------|------|-------------|
| `llm_score` | `Integer` | Raw LLM score 0–100 |
| `rank` | `Float` | Derived rank 0.0–3.0 (see §5.2) |
| `llm_justification` | `ARRAY(Text)` | 1–4 bullet strings from the model |
| `llm_reasoning` | `JSONB` | Full reasoning object (chain_of_thought, dimensions, metadata) |
| `analyst_feedback` | `JSONB` | Pre-populated feedback skeleton for analyst annotations |

---

## 5.2 Scoring Model

### 5.2.1 Rank Derivation

The `derive_rank()` function (`app/constants/llm_scoring.py`) converts the continuous 0–100 LLM score into the discrete `Company.rank` integer used for frontend sorting and filtering.

**Implementation:**

```python
def derive_rank(llm_score: int) -> int:
    if llm_score >= 80:
        return 3
    if llm_score >= 70:
        return 2
    if llm_score >= 50:
        return 1
    return 0
```

**Score threshold table:**

| `llm_score` | `rank` | Interpretation | Recommended Action |
|-------------|--------|----------------|--------------------|
| 80–100 | 3 | Highly relevant — strongly suggest outreach | Priority review |
| 70–79 | 2 | Relevant — recommend speaking with them | Standard evaluation queue |
| 50–69 | 1 | Somewhat relevant — chat if time allows | Lower priority |
| 0–49 | 0 | Not worth reaching out | Deprioritized / filtered |

The `rank` value stored in the database is cast to `float` (`float(derive_rank(...))`) for compatibility with the existing `Company.rank` column type, which is `Float` in the ORM.

### 5.2.2 Evaluation Framework

The system prompt (`PROMPT_VERSION = "v2.0"`, defined in `app/constants/llm_scoring.py`) instructs GPT-4o to evaluate each company through eight sequential steps. Steps 0–6 each map to a scoring dimension; Step 7 covers highlights.

**7-dimension evaluation framework:**

| Step | Dimension | Key Criteria | Hard Disqualifiers |
|------|-----------|-------------|-------------------|
| STEP 0 | Entity legitimacy | Real, investable software startup | Non-company, VC fund, dev shop, services firm, scam → score ≤15 |
| STEP 1 | Geography | US, Canada, SE Asia, Australia, South Korea, Middle East | Non-target geography caps score at ≤35 |
| STEP 2 | Industry fit | Fintech, climate tech (software-only), deep tech, B2B SaaS | Hardware → `industry_fit` ≤20, overall ≤30; bio/pharma/healthcare/edtech/gaming excluded |
| STEP 3 | Stage fit | Pre-Seed / Seed / Series A best fit; Series B soft negative | Headcount >75 or total funding >$20M → `stage_fit` ≤15 |
| STEP 4 | Business model | B2B / B2B2C strong; B2C weak but not disqualifying | B2C caps overall score at 65 unless exceptional signals present |
| STEP 5 | Founder strength | Domain expertise, prior startup experience, exits, pedigree | No founder data → dimension is `null`, excluded from score |
| STEP 6 | Team strength | Overall employee quality, relevant backgrounds, senior hires | No team data → dimension is `null`, excluded from score |
| STEP 7 | Highlights | Traction, awards, press, customer logos, unusual growth signals | No highlights data → dimension is `null`, excluded from score |

**Geography target list** (STEP 1): United States, Canada, Singapore, Thailand, Indonesia, Malaysia, Philippines, Vietnam, Australia, South Korea, UAE, Saudi Arabia, and other Middle East markets. Unknown geography is not penalized.

**Industry exclusion list** (STEP 2): Biology, Biomedical, Pharma, Life Sciences, Healthcare, Medicine, Apparel/Fashion, Education/Edtech, Gaming, and physical/hardware products.

**Scoring calibration reference points** (from the system prompt):

| Scenario | Expected Score Range |
|----------|---------------------|
| B2C fintech app with real funding and good team | 55–65 |
| Crypto exchange with no funding and no clear B2B model | 50–60 |
| Series B fintech with strong signals | 65–75 |
| Strong-thesis-fit B2B company with missing founder/team/investor data | 70–80 |
| Company founded 7+ years ago with mediocre traction | 45–55 |
| Dev shop or services company even if in fintech/blockchain | 0–15 |
| Hardware company with a software layer | 20–30 |

**Missing data handling:** Dimensions where no real data exists (`founder_strength`, `team_strength`, `investor_strength`, `highlights`) are returned as `null` by the model and excluded from the overall score calculation. Missing data means "we don't know" — it does not drag the score down.

### 5.2.3 Output Schema

The model returns a structured JSON object validated against `LLMScoringOutput` (`app/schemas/llm_scoring.py`). The `CompanyScoreResult` schema extends it with pipeline metadata.

**`DimensionScore` schema:**

```python
class DimensionScore(BaseModel):
    score: int  # 0-100
    reasoning: str
```

**`LLMScoringOutput` schema:**

```python
class LLMScoringOutput(BaseModel):
    chain_of_thought: str
    industry_fit:      Optional[DimensionScore]
    stage_fit:         Optional[DimensionScore]
    business_model:    Optional[DimensionScore]
    founder_strength:  Optional[DimensionScore]
    team_strength:     Optional[DimensionScore]
    investor_strength: Optional[DimensionScore]
    highlights:        Optional[DimensionScore]
    llm_score:         int  # 0-100
    llm_justification: list[str]  # 1-4 bullet strings
```

**`CompanyScoreResult` schema** (extends `LLMScoringOutput`):

```python
class CompanyScoreResult(LLMScoringOutput):
    company_id:      int
    web_search_used: bool = False
    enrichment_used: list[str] = Field(default_factory=list)
```

**Output fields:**

| Field | Type | Description |
|-------|------|-------------|
| `chain_of_thought` | `str` | Full step-by-step reasoning walking through all eight steps |
| `industry_fit` | `Optional[DimensionScore]` | Industry and sector relevance score + reasoning |
| `stage_fit` | `Optional[DimensionScore]` | Funding stage and company age fit score + reasoning |
| `business_model` | `Optional[DimensionScore]` | B2B/B2B2C/B2C classification score + reasoning |
| `founder_strength` | `Optional[DimensionScore]` | Founding team quality score + reasoning; `null` if no founder data |
| `team_strength` | `Optional[DimensionScore]` | Overall team quality score + reasoning; `null` if no team data |
| `investor_strength` | `Optional[DimensionScore]` | Investor pedigree score + reasoning; `null` if no investor data |
| `highlights` | `Optional[DimensionScore]` | Traction and milestone signals score + reasoning; `null` if no highlights |
| `llm_score` | `int` | Composite relevance score 0–100 |
| `llm_justification` | `list[str]` | 1–4 bullet strings explaining the overall score |
| `company_id` | `int` | Foreign key back to `Company.id` |
| `web_search_used` | `bool` | Whether the model invoked `web_search_preview` during scoring |
| `enrichment_used` | `list[str]` | Enrichment sources applied (e.g., `["harmonic_founders", "web_search_preview"]`) |

**Pydantic nullable fields note:** `Optional[DimensionScore]` fields in `LLMScoringOutput` are declared WITHOUT `= None`. This is intentional: Pydantic v2 includes fields in `required` only when they lack a default. OpenAI strict mode requires every nullable field to appear in `required`. Adding `= None` would remove these fields from `required` and break structured output parsing.

---

## 5.3 OpenAI Resource

The `OpenAIResource` class (`app/resources/open_ai.py`) is the integration point for all LLM calls. The scoring pipeline uses two methods: `score_company_relevance()` and the static helper `_strict_json_schema()`.

### 5.3.1 score_company_relevance()

**Method signature:**

```python
def score_company_relevance(self, enriched: EnrichedCompanyData) -> CompanyScoreResult:
```

**Implementation overview:**

```python
def score_company_relevance(self, enriched) -> object:
    from app.schemas.llm_scoring import LLMScoringOutput, CompanyScoreResult
    from app.constants.llm_scoring import SYSTEM_PROMPT

    user_message = self._build_scoring_user_message(enriched)

    response = self._client.responses.create(
        model=self.extraction_model,          # "gpt-4o-2024-08-06"
        tools=[{"type": "web_search_preview"}],
        text={
            "format": {
                "type": "json_schema",
                "name": "company_score",
                "schema": self._strict_json_schema(
                    LLMScoringOutput.model_json_schema()
                ),
                "strict": True,
            }
        },
        input=[
            {"role": "system", "content": SYSTEM_PROMPT},
            {"role": "user",   "content": user_message},
        ],
    )

    parsed = LLMScoringOutput.model_validate_json(response.output_text)

    web_search_used = any(
        getattr(item, "type", None) == "web_search_call"
        for item in (response.output or [])
    )
    ...
    return CompanyScoreResult(...)
```

**Key implementation details:**

- Uses the **OpenAI Responses API** (`client.responses.create`), not the Chat Completions API — this is required to pass `tools=[{"type": "web_search_preview"}]` alongside a structured JSON schema output format
- `model=self.extraction_model` resolves to `"gpt-4o-2024-08-06"` as configured in the resource
- Web search usage is detected by scanning `response.output` for any item with `type == "web_search_call"`, then recorded in `enrichment_used` and `web_search_used`
- `llm_justification` is capped at 4 bullets: `parsed.llm_justification[:4]`
- Raises `ValueError` if `response.output_text` is empty

### 5.3.2 _build_scoring_user_message()

The `_build_scoring_user_message()` method formats a structured text prompt from the enriched company data. The user message is distinct from the system prompt — it contains the company-specific data the model evaluates against the thesis criteria in `SYSTEM_PROMPT`.

**User message sections:**

1. Company name, location (city/state/country), stage, founding date
2. Description
3. Tags (formatted as `"TagValue (type)"`, sorted and deduplicated)
4. Company highlights (formatted as `"Category: text"`)
5. Funding block: total raised, stage, rounds, last funding date, investor names
6. Headcount
7. Employee highlights: summary counts by category, followed by individual detail lines
8. Enriched founder profiles (only present when Harmonic enrichment succeeded), including education, previous companies, and highlights per founder

The method gracefully handles missing or `None` values at every level — absent JSON fields default to `"Unknown"` or `"None"` strings rather than raising.

### 5.3.3 _strict_json_schema()

**Method signature:**

```python
@staticmethod
def _strict_json_schema(schema: dict) -> dict:
```

**Purpose:** OpenAI's strict JSON schema mode requires `additionalProperties: false` on every `object` node in the schema, including nested objects in `$defs`. Pydantic's `model_json_schema()` does not emit this field by default.

**Implementation:**

```python
@staticmethod
def _strict_json_schema(schema: dict) -> dict:
    import copy
    schema = copy.deepcopy(schema)

    def _fix(node):
        if not isinstance(node, dict):
            return
        if node.get("type") == "object":
            node.setdefault("additionalProperties", False)
        for v in node.values():
            if isinstance(v, dict):
                _fix(v)
            elif isinstance(v, list):
                for item in v:
                    _fix(item)

    _fix(schema)
    for defn in schema.get("$defs", {}).values():
        _fix(defn)
    return schema
```

**Usage:**

```python
self._strict_json_schema(LLMScoringOutput.model_json_schema())
```

This post-processed schema is passed as the `text.format.schema` argument to `client.responses.create`. Omitting this call causes OpenAI to reject the schema in strict mode and return a validation error, resulting in silent 0-company scoring runs.

---

## 5.4 Analyst Feedback Scaffold

Every scored company receives a deep copy of `ANALYST_FEEDBACK_SKELETON` stored in `Company.analyst_feedback` (JSONB). The skeleton provides a consistent structure for investment analysts to annotate the LLM's scoring decisions without modifying source code.

**`ANALYST_FEEDBACK_SKELETON`** (`app/constants/llm_scoring.py`):

```python
ANALYST_FEEDBACK_SKELETON = {
    "industry_fit": {
        "comment": None,
        "override_score": None,
        "submitted_by": None,
        "submitted_at": None,
    },
    "stage_fit": {
        "comment": None,
        "override_score": None,
        "submitted_by": None,
        "submitted_at": None,
    },
    "business_model": {
        "comment": None,
        "override_score": None,
        "submitted_by": None,
        "submitted_at": None,
    },
    "founder_strength": {
        "comment": None,
        "override_score": None,
        "submitted_by": None,
        "submitted_at": None,
    },
    "team_strength": {
        "comment": None,
        "override_score": None,
        "submitted_by": None,
        "submitted_at": None,
    },
    "investor_strength": {
        "comment": None,
        "override_score": None,
        "submitted_by": None,
        "submitted_at": None,
    },
    "highlights": {
        "comment": None,
        "override_score": None,
        "submitted_by": None,
        "submitted_at": None,
    },
    "overall": {
        "comment": None,
        "submitted_by": None,
        "submitted_at": None,
    },
}
```

**Per-dimension fields** (all seven scoring dimensions):

| Field | Type | Description |
|-------|------|-------------|
| `comment` | `str \| None` | Free-text analyst comment on this dimension |
| `override_score` | `int \| None` | Analyst's corrected score for the dimension (0–100) |
| `submitted_by` | `str \| None` | Analyst identifier (email or user ID) |
| `submitted_at` | `str \| None` | ISO 8601 timestamp of submission |

**`overall` entry fields** (no `override_score` — the overall score is not directly overridable at this level):

| Field | Type | Description |
|-------|------|-------------|
| `comment` | `str \| None` | Free-text overall investment thesis comment |
| `submitted_by` | `str \| None` | Analyst identifier |
| `submitted_at` | `str \| None` | ISO 8601 timestamp of submission |

**`copy.deepcopy()` is required:** `_build_company_update()` calls `copy.deepcopy(ANALYST_FEEDBACK_SKELETON)` to ensure each company receives a fully independent copy. Without a deep copy, mutations to one company's `analyst_feedback` dict would bleed into the shared skeleton object.

---

## 5.5 Key Files Reference

| File | Purpose |
|------|---------|
| `app/jobs/classify_ingested_companies.py` | 4-op job definition; `_build_company_update()` helper; `fetch_founder_profiles()` helper |
| `app/schemas/llm_scoring.py` | `CompanyForScoring`, `FounderProfile`, `EnrichedCompanyData`, `DimensionScore`, `LLMScoringOutput`, `CompanyScoreResult` |
| `app/constants/llm_scoring.py` | `SYSTEM_PROMPT` (v2.0), `PROMPT_VERSION`, `ANALYST_FEEDBACK_SKELETON`, `derive_rank()` |
| `app/resources/open_ai.py` | `score_company_relevance()`, `_build_scoring_user_message()`, `_strict_json_schema()` |
| `app/db/db_manager.py` | `fetch_unclassified_companies()`, `bulk_update_llm_scores()` |
| `app/resources/__init__.py` | `OpenAIResource`, `HarmonicResource`, `DatabaseResource` registration |

The design specification that drove this pipeline replacement is at `docs/superpowers/specs/2026-04-29-llm-scoring-modernization-design.md` in the umbrella workspace.
