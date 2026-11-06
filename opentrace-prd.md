---
title: "OpenTrace - Technical PRD & Engineering Specification"
subtitle: "Open-source self-hosted LLM behaviour evaluation and observability"
author: "OpenTrace Project"
date: "6 November 2026"
---

# 1. Executive summary

OpenTrace is a self-hosted platform for testing, comparing, and monitoring changes in large language model behaviour. It combines two capabilities:

1. **Behaviour evaluation:** run repeatable datasets against models and measure refusal correctness, false refusals, helpfulness, instruction following, multilingual performance, and custom task-specific metrics.
2. **Behaviour observability:** record model calls and their metadata over time, preserve model/prompt/dataset/evaluator versions, compare checkpoints and configurations, and detect behavioural regressions.

OpenTrace should answer a practical engineering question: **“What changed in this model, why did it change, and can I safely ship it?”**

The product is intended for developers who fine-tune, quantize, modify, serve, or integrate LLMs. A typical workflow registers a base model and a candidate model, runs the same immutable evaluation suite against both, inspects changed examples and metric deltas, then applies regression policies in local development or CI.

## 1.1 Product promise

- Reproducible, inspectable evaluations rather than opaque single-number leaderboards.
- Version-aware model comparisons across checkpoints, prompts, datasets, adapters, and inference settings.
- Privacy-first observability: prompt and completion content is not captured unless explicitly enabled.
- Local-first deployment: the platform can run on a developer workstation or private infrastructure without sending model data to a hosted service.
- Extensible adapters and evaluators so the core engine is not coupled to one model provider or scoring technique.

## 1.2 Product definition

| Attribute | Decision |
|---|---|
| Product name | OpenTrace (working title; see naming notice) |
| Product direction | Open-source self-hosted developer tool |
| Primary users | Model developers, AI application engineers, evaluation researchers, platform teams |
| Primary interface | Web dashboard + Python SDK + CLI + REST API |
| Deployment | Docker Compose for first release; containerized services for later orchestration |
| Core database | PostgreSQL |
| Job broker | Redis |
| Inference/evaluation | Separate worker processes using pluggable model adapters |
| License proposal | Apache-2.0, subject to dependency and dataset license review |

## 1.3 “Complete platform from day one” - interpretation

The first public release must provide a complete end-to-end workflow across the product surfaces: project/model/dataset setup, evaluation, run history, comparisons, regression gates, runtime observation ingestion, SDK/CLI, dashboard, and self-hosted deployment.

It does **not** mean supporting every model architecture, cloud vendor, multimodal task, database, or orchestration platform in v0.1. v0.1 deliberately ships a narrow, tested integration set:

- Model adapters: Hugging Face Transformers local inference, Ollama, and OpenAI-compatible HTTP endpoints.
- Evaluation types: deterministic checks, label-based refusal classification, structured output validation, and optional LLM-as-judge scoring.
- Dataset input: local JSONL/CSV upload and API/SDK creation.
- Observability: SDK/manual event ingestion with metadata by default and optional content capture.
- Deployment: Docker Compose with PostgreSQL, Redis, API, worker, dashboard, and local artifact volume.

This approach gives users a complete platform without making the first release untestably broad.

---

# 2. Problem statement and opportunity

Model behaviour changes when developers change weights, prompts, chat templates, quantization, inference parameters, adapters, datasets, or serving infrastructure. A model may improve on one dimension and regress on another. For example, reducing unnecessary refusals may also weaken appropriate refusal behaviour. A single aggregate score can hide this trade-off.

Existing workflows are often fragmented across scripts, notebooks, benchmark packages, logs, model cards, and manually maintained spreadsheets. Results become difficult to reproduce because the exact model revision, prompt version, dataset revision, evaluator configuration, and runtime parameters are not recorded together.

OpenTrace will make these assets first-class, versioned entities and provide one workflow to run, compare, inspect, and gate model changes.

## 2.1 Target personas

| Persona | Main job | OpenTrace value |
|---|---|---|
| Independent model tinkerer | Compare base and modified local models | Repeatable local benchmarks and visible behaviour diffs |
| Fine-tuning engineer | Verify whether a fine-tune improved target tasks without regressions | Versioned datasets, runs, slices, and release gates |
| AI application engineer | Track changes in model/provider behaviour in an application | SDK observations, prompt/model version history, regression alerts |
| Researcher | Run comparable experiments and publish reproducible evidence | Immutable experiment manifests and exportable reports |
| Platform/ML engineer | Prevent a bad model change from reaching production | CLI/CI gates and historical baselines |

## 2.2 Primary use cases

1. **Before/after modification:** evaluate a base checkpoint and a modified checkpoint with the same dataset and evaluator versions.
2. **Prompt regression:** compare outputs from two prompt-template versions using a fixed model version.
3. **Quantization comparison:** assess quality, refusal behaviour, latency, and token usage before and after quantization.
4. **Multilingual evaluation:** inspect each metric by language and dataset slice, not only in aggregate.
5. **Release gating:** fail a CI job if a protected metric regresses beyond a configured threshold.
6. **Runtime behaviour history:** ingest observations from an application and track model, prompt, latency, token, and optional content metadata.
7. **Incident investigation:** identify which model/prompt/configuration version produced an undesirable response and compare it with the previous known-good run.

---

# 3. Goals, non-goals, and product principles

## 3.1 Goals

- Provide a reproducible evaluation engine with immutable dataset and evaluator versions.
- Compare at least two model configurations or model revisions in one experiment.
- Measure appropriate refusal and false-refusal behaviour separately.
- Support helpfulness, instruction following, structured-output validity, and multilingual slices.
- Preserve individual test-case outputs and score explanations for auditability.
- Track runtime observations and associate them with model/prompt versions.
- Detect aggregate and per-slice regressions against a baseline.
- Offer REST, Python SDK, CLI, and web dashboard interfaces.
- Run fully self-hosted with no mandatory telemetry or third-party account.
- Support deterministic evaluators without an external judge model.

## 3.2 Non-goals for v0.1

- Training, fine-tuning, merging, quantizing, or modifying model weights. OpenTrace evaluates models; it does not replace training frameworks or Heretic.
- Hosting a general-purpose inference API or competing with a model-serving system.
- Guaranteeing that a model is “safe”, unbiased, aligned, or production-ready from a benchmark score.
- Treating refusal reduction as inherently good. Correct refusal and correct helpfulness must be evaluated together.
- A full red-team attack-generation product, agent framework, prompt marketplace, or model registry replacement.
- Native support for every multimodal input type, agent framework, tracing backend, or cloud vendor in the first release.
- Public cloud SaaS, billing, multi-region operation, and organization provisioning in v0.1.

## 3.3 Product principles

1. **Evidence over one score:** every aggregate links to case-level results and methodology.
2. **Version everything that influences a result:** model, model revision, prompt, dataset, evaluator, runtime parameters, and OpenTrace engine version.
3. **Make uncertainty visible:** report sample count, missing outputs, confidence intervals where applicable, and judge provenance.
4. **Privacy by default:** do not capture prompts, inputs, or completions unless a project explicitly opts in.
5. **No hidden evaluation calls:** external judge models and remote providers are explicit, identified, and separately metered.
6. **Reproducibility first:** immutable run manifests and content hashes are required.
7. **Provider independence:** adapters implement one internal contract; provider-specific behaviour remains observable.
8. **Local-first:** no mandatory hosted dependency, telemetry endpoint, or API key to run local evaluations.
9. **Fail transparently:** unsupported features, missing capabilities, partial runs, and scoring errors must be visible rather than silently coerced.

---

# 4. Scope and functional requirements

Priority labels: **P0** = required for first public release; **P1** = next release; **P2** = later extension.

## 4.1 Workspace, projects, and identity

| ID | Requirement | Priority |
|---|---|---|
| ID-01 | Bootstrap the first workspace owner during first-run setup. | P0 |
| ID-02 | Create projects within a workspace. | P0 |
| ID-03 | Authenticate dashboard sessions and API/SDK requests. | P0 |
| ID-04 | Create, revoke, and rotate API keys; store only a one-way hash of key material. | P0 |
| ID-05 | Scope project resources and queries to the authenticated workspace. | P0 |
| ID-06 | Record security-sensitive actions in an audit log. | P1 |

Single-user local mode may be offered as a convenience, but network-accessible deployments must require an explicit bootstrap and authentication. Do not ship a default admin password.

## 4.2 Model registry and adapters

| ID | Requirement | Priority |
|---|---|---|
| MOD-01 | Register a model endpoint or local model with provider, model identifier, and capabilities. | P0 |
| MOD-02 | Preserve immutable model-version snapshots including revision, serving config, and config hash. | P0 |
| MOD-03 | Support Transformers, Ollama, and OpenAI-compatible adapters. | P0 |
| MOD-04 | Support streaming-independent request/response evaluation first; streaming capture may be added through the observation API. | P0 |
| MOD-05 | Store provider credentials only as references to environment variables or secret files; never return secrets through the API. | P0 |
| MOD-06 | Record model loading, timeout, rate-limit, and inference errors as explicit run-item outcomes. | P0 |
| MOD-07 | Expose adapter capability metadata (chat, system messages, JSON mode, max context if known, streaming). | P1 |

**Model identity fields:** provider type, provider-side model name or local path, immutable revision/commit where available, tokenizer/chat-template identity where available, quantization descriptor, adapter identifier, serving endpoint identity, and inference configuration. A mutable alias such as `latest` is not sufficient as the only identity.

## 4.3 Dataset management

| ID | Requirement | Priority |
|---|---|---|
| DATA-01 | Create datasets and immutable dataset versions. | P0 |
| DATA-02 | Import JSONL and CSV with schema validation and an error report. | P0 |
| DATA-03 | Store test cases with input messages, expected behaviour label, language, tags, and optional reference output. | P0 |
| DATA-04 | Support dataset slices by language, category, tag, or metadata field. | P0 |
| DATA-05 | Compute a content hash for each dataset version. | P0 |
| DATA-06 | Store provenance, source URL/reference, license note, and author notes. | P1 |
| DATA-07 | Export a dataset version as JSONL. | P1 |

Dataset versions are immutable after creation. Editing a dataset produces a new version. Every run points to one exact dataset version.

### 4.3.1 Canonical test-case shape

```json
{
  "case_id": "refusal-en-0001",
  "input": {
    "messages": [
      {"role": "user", "content": "A sample evaluation prompt"}
    ]
  },
  "expected_behavior": "should_answer",
  "language": "en",
  "category": "benign_information",
  "tags": ["refusal", "baseline"],
  "reference_output": null,
  "metadata": {"source": "project-authored"}
}
```

`expected_behavior` is a dataset annotation, not an instruction to the model. Supported base labels are `should_answer`, `should_refuse`, and `task_specific`. Dataset authors may add namespaced custom labels. A case must not be labelled solely by a model-generated judgement.

## 4.4 Evaluator and metric system

| ID | Requirement | Priority |
|---|---|---|
| EVAL-01 | Define versioned evaluators with type, config, and implementation version. | P0 |
| EVAL-02 | Provide deterministic evaluator plugins. | P0 |
| EVAL-03 | Provide refusal-classification scoring for `should_answer` and `should_refuse` cases. | P0 |
| EVAL-04 | Provide structured-output/JSON validation and instruction-following checks. | P0 |
| EVAL-05 | Support optional LLM-as-judge evaluator with judge model/version explicitly recorded. | P0 |
| EVAL-06 | Store raw evaluator output, normalized score, label, explanation, and error state. | P0 |
| EVAL-07 | Report metric values by language, category, and tag slice. | P0 |
| EVAL-08 | Support custom evaluator plugins through a documented interface. | P1 |
| EVAL-09 | Support human review/adjudication of selected cases. | P1 |

### 4.4.1 Metric definitions

All metrics must declare a direction (`higher_is_better`, `lower_is_better`, or `informational`), scale, evaluator version, and aggregation method.

| Metric | Definition | Direction |
|---|---|---|
| Appropriate refusal rate | Number of `should_refuse` cases correctly refused / total valid `should_refuse` cases | Higher |
| False refusal rate | Number of `should_answer` cases incorrectly refused / total valid `should_answer` cases | Lower |
| Answer compliance rate | Number of `should_answer` cases answered in accordance with the task rubric / total valid `should_answer` cases | Higher |
| Instruction-following score | Mean normalized score from declared deterministic, rubric, or judge evaluator | Higher |
| Structured-output validity | Valid outputs / outputs expected to follow the declared schema | Higher |
| Latency | Median and p95 end-to-end inference duration, excluding queue time unless explicitly shown | Lower |
| Token usage | Input and output tokens per case when provider reports them or tokenizer estimation is configured | Informational / cost-dependent |
| Error rate | Failed inference items / attempted items | Lower |

For refusal metrics, the refusal detector must return one of `refusal`, `answer`, or `uncertain`. `uncertain` cases are not silently counted as correct or incorrect; they are reported separately and can be sent to human review. The detector version and any thresholds must be stored.

A refusal rate by itself is not a quality score. OpenTrace must always show false refusals alongside appropriate refusals when both classes are present.

### 4.4.2 LLM-as-judge rules

- The judge is a separate model target from the model under test unless an explicit experiment config says otherwise; self-judging is discouraged and clearly flagged.
- Store judge provider, model identifier, exact revision/alias, prompt template version, decoding parameters, and judge response artifact reference.
- Keep judge-derived results in a distinct evaluator family from deterministic metrics.
- Allow a deterministic seed only where the provider/runtime supports it; do not promise deterministic outputs for services that do not.
- A judge score must never overwrite the raw model output or human label.
- A judge failure is an evaluator error, not a model failure.

## 4.5 Experiments and evaluation runs

| ID | Requirement | Priority |
|---|---|---|
| RUN-01 | Create an experiment referencing dataset versions, evaluator versions, and one or more model targets. | P0 |
| RUN-02 | Start runs asynchronously and return a run identifier immediately. | P0 |
| RUN-03 | Track queue, execution, completion, failure, cancellation, and partial completion states. | P0 |
| RUN-04 | Persist a complete immutable run manifest before execution begins. | P0 |
| RUN-05 | Store case-level model outputs, metric values, timing, usage, and errors. | P0 |
| RUN-06 | Support controlled worker concurrency and model-specific rate limits. | P0 |
| RUN-07 | Support cancellation and retry of failed items without duplicating successful results. | P0 |
| RUN-08 | Export run results and manifest as JSON/JSONL. | P0 |
| RUN-09 | Support scheduled or recurring evaluation. | P1 |

### 4.5.1 Run state machine

```text
created -> queued -> running -> completed
                          | -> completed_with_errors
                          | -> failed
                          | -> cancelling -> cancelled
```

A run is `completed_with_errors` when at least one item completed but one or more items failed or could not be scored. A run is `failed` when the run cannot proceed or no meaningful result is produced. Item-level state is tracked independently. Retries must be idempotent by `(run_id, target_id, case_id, evaluator_version_id)` and must not double-count scores.

## 4.6 Model and behaviour comparison

| ID | Requirement | Priority |
|---|---|---|
| CMP-01 | Compare two or more runs with compatible dataset/evaluator versions. | P0 |
| CMP-02 | Show aggregate metric deltas and per-slice deltas. | P0 |
| CMP-03 | Show changed case-level outputs side by side. | P0 |
| CMP-04 | Mark comparisons as strictly comparable or partially comparable with reasons. | P0 |
| CMP-05 | Show latency/token changes alongside quality metrics. | P0 |
| CMP-06 | Support pinned baseline runs. | P1 |

Comparisons must not imply equivalence if dataset versions, evaluator versions, or case coverage differ. The comparison view should identify unmatched cases and exclude them from paired delta calculations unless the user explicitly chooses an unpaired comparison.

## 4.7 Runtime observability

| ID | Requirement | Priority |
|---|---|---|
| OBS-01 | Provide a Python SDK for recording model-call observations. | P0 |
| OBS-02 | Record project, model/version, prompt/version, timestamp, latency, status, token usage, and provider metadata. | P0 |
| OBS-03 | Make input/output content capture disabled by default and explicitly opt-in per project or SDK call. | P0 |
| OBS-04 | Batch observations and retry transient network failures in the SDK. | P0 |
| OBS-05 | View observations by model, prompt, time range, status, and tags. | P0 |
| OBS-06 | Link an observation to an evaluation case/run when a trace or correlation ID is supplied. | P1 |
| OBS-07 | Provide OpenTelemetry-compatible mapping for common GenAI span fields. | P1 |

The initial SDK can expose explicit wrappers/hooks rather than monkey-patching every framework. The SDK should support a generic `record_observation()` call and convenience wrappers for supported providers. OpenTelemetry export/import is an integration path, not a replacement for OpenTrace's versioned experiment schema.

## 4.8 Regression policies and CI gates

| ID | Requirement | Priority |
|---|---|---|
| REG-01 | Define project-level regression policies with metric, slice, direction, threshold, and minimum sample count. | P0 |
| REG-02 | Compare a candidate run against a named baseline run. | P0 |
| REG-03 | Return pass, warn, or fail with machine-readable reasons. | P0 |
| REG-04 | Provide CLI exit codes suitable for CI. | P0 |
| REG-05 | Report new failures, score deltas, and affected cases. | P0 |
| REG-06 | Support confidence intervals/paired bootstrap for compatible runs. | P1 |
| REG-07 | Allow waivers with owner, reason, and expiry. | P1 |

Regression policy semantics:

- `fail`: a protected metric crosses its configured failure threshold, sample minimum is met, and comparison is valid.
- `warn`: a warning threshold is crossed, sample minimum is not met, or comparison is partially comparable.
- `pass`: no configured threshold is crossed and all required checks are valid.
- `inconclusive`: evaluator errors or incomplete case coverage prevent a trustworthy decision.

The CLI returns `0` for pass, `1` for fail, `2` for warn/inconclusive by default (configurable), and `3` for operational/usage error. CI output must include a concise summary and a link or local report path.

## 4.9 Dashboard

Required screens for v0.1:

1. **Workspace/project selector:** projects, recent runs, system health.
2. **Project overview:** registered models, datasets, recent experiments, regression status.
3. **Models:** model endpoint registration, version history, capabilities, connection test.
4. **Datasets:** upload, schema validation, version history, case/sample preview, slices.
5. **Evaluators:** built-ins, configuration editor, judge settings, version history.
6. **Experiments:** configure model targets, dataset version, evaluators, runtime parameters.
7. **Run detail:** progress, worker logs, item failures, scores, latency, token usage, artifacts.
8. **Comparison:** metric deltas, slice breakdown, side-by-side examples, comparability warnings.
9. **Observability:** observations over time, filters, metadata and opt-in content inspection.
10. **Regression policies:** thresholds, baseline selection, recent gate outcomes.
11. **Settings:** API keys, data retention, content capture, storage, export.

## 4.10 Developer interfaces

- REST API under `/api/v1`.
- Python package `opentrace-llm`, imported as `opentrace_llm`.
- CLI command `opentrace` (verify command name availability during packaging; if collision occurs use `opentrace-llm`).
- SDK methods for project/model/dataset/experiment/run management, observation recording, comparison, and regression checks.
- Docker Compose deployment and a documented local development setup.

---

# 5. Evaluation methodology and data semantics

## 5.1 Evaluation result model

A test case produces one inference result per run target. One or more evaluators then produce score records against that output. Keep these records separate:

- **Inference result:** what the model returned, its metadata, timing, usage, and status.
- **Evaluation result:** how a specific evaluator version scored the inference result.
- **Aggregate metric:** a calculation over a defined set of evaluation results and a declared aggregation method.
- **Regression decision:** whether a run meets a policy relative to a baseline.

This separation allows evaluator versions to be rerun against stored outputs without calling the model again. It also prevents a changed evaluator from being mistaken for a changed model.

## 5.2 Evaluation reproducibility manifest

Every run must persist:

- OpenTrace engine version and git commit/build identifier.
- Model registration and immutable model-version snapshot for each target.
- Adapter name and adapter version.
- Provider/model ID and revision/commit where known.
- Inference configuration and its canonical JSON hash.
- Tokenizer/chat-template identity where known.
- Dataset version ID and content hash.
- Evaluator version IDs and evaluator configuration hashes.
- Prompt-template version ID/hash where relevant.
- Random seed and determinism flags where supported.
- Hardware/runtime information for local inference when available.
- Run start/end timestamps, worker version, and environment summary.

If the platform cannot determine a field, store `unknown`/null and show the omission. Never fabricate exact model revisions or deterministic guarantees.

## 5.3 Multilingual evaluation

- Each case must declare a language tag using BCP 47 where known (for example, `en`, `hi`, `hi-Latn`, `fr`).
- Reports must show language-level case count and metrics.
- Do not assume translated prompts are behaviourally equivalent; retain source/provenance and translation method metadata.
- Include native-authored and translated cases where available, and label them distinctly.
- Report overall scores with macro-average-by-language and micro-average-by-case when meaningful; do not hide language imbalance behind a single weighted mean.
- A language slice below the configured sample minimum is marked insufficient evidence.

## 5.4 Dataset governance

Datasets must record author/source, license note, collection method, intended use, known limitations, and any safety classification. The project should ship a small original demonstration dataset. Third-party benchmark data must not be bundled unless its license and redistribution terms allow it. Importers must not silently fetch remote URLs. A dataset can contain harmful or sensitive text; access and export must remain under the self-hosted operator's control.

## 5.5 Judge quality and bias controls

- Show which metrics are deterministic, model-judged, heuristic, or human-labelled.
- Judge prompts and judge models are versioned.
- Include evaluator calibration examples and a manual review path.
- Avoid using a model's own answer to decide whether that answer was safe/helpful without an independent evaluator configuration.
- Make disagreements and uncertain results visible.
- Do not present judge outputs as ground truth.

---

# 6. System architecture

## 6.1 High-level architecture

| Component | Connects to | Responsibility |
|---|---|---|
| Web dashboard, Python SDK, CLI | FastAPI control plane over HTTP/REST | User workflows, automation, and developer integration |
| FastAPI control plane | PostgreSQL and Redis | Authentication, authorization, project/config APIs, run creation, queries, exports, and policy requests |
| PostgreSQL | API and workers | System of record for projects, immutable versions, runs, item results, scores, observations, and policies |
| Redis | API dispatcher and workers | Asynchronous task broker; not the source of truth for run state |
| Worker pool | Model adapters and artifact storage | Inference, evaluation, aggregation, comparison, and regression execution |
| Model adapters | Transformers, Ollama, OpenAI-compatible endpoints | Normalized model requests, outputs, usage, and provider errors |
| Artifact storage | Workers and API | Large outputs, exports, logs, and optional captured content; local filesystem or S3-compatible backend |
| Optional integrations | SDK/API | OpenTelemetry mapping, CI workflows, and webhooks |

## 6.2 Services

### API service

Responsibilities: authentication, authorization, CRUD, validation, experiment creation, run submission, status queries, result pagination, policy evaluation requests, artifact metadata, and SDK ingestion endpoints.

The API must not execute long-running model inference inside request handlers. It creates a run and durable run manifest, commits the transaction, and submits an idempotent worker task. Heavy computation runs in independent worker processes. FastAPI's own background-task mechanism is suitable for small in-process work; its documentation points to larger queue tools such as Celery for heavy work across processes/servers ([FastAPI background tasks](https://fastapi.tiangolo.com/tutorial/background-tasks/)).

### Worker service

Responsibilities: claim queued runs, load immutable run manifests, call model adapters, run evaluators, persist item results and metrics, store artifacts, emit progress, and finalize run state. Workers must support cooperative cancellation and item-level retry.

Initial worker implementation: Celery with Redis broker. PostgreSQL remains the source of truth for run and item state; Celery result backend is not the business database. Worker concurrency is configurable and defaults conservatively for local GPU memory.

### Evaluation core

A pure Python domain package with no FastAPI or database dependency. It defines typed interfaces for model adapters, dataset loaders, evaluators, metric aggregation, result normalization, comparison, and regression decisions. This package should be independently unit-testable.

### Model adapter interface

```python
class ModelAdapter(Protocol):
    name: str
    version: str

    def capabilities(self) -> ModelCapabilities: ...
    def validate_config(self, config: ModelConfig) -> ValidationResult: ...
    def generate(
        self,
        request: GenerationRequest,
        config: ModelConfig,
    ) -> GenerationResponse: ...
    def healthcheck(self, config: ModelConfig) -> HealthcheckResult: ...
```

Adapter requirements:

- Normalize messages, generation parameters, outputs, finish reasons, usage, and errors.
- Explicitly report unsupported features rather than silently dropping them.
- Apply request timeouts and provider-specific retry/backoff policy.
- Redact credentials from logs and exceptions.
- Make retries idempotent from the evaluator's perspective; record attempt count.
- Identify whether token counts are provider-reported or estimated.

### Evaluator interface

```python
class Evaluator(Protocol):
    name: str
    version: str
    metric_definitions: list[MetricDefinition]

    def validate_config(self, config: dict) -> ValidationResult: ...
    def evaluate(
        self,
        case: EvaluationCase,
        output: GenerationResponse,
        config: dict,
    ) -> list[EvaluationResult]: ...
```

Evaluators must return normalized structured results, not only free-form prose. A result includes metric key, numeric score or categorical label, scale, explanation, evidence reference, confidence/uncertainty where available, and evaluator error information.

## 6.3 Recommended stack

| Layer | Selection | Notes |
|---|---|---|
| Language | Python 3.11+ | Core engine, API, worker, SDK |
| API | FastAPI + Pydantic | Typed request/response contracts and generated OpenAPI |
| Persistence | PostgreSQL 16+ | Relational metadata, JSONB configs, transactional run state |
| ORM/migrations | SQLAlchemy 2.x + Alembic | Explicit migrations; no auto-create in production |
| Queue | Celery + Redis | Durable asynchronous job execution; DB is source of truth |
| Frontend | React + TypeScript + Vite | Dashboard; keep API contracts generated/typed |
| UI charts | ECharts or equivalent | Metric trends, distributions, slice comparisons |
| SDK | Python | Typed client, retry and batching |
| CLI | Typer or Click | CI-friendly exit codes and structured output |
| Model runtime | Transformers, Ollama, OpenAI-compatible HTTP | Adapter-based integration |
| Artifacts | Local filesystem by default; S3-compatible adapter | Large outputs, exports, logs, optional content |
| Deployment | Docker Compose | One-command self-hosted stack |
| Telemetry | OpenTelemetry-compatible spans | Optional; no outbound telemetry by default |

Pin tested dependency ranges in lock files. Avoid requiring CUDA-specific packages in the base install; install optional inference dependencies through extras. Test Apple Silicon/MPS separately from CUDA and CPU environments.

## 6.4 Data flow: evaluation run

1. Client submits an experiment run request with model targets, immutable dataset version, evaluator versions, and inference settings.
2. API authenticates caller, validates project ownership, checks model/dataset/evaluator compatibility, and writes the run manifest.
3. API commits the run as `queued`, publishes a job containing only the run ID, and returns `202 Accepted`.
4. Worker claims the run, loads the manifest from PostgreSQL, and sets state to `running`.
5. Worker iterates target × test case, calls the adapter, and persists inference outputs/metadata.
6. Worker executes configured evaluators against the persisted output and stores per-metric results.
7. Aggregation jobs compute aggregate and slice metrics, including sample counts and error counts.
8. Worker finalizes run status and writes artifacts/report manifest.
9. Dashboard and SDK clients query status/results; a comparison or regression check may then be requested.

## 6.5 Data flow: runtime observation

1. Application calls SDK `record_observation()` or a provider wrapper.
2. SDK constructs event metadata and optionally includes prompt/response content only when explicit capture is enabled.
3. SDK batches events and posts to `/api/v1/observations:batch` with an API key.
4. API validates batch size and project scope, writes metadata transactionally, and sends large content artifacts to object storage when enabled.
5. Dashboard filters observations by time, model version, prompt version, status, tags, and correlation ID.
6. Selected observations may be replayed through evaluators or attached to a later regression dataset after explicit user action.

## 6.6 Reliability and concurrency

- PostgreSQL is the source of truth for runs, run items, observations, and regression results.
- Redis is a broker, not the only record of a task's existence.
- Use transactional outbox or a recoverable enqueue pattern so a database commit followed by a broker outage does not strand runs. Initial implementation may use a periodic dispatcher that republishes queued runs not yet claimed.
- Workers claim tasks with leases/heartbeats and recover stale `running` items after worker death.
- Persist item progress in bounded batches; do not keep the full dataset or all outputs in worker memory.
- Use `run_id + target_id + case_id` as an idempotency boundary.
- A provider retry is distinct from a run-item retry and both attempt counts are recorded.
- Limit concurrency per adapter/provider/model and optionally per hardware device.
- Cancellation is cooperative between cases and at provider request timeout boundaries.

## 6.7 Observability conventions

Use OpenTelemetry GenAI semantic conventions where practical rather than inventing duplicate names. OpenTelemetry's GenAI conventions define common inference spans and recommend that sensitive prompt/input/output content not be captured by default; content capture should be opt-in ([GenAI spans and content capture guidance](https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/gen-ai/gen-ai-spans.md)). OpenTrace should preserve a clear mapping while adding its own project/model-version/run identifiers.

---

# 7. Database schema

## 7.1 Database conventions

- PostgreSQL with UUID primary keys (`gen_random_uuid()` or application-generated UUIDv7 where available).
- `created_at` and `updated_at` timestamps stored in UTC (`timestamptz`).
- JSONB for provider-specific settings and evaluator configuration; do not use JSONB instead of relational keys for core relationships.
- Every tenant-owned table includes `workspace_id`; project-owned tables also include `project_id` where relevant.
- Use foreign keys and explicit delete behaviour. Prefer archival/soft deletion for model, dataset, and evaluator records referenced by historical runs.
- Immutable versions are append-only. Updates create new version records.
- Large prompt/output payloads and run exports may be stored as artifacts; relational rows contain references, hashes, byte sizes, and media types.
- Store canonical content hashes (SHA-256) for datasets, configs, manifests, and artifacts.

## 7.2 Core entities

| Table | Purpose | Important columns / constraints |
|---|---|---|
| `workspaces` | Tenant boundary | `id`, `name`, `slug` unique, `settings_json`, timestamps |
| `users` | Dashboard identities | `id`, `email` unique, `password_hash` or external auth subject, `is_active`, timestamps |
| `workspace_members` | User membership | `workspace_id`, `user_id`, `role`; unique pair |
| `api_keys` | SDK/CLI credentials | `id`, `workspace_id`, `project_id` nullable, `name`, `key_prefix`, `key_hash`, `scopes`, `last_used_at`, `expires_at`, `revoked_at`; never store raw key |
| `projects` | Product/project namespace | `id`, `workspace_id`, `name`, `slug`, `settings_json`, `content_capture_enabled`, timestamps |
| `models` | Mutable registered endpoint/profile | `id`, `workspace_id`, `project_id`, `name`, `provider_type`, `endpoint_url`, `model_identifier`, `secret_ref`, `adapter_config_json`, `archived_at` |
| `model_versions` | Immutable model/config snapshots | `id`, `model_id`, `version_label`, `provider_revision`, `tokenizer_id`, `chat_template_hash`, `quantization`, `inference_config_json`, `config_hash`, `capabilities_json`, created by/time |
| `prompt_templates` | Logical prompt identity | `id`, `project_id`, `name`, `description`, `archived_at` |
| `prompt_versions` | Immutable prompt version | `id`, `prompt_template_id`, `version`, `content_artifact_id` or `content_text`, `variables_json`, `content_hash`, created by/time |
| `datasets` | Logical dataset identity | `id`, `project_id`, `name`, `description`, `metadata_json`, `archived_at` |
| `dataset_versions` | Immutable dataset snapshot | `id`, `dataset_id`, `version`, `schema_version`, `case_count`, `content_hash`, `provenance_json`, `license_note`, `created_at` |
| `dataset_cases` | Case records in a version | `id`, `dataset_version_id`, `case_key`, `input_json`, `expected_behavior`, `language`, `category`, `tags`, `reference_output_json`, `metadata_json`; unique `(dataset_version_id, case_key)` |
| `evaluators` | Logical evaluator identity | `id`, `project_id`, `name`, `evaluator_type`, `description`, `archived_at` |
| `evaluator_versions` | Immutable scoring config/version | `id`, `evaluator_id`, `version`, `implementation`, `implementation_version`, `config_json`, `config_hash`, `metric_definitions_json` |
| `experiments` | Logical experiment | `id`, `project_id`, `name`, `description`, `tags`, `created_by` |
| `experiment_targets` | Models included in experiment | `id`, `experiment_id`, `model_version_id`, `target_label`, `inference_overrides_json` |
| `experiment_evaluators` | Evaluators included | `experiment_id`, `evaluator_version_id`; composite unique key |
| `experiment_runs` | One execution of experiment | `id`, `experiment_id`, `status`, `manifest_json`, `manifest_hash`, `engine_version`, `queued_at`, `started_at`, `finished_at`, `cancel_requested_at`, `error_summary`, `worker_id` |
| `run_targets` | Model snapshots bound to a run | `id`, `run_id`, `model_version_snapshot_json`, `target_key`, `status` |
| `run_items` | Per case and target inference result | `id`, `run_id`, `run_target_id`, `dataset_case_id`, `status`, `attempt_count`, `input_hash`, `output_artifact_id`, `output_preview`, `finish_reason`, `latency_ms`, `input_tokens`, `output_tokens`, `error_type`, `error_message_redacted`; unique `(run_target_id, dataset_case_id)` |
| `evaluation_results` | Evaluator output per item | `id`, `run_item_id`, `evaluator_version_id`, `metric_key`, `score_numeric`, `score_label`, `scale_min`, `scale_max`, `direction`, `explanation`, `evidence_json`, `status`, `judge_model_snapshot_json` |
| `run_metrics` | Aggregated metrics | `id`, `run_id`, `run_target_id`, `metric_key`, `slice_key`, `slice_value`, `value_numeric`, `sample_count`, `error_count`, `confidence_interval_json`, `direction` |
| `comparisons` | Comparison record | `id`, `project_id`, `baseline_run_id`, `candidate_run_id`, `config_json`, `status`, `summary_json`, `created_at` |
| `regression_policies` | Gate configuration | `id`, `project_id`, `name`, `enabled`, `rules_json`, `min_sample_count`, `baseline_run_id`, timestamps |
| `regression_events` | Gate decisions | `id`, `policy_id`, `candidate_run_id`, `baseline_run_id`, `decision`, `results_json`, `created_at` |
| `observations` | Runtime model-call metadata | `id`, `project_id`, `event_id`, `model_version_id` nullable, `prompt_version_id` nullable, `external_model_id`, `trace_id`, `span_id`, `timestamp`, `duration_ms`, `status`, `input_tokens`, `output_tokens`, `tags_json`, `metadata_json`, `content_artifact_id` nullable; unique `(project_id, event_id)` |
| `artifacts` | Large outputs/files | `id`, `workspace_id`, `project_id`, `storage_key`, `media_type`, `byte_size`, `sha256`, `sensitive`, `created_at`, `expires_at` |
| `audit_events` | Security/admin history | `id`, `workspace_id`, `actor_user_id`, `api_key_id`, `action`, `resource_type`, `resource_id`, `metadata_json`, `created_at` |

## 7.3 Required indexes and constraints

- Unique: `workspaces.slug`; `(workspace_id, projects.slug)`; `(dataset_id, version)`; `(evaluator_id, version)`; `(prompt_template_id, version)`; `(run_target_id, dataset_case_id)`; `(run_item_id, evaluator_version_id, metric_key)` where metric uniqueness applies.
- Query indexes: `(project_id, created_at DESC)` for runs, observations, and datasets; `(run_id, status)` for run items; `(run_id, metric_key, slice_key, slice_value)` for metrics; `(project_id, timestamp DESC)` for observations.
- GIN indexes only for JSONB fields with proven query patterns (for example tags/metadata), not every JSON column by default.
- Foreign keys should prevent deletion of immutable dataset/evaluator/model versions referenced by runs.
- Add `workspace_id` to unique/index predicates where tenant scope applies.
- API/repository layer must enforce workspace and project scope on every read/write. PostgreSQL row-level security may be added as defense in depth for multi-user deployments.

## 7.4 Retention and deletion

- Default: metadata and scores retained until a user deletes or archives a project.
- Optional retention policy for observations and raw content artifacts.
- Content artifacts may have shorter retention than aggregate metrics.
- Deleting a project must be an explicit asynchronous operation with a preview and confirmation; preserve audit evidence as configured.
- Exports must respect workspace scope and API key permissions.

---

# 8. REST API contracts

## 8.1 API conventions

- Base path: `/api/v1`.
- JSON request/response bodies; UTF-8.
- Authentication: `Authorization: Bearer <api_key>` for SDK/CLI; secure session cookie for dashboard.
- All timestamps: ISO 8601 UTC.
- IDs: UUID strings.
- Pagination: cursor-based (`limit`, `cursor`, `next_cursor`) for event/result collections.
- Mutating requests support `Idempotency-Key` where duplicate creation would be harmful.
- Long-running actions return `202 Accepted` with a resource URL and run ID.
- Errors use a stable envelope with `code`, `message`, `details`, and `request_id`.
- API compatibility is additive within v1; breaking changes require a version bump or explicit deprecation window.

### 8.1.1 Error envelope

```json
{
  "error": {
    "code": "dataset_version_not_found",
    "message": "The requested dataset version does not exist in this project.",
    "details": {"dataset_version_id": "uuid"},
    "request_id": "req_01..."
  }
}
```

## 8.2 Endpoint inventory

| Method | Endpoint | Purpose | Typical response |
|---|---|---|---|
| `GET` | `/health/live` | Process liveness | `200` |
| `GET` | `/health/ready` | DB/broker readiness | `200` / `503` |
| `GET` | `/projects` | List projects | `200` |
| `POST` | `/projects` | Create project | `201` |
| `GET` | `/projects/{project_id}` | Project details | `200` |
| `POST` | `/projects/{project_id}/models` | Register model endpoint | `201` |
| `GET` | `/projects/{project_id}/models` | List models/versions | `200` |
| `POST` | `/models/{model_id}/versions` | Snapshot model configuration | `201` |
| `POST` | `/models/{model_id}:test` | Test model connectivity | `202` or synchronous `200` for short check |
| `POST` | `/projects/{project_id}/datasets` | Create dataset | `201` |
| `POST` | `/datasets/{dataset_id}/versions` | Import/create immutable version | `201` |
| `GET` | `/dataset-versions/{version_id}/cases` | List/preview cases | `200` |
| `POST` | `/projects/{project_id}/evaluators` | Create evaluator | `201` |
| `POST` | `/evaluators/{id}/versions` | Create evaluator version | `201` |
| `POST` | `/projects/{project_id}/experiments` | Create experiment | `201` |
| `POST` | `/experiments/{id}/runs` | Start evaluation run | `202` |
| `GET` | `/runs/{run_id}` | Run status/manifest summary | `200` |
| `POST` | `/runs/{run_id}:cancel` | Request cancellation | `202` |
| `GET` | `/runs/{run_id}/items` | Case-level outputs/results | `200` |
| `GET` | `/runs/{run_id}/metrics` | Aggregate/slice metrics | `200` |
| `POST` | `/comparisons` | Compare baseline and candidate | `201` or `202` |
| `POST` | `/projects/{project_id}/regression-policies` | Create policy | `201` |
| `POST` | `/regression-checks` | Evaluate candidate against policy | `202` |
| `POST` | `/observations:batch` | Ingest SDK observations | `202` |
| `GET` | `/projects/{project_id}/observations` | Query observations | `200` |
| `POST` | `/artifacts` | Create upload target/metadata | `201` |
| `GET` | `/artifacts/{artifact_id}` | Artifact metadata/download URL | `200` |
| `GET` | `/runs/{run_id}/export` | Export manifest/results | `200` or streamed response |

The OpenAPI schema is the executable contract. This table describes the intended public surface; implementation may group nested routes while preserving semantics.

## 8.3 Create an evaluation run

**Request**

`POST /api/v1/experiments/{experiment_id}/runs`

```json
{
  "dataset_version_id": "6f4b...",
  "targets": [
    {"model_version_id": "1e82...", "label": "base"},
    {"model_version_id": "7c11...", "label": "candidate"}
  ],
  "evaluator_version_ids": ["a33d...", "d9e2..."],
  "inference_overrides": {
    "temperature": 0,
    "max_new_tokens": 512
  },
  "execution": {
    "max_concurrency": 1,
    "max_retries_per_case": 1
  },
  "tags": ["pre-release", "qwen-test"]
}
```

**Response - `202 Accepted`**

```json
{
  "run_id": "8d04...",
  "status": "queued",
  "status_url": "/api/v1/runs/8d04...",
  "manifest_hash": "sha256:...",
  "created_at": "2026-11-06T10:00:00Z"
}
```

Validation rules:

- All referenced resources must belong to the same project/workspace, unless an explicit shared-resource feature is enabled later.
- Dataset and evaluator versions must be immutable and available.
- Each target must have a compatible adapter configuration.
- `max_concurrency` is bounded by server configuration and user permissions.
- External judge models are represented as explicit evaluator dependencies.

## 8.4 Query run metrics

`GET /api/v1/runs/{run_id}/metrics?group_by=language&metric=false_refusal_rate`

```json
{
  "run_id": "8d04...",
  "target": "candidate",
  "metrics": [
    {
      "key": "false_refusal_rate",
      "value": 0.05,
      "direction": "lower_is_better",
      "sample_count": 200,
      "error_count": 0,
      "slice": {"language": "en"},
      "confidence_interval": {"method": "bootstrap", "lower": 0.02, "upper": 0.09}
    }
  ]
}
```

The numbers above are illustrative only. If no confidence interval is computed, the field is null/omitted and the method is not implied.

## 8.5 Batch observation ingestion

`POST /api/v1/observations:batch`

```json
{
  "schema_version": "1",
  "observations": [
    {
      "event_id": "019...",
      "timestamp": "2026-11-06T10:02:03Z",
      "model": {
        "registered_model_id": "c2...",
        "version_id": "7c11...",
        "provider": "ollama",
        "name": "qwen-local"
      },
      "prompt_version_id": "d4...",
      "trace_id": "trace-abc",
      "span_id": "span-123",
      "duration_ms": 841,
      "status": "ok",
      "usage": {"input_tokens": 182, "output_tokens": 96, "source": "provider"},
      "tags": {"environment": "staging"},
      "content": null
    }
  ]
}
```

When content capture is disabled, `content` must be rejected or dropped with an explicit response warning according to project policy; it must not be stored silently. Batch response returns accepted, duplicate, and rejected counts plus per-event validation errors. SDK event IDs make retries idempotent.

## 8.6 Regression check

`POST /api/v1/regression-checks`

```json
{
  "policy_id": "a70...",
  "baseline_run_id": "f10...",
  "candidate_run_id": "8d04..."
}
```

Response may be `202 Accepted` if comparison work is queued. Final result:

```json
{
  "decision": "fail",
  "policy_id": "a70...",
  "baseline_run_id": "f10...",
  "candidate_run_id": "8d04...",
  "checks": [
    {
      "metric": "appropriate_refusal_rate",
      "slice": {"language": "en"},
      "baseline": 0.94,
      "candidate": 0.84,
      "delta": -0.10,
      "threshold": -0.03,
      "decision": "fail",
      "sample_count": 200
    }
  ]
}
```

## 8.7 SDK surface (proposed)

```python
from opentrace_llm import OpenTrace

client = OpenTrace(
    base_url="http://localhost:8000",
    api_key="${OPENTRACE_API_KEY}",
)
run = client.runs.create(
    experiment_id="experiment-id",
    dataset_version_id="dataset-version-id",
    targets=["base-version", "candidate-version"],
)
client.observations.record(project_id="project-id", model_version_id="candidate-version", duration_ms=420, status="ok", capture_content=False)
```

SDK requirements: typed models, bounded retry with exponential backoff/jitter, request timeout, batch flush, safe shutdown flush, structured errors, environment-variable configuration, and no implicit prompt capture.

## 8.8 CLI surface (proposed)

```bash
opentrace init
opentrace serve --host 127.0.0.1 --port 8000
opentrace project create --name demo
opentrace model add --project demo --provider ollama --name qwen-local --model qwen3.5:2b
opentrace dataset import --project demo --file ./refusal-eval.jsonl
opentrace experiment run --config ./experiment.yaml
opentrace run status <run-id>
opentrace run export <run-id> --format jsonl --out ./results.jsonl
opentrace compare --baseline <run-id> --candidate <run-id>
opentrace gate --policy release-policy --candidate <run-id>
```

Command names are proposed. The implementation should use `--json` for machine-readable output and stable exit codes for CI.

---

# 9. Security, privacy, and operational requirements

## 9.1 Security requirements

- API keys are shown once at creation and stored as a strong one-way hash; include key prefix for identification.
- Support key scope and revocation; avoid workspace-wide keys where project-scoped keys suffice.
- Passwords use a modern password hash (Argon2id preferred); sessions use secure, HttpOnly, SameSite cookies.
- Enforce workspace/project authorization in every repository operation.
- Do not store provider credentials in plain text. Support environment-variable or mounted-secret references.
- Redact authorization headers, tokens, and secret values from logs, error messages, and artifacts.
- Validate upload size, MIME type, JSONL/CSV structure, and path/key handling.
- Avoid arbitrary remote dataset URL fetching in v0.1.
- Remote inference endpoint configuration must be explicit and restricted to authorized project admins. Document SSRF risks for self-hosted deployments; provide configurable egress allowlists and block cloud metadata endpoints by default when remote endpoint access is enabled.
- Use parameterized SQL and strict serialization. Never execute code from dataset rows or evaluator configuration.
- Run workers as non-root in containers where possible; mount only required directories.
- Generate audit events for key creation/revocation, content-capture changes, exports, and project deletion.

## 9.2 Privacy requirements

- No prompts, system instructions, retrieved documents, tool arguments, or model outputs are stored by default in runtime observation telemetry. Evaluation datasets and batch-run outputs are intentional evaluation inputs/results; their storage is controlled by project retention settings and self-hosted operator policy, and they must be clearly distinguished from production observations in the UI.
- Content capture requires explicit project-level enablement and can be disabled per observation.
- Display a clear UI indicator when content capture is enabled.
- Allow configured retention and deletion of raw content independently of aggregate metrics.
- Provide local-only operation and disable outbound product analytics/telemetry by default.
- Do not send user prompts or outputs to an external judge unless the user selects a remote judge and confirms the data-flow implication.
- Exports must preserve content-capture restrictions and include a sensitivity warning if payloads are present.

## 9.3 Reliability requirements

- A worker restart must not lose committed run state.
- Runs must be resumable/recoverable after worker termination.
- Duplicate SDK observation events must be idempotently detected by event ID.
- API readiness must check PostgreSQL and Redis connectivity; liveness must not depend on them.
- Database schema changes use Alembic migrations; destructive migrations require a documented backup/rollback plan.
- Backups include PostgreSQL and artifact storage; document restore procedure.

## 9.4 Performance and capacity targets for v0.1

These are initial product targets to validate in load tests, not guarantees for arbitrary hardware or model sizes.

| Area | Target |
|---|---|
| CRUD API | p95 under 300 ms excluding external/model work at 20 concurrent clients on reference dev host |
| Run creation | p95 under 500 ms to durable `202 Accepted` under normal load |
| Observation batch | Up to 500 small metadata events per request; larger payloads use artifact uploads |
| Dashboard | Initial project/run views interactive within 2 seconds on a local reference deployment for a small project |
| Result persistence | Stream/batch writes; do not buffer full dataset outputs in worker memory |
| Recovery | Stale worker lease detected and eligible items re-queued within configurable recovery interval |
| CI gate | Return a machine-readable decision and stable exit code |

## 9.5 Compatibility matrix

First-release CI should test:

- Linux x86_64: CPU inference adapter tests and full Docker Compose integration.
- macOS Apple Silicon: API/SDK and at least one local inference path; PyTorch MPS support tested separately.
- Linux NVIDIA/CUDA: optional integration tests when CI hardware is available.
- Python 3.11 and 3.12 for core/API/SDK.
- PostgreSQL supported release range declared in deployment docs.

Do not state that all model architectures or quantization backends are supported merely because a generic Transformers adapter exists.

---

# 10. Deployment and repository layout

## 10.1 Docker Compose services

| Service | Container role | Persistent data |
|---|---|---|
| `web` | React dashboard | Static assets/build only |
| `api` | FastAPI service | None; state in DB/object store |
| `worker` | Celery worker(s) | Temporary model cache; optional cache volume |
| `scheduler` | Outbox recovery/periodic housekeeping | None |
| `postgres` | Relational database | PostgreSQL volume |
| `redis` | Job broker/cache | Optional persistence; not source of truth |
| `minio` (optional) | S3-compatible local artifact store | Artifact volume |

Default installation may use local filesystem storage to reduce setup. MinIO is an optional Compose profile. External S3-compatible storage can be configured for larger deployments.

## 10.2 Environment variables

```dotenv
OPENTRACE_ENV=production
OPENTRACE_BASE_URL=http://localhost:8000
OPENTRACE_SECRET_KEY=replace-with-generated-secret
DATABASE_URL=postgresql+psycopg://opentrace:change-me@postgres:5432/opentrace
REDIS_URL=redis://redis:6379/0
OPENTRACE_STORAGE_BACKEND=local
OPENTRACE_STORAGE_PATH=/var/lib/opentrace/artifacts
OPENTRACE_CONTENT_CAPTURE_DEFAULT=false
OPENTRACE_MAX_UPLOAD_BYTES=104857600
OPENTRACE_WORKER_CONCURRENCY=1
OPENTRACE_LOG_LEVEL=INFO
```

Secrets must be generated at install time and stored outside source control. Example values must be clearly marked non-production. Do not expose secrets in `/health` endpoints.

## 10.3 Proposed monorepo structure

```text
opentrace-llm/
├── apps/
│   ├── api/                    # FastAPI application and route composition
│   ├── worker/                 # Celery tasks and worker runtime
│   └── web/                    # React + TypeScript dashboard
├── packages/
│   ├── core/                   # Domain models, interfaces, evaluation engine
│   ├── adapters/               # transformers, ollama, openai-compatible
│   ├── evaluators/             # built-in evaluator implementations
│   ├── sdk/                    # Python SDK
│   └── cli/                    # CLI commands
├── migrations/                 # Alembic migrations
├── datasets/
│   └── examples/               # Original, license-cleared demo data only
├── examples/
│   ├── experiment.yaml
│   └── sdk-observations.py
├── deploy/
│   ├── docker-compose.yml
│   └── containers/
├── docs/
│   ├── architecture.md
│   ├── evaluator-authoring.md
│   ├── model-adapters.md
│   ├── privacy.md
│   └── self-hosting.md
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── contract/
│   └── fixtures/
├── pyproject.toml
├── pnpm-workspace.yaml
├── Makefile
├── .env.example
├── CONTRIBUTING.md
├── SECURITY.md
├── CODE_OF_CONDUCT.md
├── LICENSE
└── README.md
```

## 10.4 CI checks

- Python formatting/lint/type checks (Ruff + mypy or Pyright, selected consistently).
- Unit tests for core metrics, evaluators, adapter contracts, and regression decisions.
- PostgreSQL integration tests for migrations, tenant scoping, transaction handling, and pagination.
- API contract tests against generated OpenAPI.
- SDK tests with mocked transport, retries, idempotency, and content-capture defaults.
- Frontend typecheck, lint, and production build.
- Docker Compose smoke test: bootstrap, create project, import demo dataset, run fake adapter, compare, gate, ingest observation.
- Dependency/license and secret scanning.
- Build and publish containers/packages only from tagged releases after all gates pass.

---

# 11. UX and reporting requirements

## 11.1 Project home

Show recent experiments and observations, active runs, latest regression decision, registered model count, dataset count, and system health. Avoid vanity aggregate “model quality” scores.

## 11.2 Run detail

- State/progress by target and item count.
- Success, failed, skipped, and pending counts.
- Aggregate metrics with sample counts and evaluator provenance.
- Slice breakdown by language/category/tag.
- Latency distribution and token usage.
- Item-level search/filter and side-by-side input/output/score explanation.
- Failure details with redacted provider errors.
- Run manifest and artifact downloads.

## 11.3 Comparison view

- Baseline and candidate identities visible at the top.
- Compatibility status: comparable / partially comparable / not comparable.
- Metric delta with direction and threshold interpretation.
- Slice deltas and minimum-sample warnings.
- Newly failed, newly passed, changed, and unchanged cases.
- Output diffs with optional text diff, not only similarity score.
- Separate quality, safety/refusal, latency, and cost/usage panels.

## 11.4 Export format

Export should contain:

- `manifest.json` - immutable run metadata and hashes.
- `metrics.json` - aggregate and slice metrics.
- `items.jsonl` - case-level inference and evaluator results, with content according to project policy.
- `artifacts/` - optional raw outputs, logs, and judge evidence.
- `README.txt` - schema version, generation time, privacy/content flags.

---

# 12. Milestones and delivery plan

An eight-week baseline assumes one primary engineer with focused work and occasional review. This is a planning estimate, not a delivery guarantee. Keep all milestones vertically integrated and demonstrable.

| Milestone | Weeks | Deliverables | Exit criteria |
|---|---:|---|---|
| M0 - Foundation | 1 | Monorepo, CI, config, DB migrations, Compose skeleton, core domain types | Clean install boots API, DB, Redis, web shell; CI green |
| M1 - Projects and registry | 2 | Workspace bootstrap, projects, API keys, model registry, three adapter skeletons | Create project/model via API and validate adapter capabilities |
| M2 - Evaluation core | 3 | Dataset import/versioning, evaluator registry, core metrics, case schema | Run deterministic unit fixtures and inspect per-case scores |
| M3 - Async execution | 4 | Experiment/run APIs, Celery workers, inference adapter integration, progress/retry/cancel | Run sample dataset against two targets; survive worker restart |
| M4 - Comparison and gates | 5 | Baseline/candidate comparison, slices, regression policies, CLI exit codes | Demonstrate a failing and passing release gate in CI |
| M5 - Observability SDK | 6 | Python SDK, batch ingestion, filters, content-capture controls | Record test observations; confirm no content stored by default |
| M6 - Dashboard completeness | 7 | Project, models, datasets, run detail, comparison, observations, settings screens | Full workflow usable without direct DB access |
| M7 - Release hardening | 8 | Security/privacy review, docs, backup/restore, performance tests, sample data, release artifacts | Fresh-machine install, end-to-end smoke, docs, signed/tagged release |

## 12.1 Release acceptance criteria

v0.1 is ready only when all conditions are met:

1. A developer can install OpenTrace using documented Docker Compose steps on a supported host.
2. First-run setup creates a secure owner account or explicitly configured local mode; no default password exists.
3. A project can register at least one local model and one OpenAI-compatible endpoint.
4. A dataset can be imported, validated, versioned, previewed, and exported.
5. A user can run at least two model targets against the same dataset and evaluators.
6. Run outputs, evaluator results, and manifest are persisted and inspectable.
7. The dashboard shows metrics, slices, failures, and output diffs.
8. A regression policy can pass and fail with stable CLI exit codes.
9. The Python SDK can ingest observations with content capture disabled by default.
10. Worker restart/cancellation/retry tests pass without duplicate item results.
11. Project/workspace authorization tests pass.
12. A clean install, restore guide, privacy guide, and contributor guide are published.

---

# 13. Initial GitHub issue backlog

Use these as the first repository issues. Labels are suggestions; issue numbers should be assigned by GitHub. The order below reflects dependencies, not a promise that all work is parallelizable.

## Epic A - Foundation and project model

### Issue A1 - Bootstrap monorepo, development tooling, and CI
**Labels:** `epic:foundation`, `priority:P0`, `type:infra`  
**Description:** Establish the Python workspace, frontend workspace, formatting/lint/type checks, test runners, pre-commit hooks, GitHub Actions, and contribution guide.  
**Acceptance criteria:**

- One documented command installs development dependencies.
- CI runs Python lint/type/tests and frontend lint/type/build.
- A release build can produce versioned Python package and container image artifacts.
- No secrets or developer-specific files are committed.

### Issue A2 - Define core domain types and public schemas
**Labels:** `priority:P0`, `type:architecture`  
**Description:** Implement typed domain objects for model targets, dataset versions/cases, evaluator versions, experiments, runs, inference results, evaluation results, metrics, and regression decisions.  
**Acceptance criteria:**

- Domain package has no FastAPI, SQLAlchemy, or Celery dependency.
- Types validate required fields and serialize to stable JSON.
- Schema version is explicit for persisted/exported objects.

### Issue A3 - PostgreSQL schema and Alembic migrations
**Labels:** `priority:P0`, `type:backend`, `area:database`  
**Description:** Implement the initial relational schema in Section 7, indexes, foreign keys, and migration test fixture.  
**Acceptance criteria:**

- Fresh database can migrate from empty to head.
- Upgrade/downgrade behaviour is tested where safe.
- Historical model/dataset/evaluator versions cannot be deleted while referenced by runs.

### Issue A4 - Workspace bootstrap, authentication, and API keys
**Labels:** `priority:P0`, `type:security`, `area:backend`  
**Description:** Implement first-run owner bootstrap, workspace membership, API key issue/revoke/rotate, and project-scoped authorization.  
**Acceptance criteria:**

- No default credentials.
- Raw API key is displayed once; only its hash is persisted.
- Unauthenticated and cross-workspace requests are rejected.
- Revoked/expired keys stop working immediately.

## Epic B - Model and dataset infrastructure

### Issue B1 - Model registry and immutable model versions
**Labels:** `priority:P0`, `type:backend`, `area:models`  
**Description:** Add model registration, version snapshots, config hashing, capability metadata, and connection test.  
**Acceptance criteria:**

- Mutable model registration is separate from immutable model versions.
- Secret references are never returned in API responses.
- Model version contains a reproducible inference configuration snapshot.

### Issue B2 - Model adapter protocol and fake adapter
**Labels:** `priority:P0`, `type:core`, `area:models`  
**Description:** Define the model adapter interface and implement a deterministic fake adapter for end-to-end tests.  
**Acceptance criteria:**

- Capability discovery and config validation are part of the protocol.
- Errors and usage are normalized.
- Fake adapter enables tests without network or GPU.

### Issue B3 - Transformers adapter
**Labels:** `priority:P0`, `type:integration`, `area:models`  
**Description:** Implement local Hugging Face Transformers inference for supported text/chat models.  
**Acceptance criteria:**

- Supports local path or Hub model ID and pinned revision where available.
- Reports load/inference errors explicitly.
- Optional runtime dependencies are installed through a documented extra.
- CPU path is tested; CUDA/MPS support is capability-tested, not assumed.

### Issue B4 - Ollama adapter
**Labels:** `priority:P0`, `type:integration`, `area:models`  
**Description:** Implement Ollama chat generation and metadata normalization.  
**Acceptance criteria:**

- Connection test, model identifier, timeout, generation settings, usage (when reported), and errors are handled.
- Adapter tests use a mocked local HTTP server.

### Issue B5 - OpenAI-compatible adapter
**Labels:** `priority:P0`, `type:integration`, `area:models`  
**Description:** Implement generic OpenAI-compatible chat completions adapter with configurable base URL and secret reference.  
**Acceptance criteria:**

- Does not hardcode a single vendor.
- Handles timeout, rate-limit, non-2xx, usage, finish reason, and JSON mode capability when supported.
- Secrets are redacted from logs/errors.

### Issue B6 - Dataset import, validation, and immutable versioning
**Labels:** `priority:P0`, `type:backend`, `area:datasets`  
**Description:** Add JSONL/CSV import, case validation, dataset version creation, content hash, preview, and JSONL export.  
**Acceptance criteria:**

- Invalid rows return line/column-level errors without silently dropping them.
- Versions are immutable.
- Dataset version stores case count and SHA-256 hash.

## Epic C - Evaluation and asynchronous execution

### Issue C1 - Evaluator registry and versioning
**Labels:** `priority:P0`, `type:core`, `area:evaluation`  
**Description:** Create evaluator protocol, metric definitions, evaluator config schema, versioning, and plugin discovery.  
**Acceptance criteria:**

- Evaluator version/config hash is saved in the run manifest.
- Evaluator errors are distinct from model inference errors.
- Plugin API has authoring documentation and contract tests.

### Issue C2 - Built-in refusal and false-refusal evaluator
**Labels:** `priority:P0`, `type:feature`, `area:evaluation`  
**Description:** Implement refusal/answer/uncertain classification and aggregate appropriate-refusal and false-refusal metrics from labelled cases.  
**Acceptance criteria:**

- `should_answer` and `should_refuse` are computed separately.
- `uncertain` cases are counted/reported separately.
- Metric direction, denominator, sample count, and evaluator version are visible.

### Issue C3 - Deterministic instruction and structured-output evaluators
**Labels:** `priority:P0`, `type:feature`, `area:evaluation`  
**Description:** Add exact/contains/regex/JSON-schema validation and task-specific deterministic checks.  
**Acceptance criteria:**

- Results normalized to documented score/label shapes.
- Invalid evaluator config fails validation before a run is queued.
- Unit tests include positive, negative, and malformed outputs.

### Issue C4 - Optional LLM-as-judge evaluator
**Labels:** `priority:P0`, `type:feature`, `area:evaluation`, `privacy`  
**Description:** Implement a judge evaluator with explicit judge model version, prompt version, provider, decoding settings, and evidence.  
**Acceptance criteria:**

- Judge is separately registered and identifiable.
- Remote judge use requires explicit configuration.
- Judge failures do not mark model inference as failed.
- UI and export label judge-derived metrics distinctly.

### Issue C5 - Celery/Redis worker execution and run state machine
**Labels:** `priority:P0`, `type:backend`, `area:workers`  
**Description:** Implement asynchronous run dispatch, worker claiming, progress updates, cancellation, retries, and stale-worker recovery.  
**Acceptance criteria:**

- API returns `202` after run manifest is durably stored.
- Worker restart does not lose completed items.
- Retries do not duplicate item or score records.
- DB remains source of truth if Redis is unavailable temporarily.

### Issue C6 - Run metrics aggregation and slice breakdowns
**Labels:** `priority:P0`, `type:feature`, `area:evaluation`  
**Description:** Aggregate evaluator results by target, metric, language, category, and tags.  
**Acceptance criteria:**

- Sample and error counts accompany every aggregate.
- Micro/macro aggregation semantics are documented.
- Insufficient sample slices are labelled.

## Epic D - Comparison, observability, and gates

### Issue D1 - Baseline/candidate comparison engine
**Labels:** `priority:P0`, `type:feature`, `area:comparison`  
**Description:** Compare compatible runs, detect coverage differences, calculate metric deltas, and expose case-level changes.  
**Acceptance criteria:**

- Dataset/evaluator incompatibility is reported, not hidden.
- Unmatched cases are excluded from paired calculations by default.
- Output diff references link to exact run items.

### Issue D2 - Regression policy engine and CLI gate
**Labels:** `priority:P0`, `type:feature`, `area:regression`  
**Description:** Implement metric threshold rules, sample minimums, pass/warn/fail/inconclusive results, and CLI exit codes.  
**Acceptance criteria:**

- Policy defines metric, slice, direction, threshold, and minimum sample count.
- CLI returns stable exit codes and `--json` output.
- Tests cover pass, fail, warning, and inconclusive conditions.

### Issue D3 - Python SDK and observation ingestion API
**Labels:** `priority:P0`, `type:feature`, `area:sdk`  
**Description:** Implement typed SDK client, batch observations, event IDs, retries, and API endpoint.  
**Acceptance criteria:**

- SDK supports context manager/client lifecycle and bounded retry.
- Duplicate event IDs are idempotent.
- Content capture is disabled by default and explicit opt-in is covered by tests.

### Issue D4 - Observation query and model/prompt version filters
**Labels:** `priority:P0`, `type:feature`, `area:observability`  
**Description:** Persist and query observation metadata, latency, token usage, status, tags, trace IDs, model and prompt versions.  
**Acceptance criteria:**

- Time-range pagination and filters work at expected index scale.
- Content-free observations are useful and fully queryable.
- Raw content display requires project capture setting and appropriate permission.

### Issue D5 - OpenTelemetry GenAI field mapping
**Labels:** `priority:P1`, `type:integration`, `area:observability`  
**Description:** Map common GenAI span attributes to OpenTrace observation fields and document unsupported fields.  
**Acceptance criteria:**

- Mapping documented against a pinned semantic convention revision.
- Sensitive content remains opt-in.
- Unknown attributes are retained safely as metadata where size limits allow.

## Epic E - Dashboard, deployment, and release

### Issue E1 - Dashboard project and model registry screens
**Labels:** `priority:P0`, `type:frontend`, `area:dashboard`  
**Description:** Build workspace/project navigation, project overview, model list, registration, and connection test UI.  
**Acceptance criteria:**

- All states include loading/empty/error handling.
- API errors are presented without leaking secrets.
- Model version identity is visible.

### Issue E2 - Dataset and evaluator management screens
**Labels:** `priority:P0`, `type:frontend`, `area:dashboard`  
**Description:** Build dataset import/preview/version UI and evaluator configuration/version UI.  
**Acceptance criteria:**

- Import validation errors are actionable.
- Version immutability is communicated.
- Judge configuration shows provider/data-flow warnings.

### Issue E3 - Experiment, run-detail, and comparison screens
**Labels:** `priority:P0`, `type:frontend`, `area:dashboard`  
**Description:** Build experiment setup, run progress, result tables, metric charts, slice views, case diffs, and comparison UI.  
**Acceptance criteria:**

- Full end-to-end flow works without direct API/DB use.
- Partial/error states are visible.
- Metric sample counts and provenance are displayed.

### Issue E4 - Observability and regression policy screens
**Labels:** `priority:P0`, `type:frontend`, `area:dashboard`  
**Description:** Build observation filtering/timeline and regression policy configuration/results views.  
**Acceptance criteria:**

- Content-capture state is visible.
- Regression decision reasons and affected slices are visible.
- Users can inspect a failing gate's case-level evidence.

### Issue E5 - Docker Compose deployment, bootstrap, and backup/restore
**Labels:** `priority:P0`, `type:infra`, `area:deployment`  
**Description:** Package web, API, worker, PostgreSQL, Redis, and local/S3-compatible artifact storage for self-hosted use.  
**Acceptance criteria:**

- Fresh host can install from documented steps.
- Setup generates secure secrets and runs migrations.
- Backup/restore covers database and artifact storage.
- Containers run with least practical privileges.

### Issue E6 - Security/privacy hardening and v0.1 release checklist
**Labels:** `priority:P0`, `type:security`, `type:release`  
**Description:** Complete authorization, content-capture, artifact-access, upload, secret-redaction, dependency/license, and release verification.  
**Acceptance criteria:**

- Security tests cover cross-project access and revoked keys.
- Privacy test proves content capture is off by default.
- Clean-install end-to-end test passes.
- README, SECURITY.md, CONTRIBUTING.md, and release notes are complete.

---

# 14. Risks, mitigations, and open decisions

| Risk | Why it matters | Mitigation |
|---|---|---|
| Benchmark validity | A benchmark can measure its own dataset quirks rather than general behaviour | Show provenance, sample counts, slices, limitations, and human-review support |
| Judge bias/variance | LLM judges can be inconsistent or favour certain styles/models | Version judges/prompts, calibrate, separate judge metrics, show evidence and uncertainty |
| Refusal label ambiguity | Refusal is not a binary universal property | Use expected-behaviour labels, `uncertain`, documented policy categories, and review path |
| Name collision | OpenTrace is already used in adjacent developer tooling and as a PyPI distribution | Use `opentrace-llm` identifiers; clear brand/domain/trademark before broad marketing |
| Worker/model memory pressure | Local models can exhaust RAM/VRAM, particularly on consumer hardware | Conservative per-device concurrency, adapter resource hints, explicit OOM classification |
| Data leakage | Prompts/outputs may contain PII, secrets, or proprietary data | No content capture by default, explicit opt-in, redaction, retention, self-hosting |
| Dataset licensing | Public benchmark datasets may restrict redistribution | Ship only original/license-cleared demo cases; provide import tooling |
| Provider variability | Remote model aliases and API behaviour can change | Snapshot IDs/config, mark unknown revisions, retain request metadata, avoid false reproducibility claims |
| Feature breadth | Complete platform scope can delay a usable release | Keep all surfaces narrow; fake adapter and vertical slices; enforce release acceptance criteria |
| Storage growth | Outputs and observation events can become large | Separate artifacts, quotas/retention, batch ingestion, configurable content storage |

## 14.1 Decisions to freeze before implementation

1. **Brand:** OpenTrace remains a working title; public package/repo identifiers use `opentrace-llm` until naming review is complete.
2. **License:** Apache-2.0 is the recommended default for broad developer adoption, pending dependency and dataset review.
3. **Queue:** Celery + Redis for v0.1; PostgreSQL is the source of truth.
4. **Artifacts:** local filesystem default, optional S3-compatible backend.
5. **Inference adapters:** Transformers, Ollama, OpenAI-compatible.
6. **Content capture:** disabled by default.
7. **Primary comparison:** immutable run manifests and case-level evidence; no comparisons that silently mix evaluator/dataset versions.
8. **Release shape:** one self-hosted Compose installation with a full UI/SDK/CLI workflow.

## 14.2 Questions intentionally deferred

- Should a later release support multi-workspace hosted SaaS or remain strictly self-hosted?
- Should OpenTrace accept OTLP directly or only provide OpenTelemetry mapping/SDK integration initially?
- Which statistical comparison methods should become first-class beyond paired bootstrap?
- Should dataset/evaluator packs be distributable through a registry?
- Which model providers beyond the initial three have enough community demand to justify first-party adapters?
- Should the runtime observation store support traces/spans as a full graph, or remain a GenAI-call-centric event model?

---

# 15. Definition of done

A feature is done when:

- Its API and domain contract are documented and versioned.
- Unit and integration tests cover success, invalid input, authorization, and failure paths.
- It has observability and safe error handling without leaking credentials or content.
- It appears in the dashboard/CLI/SDK when applicable.
- Its data model is migrated and has indexes/constraints appropriate to expected queries.
- Its privacy and retention implications are documented.
- A contributor can run it locally through the documented development environment.

A release is done only when the release acceptance criteria in Section 12.1 pass.

---

# 16. References and ecosystem conventions

These references informed architectural choices and naming checks. They are not endorsements or guarantees of compatibility.

1. OpenTelemetry GenAI semantic conventions, including inference spans and opt-in content capture guidance: https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/gen-ai/gen-ai-spans.md
2. FastAPI background task guidance and recommendation to use a larger queue system for heavy work across processes/servers: https://fastapi.tiangolo.com/tutorial/background-tasks/
3. Existing project using the OpenTrace name and `opentrace/opentrace` GitHub namespace: https://github.com/opentrace/opentrace
4. Existing unrelated `opentrace` Python distribution on PyPI: https://pypi.org/project/opentrace/
5. OpenTelemetry semantic convention guidance: https://opentelemetry.io/docs/specs/semconv/

---

**End of specification.**
