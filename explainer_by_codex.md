# Agentic Audiences Repository Explainer

This document is a deep technical walkthrough of the repository as it exists today. It is intended to be strong source material for NotebookLM-driven explainers, presentations, and onboarding. The emphasis here is not just on the stated vision, but on what is actually implemented in code, what is still draft specification work, and where the seams between those layers currently are.

## 1. Executive summary

Agentic Audiences is an open protocol and reference implementation set for exchanging learned advertising signals as embeddings rather than as sparse segment IDs or large prompt payloads. The repository combines:

- specification work for a vendor-neutral embedding exchange format,
- taxonomy and architecture documents explaining how embeddings fit into the advertising stack,
- a reference FastAPI scoring service that scores user embeddings against campaign head vectors,
- an example Prebid RTD submodule that injects stored embeddings into OpenRTB `user.data`,
- supporting examples, tests, and packaging artifacts.

The core architectural thesis is:

1. audience-side systems generate embeddings asynchronously from identity, context, behavior, and reinforcement signals,
2. those embeddings ride along standard ad-tech transport surfaces such as OpenRTB,
3. demand-side systems score them against campaign-specific vectors in real time,
4. feedback signals can be aggregated back into model training loops.

This repo is therefore both a standards proposal and a concrete prototype.

## 2. Repository layout

The repository has five meaningful areas:

### 2.1 Top-level docs and metadata

- `README.md`: primary project framing, motivation, principles, roadmap, and relationship to buyer/seller agent ecosystems.
- `explainer.md`: an older repository explainer already present in-tree.
- `catalog-info.yaml`: Backstage-style metadata. It still references the legacy project slug `LiveRamp/user-context-protocol`, which reflects the former name.
- `CONTRIBUTING.md`: minimal contribution workflow.
- `LICENSE`, `LICENSE-APACHE`: dual licensing split between documentation/specs and reference code.
- `CODE_OF_CONDUCT.md`: present but empty.

### 2.2 Architecture and whitepaper docs

- `docs/systems-and-models.md`: the most important architecture document in the repo. It explains the end-to-end system: audience model, browser/edge transport, campaign scoring, training loop, and component ownership.
- `docs/AI_ML Models in Agentic Digital Advertising Era.pdf`: referenced throughout the repo as the broader ML whitepaper for the model ecosystem. The repo treats this PDF as supporting theory rather than executable spec.

### 2.3 Specifications

- `specs/v1.0/embedding-exchange.md`: draft wire-format spec for embedding exchange.
- `specs/v1.0/embedding-taxonomy.md`: taxonomy of embedding classes and usage semantics.
- `specs/v1.0/schema/embedding_format.schema.json`: JSON Schema for the embedding envelope.
- `specs/v1.0/schema/agent_interface.schema.json`: exists but is currently empty.
- `specs/v1.0/examples/embedding_update.json`: concrete example of a spec envelope.
- `specs/v1.0/examples/buyer_agent_request.json`: currently empty.
- `specs/v1.0/examples/seller_agent_response.json`: currently empty.
- `specs/roadmap.md`: present but currently empty.

### 2.4 Reference scoring service

- `src/user-embedding-to-campaign-scoring/`: Python FastAPI service for campaign head registration, scoring, analytics, tests, examples, and Docker packaging.

### 2.5 Prebid integration example

- `prebid-module/`: example Prebid.js Real-Time Data submodule showing how browser-stored embeddings can be injected into OpenRTB structures.

There is also `community/README.md`, but it is currently empty and acts as a placeholder for future working-group material.

## 3. Project thesis and problem framing

The repo’s framing is consistent across `README.md` and `docs/systems-and-models.md`:

- traditional ad-tech transports tend to move sparse IDs, boolean segments, key-value pairs, or prompt-style text,
- modern AI/ML systems prefer dense vector representations,
- embeddings are the proposed interoperability layer because they compress high-dimensional user, contextual, and reinforcement information into a form that supports fast similarity math,
- the system wants to preserve sub-100ms RTB constraints while making targeting more semantic and less rules-based.

The stated transition model is three-phase:

1. structured agent-to-agent interoperability,
2. learned context modeling,
3. full embedding-based intelligence exchange.

The repo positions Agentic Audiences as the data plane that complements control-plane agent ecosystems such as buyer agents and seller agents.

## 4. Systems architecture as described by the docs

`docs/systems-and-models.md` is the clearest expression of the intended production architecture.

### 4.1 Audience model

The audience model generates embeddings from contextual, identity, and behavioral inputs. The document emphasizes that generation should be asynchronous and outside the critical RTB path. The repo explicitly does not assume a single model vendor; instead it assumes a marketplace of model builders whose outputs are interoperable only when metadata and taxonomy indicate compatibility.

This is important because the repo is not saying “all 384-dimensional vectors are interchangeable.” It says the opposite: vector space identity is a first-class concern.

### 4.2 Browser and edge layer

The browser-side story is:

- a tag such as ATS.js manages first-party storage,
- publishers provide clean page signals rather than allowing aggressive crawling,
- embeddings are stored client-side or server-side,
- Prebid.js picks them up and injects them into OpenRTB `user.data[].segment[].ext`.

The repository treats first-party storage as both a privacy and latency optimization.

### 4.3 Campaign scoring service

The scoring service is intentionally narrow. It does not bid. It does not do pacing. It does not do budget management. It answers only:

“Given this user/context embedding, how strongly does it match these campaign vectors?”

That narrowness matters operationally, because it makes the service plausible as a DSP sidecar rather than a full bidding engine replacement.

### 4.4 Training loop

The training loop is described conceptually, not implemented in this repo. Advertisers or clean rooms generate campaign head vectors from CRM, CAPI, marketplace, and reinforcement data. Those heads are then registered with the scoring service. Feedback signals and reduced embedding representations can be looped back into retraining.

### 4.5 Model interoperability

The docs are clear that embedding compatibility is conditional. Compatibility can come from:

- exact shared embedding spaces,
- explicit metadata and compatibility declarations,
- alignment or mapping techniques when models are trained on comparable data regimes.

This design idea is partially implemented in the scoring service through `model`, `type`, and `compatible_with` handling.

### 4.6 Component ownership model

One useful detail from `docs/systems-and-models.md` is that the architecture is explicit about who is expected to operate each part:

- publisher or publisher-side provider operates the tag and browser storage layer,
- LiveRamp or another provider may operate the inference server,
- the DSP operates the campaign scoring sidecar,
- advertiser or clean-room environments operate campaign training,
- a signal aggregator closes the loop for downstream learning.

This matters because the repo is not describing a monolithic product. It is describing a federated ecosystem where the protocol is the interoperability boundary between independently operated systems.

## 5. The specification layer

The specification work is draft-quality but already opinionated.

### 5.1 `embedding-exchange.md`

This document defines a vendor-neutral envelope for exchanging embeddings over HTTPS JSON, with NDJSON and CBOR as optional variants. Important design ideas:

- every message has a `spec_version`,
- embeddings carry enough metadata to judge comparability,
- consent and TTL enforcement are first-class,
- security and optional attestation are part of the story,
- errors distinguish schema invalidity from embedding-space incompatibility.

The most important conceptual fields are:

- embedding object: `id`, `type`, `vector` or `quantized_b64`, `dimension`, `dtype`,
- model descriptor: `id`, `version`, `dimension`, `metric`, `type`, `embedding_space_id`,
- context descriptor: URL, title, keywords, language, placement, device, geography,
- consent object: purposes, permissible uses, and `ttl_seconds`.

### 5.2 `embedding-taxonomy.md`

This document is not just nomenclature; it defines when embeddings should be considered semantically comparable. It classifies embeddings across three axes:

- signal type,
- temporal scope,
- composition.

Signal-type classes are extensive:

- identity embeddings,
- contextual embeddings,
- reinforcement embeddings,
- creative embeddings,
- inventory embeddings,
- query/intent embeddings.

Each category has subtypes and example model families. For example:

- graph neural networks for graph-based identity,
- sentence transformers or multimodal encoders for contextual understanding,
- event-sequence models for reinforcement,
- CLIP-style joint encoders for multimodal creative.

This taxonomy is strategically important because the architecture uses it as the conceptual basis for deciding whether alignment between models is meaningful.

### 5.3 `embedding_format.schema.json`

The JSON Schema is the most concrete spec artifact. It is JSON Schema draft 2020-12 and defines a strict envelope with many `additionalProperties: false` constraints.

Required top-level fields:

- `spec_version`
- `message_id`
- `timestamp`
- `model`
- `context`
- `embeddings`

The `model` block requires:

- `id`
- `version`
- `type`
- `architecture`
- `dimension`
- `metric`
- `embedding_space_id`

The `context` block only requires `language`, but supports URL, page title, keywords, placement, device, and geography. `consent` requires `purposes` and `ttl_seconds` when present.

The schema also includes optional top-level blocks for:

- `producer`
- `identity`
- `consent`
- `security`
- `extensions`

So the schema is trying to be a full exchange envelope, not merely a bare vector payload.

One important detail: the schema’s embedding `type` enum is:

- `context`
- `creative`
- `user_intent`
- `inventory`
- `query`

That enum is narrower and differently named than the canonical types used by the scoring service, which uses internal names like `contextual`, `intent`, and `capi`.

### 5.4 Example and placeholder status

`specs/v1.0/examples/embedding_update.json` is the only non-empty example in the spec examples directory. It shows:

- a model with `embedding_space_id`,
- context keywords,
- consent with TTL,
- one embedding vector.

Several spec-adjacent files are placeholders today:

- `specs/v1.0/schema/agent_interface.schema.json` is empty,
- `specs/v1.0/examples/buyer_agent_request.json` is empty,
- `specs/v1.0/examples/seller_agent_response.json` is empty,
- `specs/roadmap.md` is empty.

That tells you this repo is farther along on embedding transport and scoring than on fully specified multi-agent request/response protocols.

## 6. Reference scoring service: what is implemented

The most complete software artifact in the repo is `src/user-embedding-to-campaign-scoring/`.

### 6.0 End-to-end request lifecycle

From an operational perspective, a single scoring transaction looks like this:

1. an external system registers one or more campaign heads with model name, type, dimension, and weights,
2. the service stores those heads partitioned by canonicalized `model:type`,
3. a bid-like request arrives at `/score` with one or more embedding-carrying user segments,
4. each segment is extracted independently and becomes one scoring job,
5. the store returns all heads whose canonical type matches and whose model is compatible,
6. the scorer computes top-k similarity,
7. analytics records the scored embedding once for each returned campaign head,
8. the API returns one response object per input embedding.

The “one response object per embedding” part is easy to miss and matters for consumers: `/score` is not a single global top-k across the whole request body; it is a per-embedding scoring surface.

### 6.1 Stack and packaging

The service uses:

- Python 3.12,
- FastAPI,
- Uvicorn,
- Pydantic v2,
- NumPy,
- scikit-learn for PCA analytics,
- optional PyTorch for accelerated scoring.

`requirements.txt` includes both runtime and test dependencies. `requirements-gpu.txt` layers PyTorch on top via the CUDA 12.4 wheel index.

Packaging:

- `Dockerfile` uses `python:3.12-slim`, installs requirements, copies `app/`, and serves on port `8080`.
- `Dockerfile.gpu` uses `nvidia/cuda:12.6.3-runtime-ubuntu24.04`, installs Python tooling, installs the GPU requirements, copies `app/`, `tests/`, and `config.yaml`, and runs Uvicorn via `python3 -m`.

The CPU image does not copy `config.yaml`, which is fine because the app falls back to defaults if the config file is absent.

### 6.2 App lifecycle and startup

`app/main.py` wires the service together using a FastAPI lifespan hook:

- load config,
- instantiate `CampaignHeadStore`,
- instantiate `AnalyticsTracker`,
- attach both to `app.state`,
- log whether scoring is running via NumPy, CPU torch, or CUDA.

Routes exposed:

- `POST /score`
- `POST /campaigns/heads`
- `PUT /campaigns/heads`
- `DELETE /campaigns/heads/{campaign_head_id}`
- `GET /analytics`
- `GET /health`

`GET /health` returns `{"status":"ok","device":...}` where `device` becomes:

- `"numpy"` if torch is not installed,
- `"cpu"` if torch is installed without CUDA,
- `"cuda:0"` if CUDA is available.

### 6.3 Configuration model

`app/config.py` defines:

- `AnalyticsConfig`
- `AppConfig`

Config sources:

- explicit path argument,
- `CONFIG_PATH` environment variable,
- default `config.yaml` beside the service package root.

The live config file contains:

- canonical embedding types,
- wire-to-canonical type aliases,
- analytics parameters.

Canonical types:

- `identity`
- `contextual`
- `reinforcement`
- `capi`
- `intent`

Alias mapping:

- `context -> contextual`
- `creative -> capi`
- `user_intent -> intent`
- `query -> intent`
- `inventory -> contextual`

This is how the implementation reconciles spec-style wire names with internal scoring partitions.

### 6.4 Campaign head store

`app/engine/store.py` is the core stateful component.

#### 6.4.1 Data model

It stores:

- `StoredHead`: campaign ID, head ID, float16 weight vector,
- `ModelConfig`: dimension, metric, `apply_l2_norm`, compatibility list,
- partitions keyed by `"{model}:{canonical_embedding_type}"`,
- a cache of stacked weight matrices for fast repeated scoring,
- a map from `campaign_head_id` to partition key,
- per-model head counts for cleanup.

#### 6.4.2 Why float16

Registered campaign weights are converted to `np.float16`. That cuts memory roughly in half while keeping enough precision for high-throughput similarity calculations. During NumPy scoring, weights are upcast to `float32` before arithmetic.

#### 6.4.3 Registration semantics

`register()` is upsert-based.

Rules:

- `len(weights)` must equal declared `dimension`,
- the first registered head for a given `model` establishes that model’s dimension and scoring config,
- later heads for the same model must match the established dimension,
- re-registering an existing `campaign_head_id` removes the old instance first and inserts the replacement.

The store does not currently validate that later heads for the same model use the same metric or `apply_l2_norm` as the first head. It simply stores the first head’s model config and reuses it for scoring that model. That means the documented idea “model configuration travels with each head” is only partly true in implementation; operationally it is really “the first head for a model establishes scoring behavior for that model.”

The same caveat applies to `compatible_with`: only the first registration for a model populates the stored `ModelConfig`. Later heads for that model may include a different compatibility list in the payload, but the store does not merge or refresh the model-level compatibility graph unless the model config is removed and recreated.

Another subtle point: `embedding_space_id` is accepted by the API model, but the store does not persist or consult it when partitioning or scoring heads. In practice, runtime compatibility is determined only by:

- `model`
- resolved embedding `type`
- `compatible_with`

#### 6.4.4 Compatibility model

Compatibility is model-name based:

- identical model names are always compatible,
- otherwise compatibility is true if `model_a` lists `model_b` in `compatible_with`,
- or `model_b` lists `model_a`.

This is symmetric at lookup time, even if only one side declared the relationship.

#### 6.4.5 Caching

`get_heads()` caches the tuple `(head_ids, weight_matrix)` by request-side partition key. When torch is available, the cached matrix is a `torch.float16` tensor on the selected device. Cache invalidation happens on register, update, and delete.

Empty results are intentionally not cached.

### 6.5 Scoring engine

`app/engine/scorer.py` implements three metrics:

- cosine similarity,
- dot product,
- negative L2 distance.

The L2 convention is important: scores are `-distance`, so higher is better even for L2.

There are two execution paths:

- NumPy path for environments without torch tensors,
- torch path for tensor-backed or GPU-backed scoring.

Optimization details:

- NumPy uses `np.argpartition` for top-k selection instead of full sorting.
- Torch uses `torch.topk`.
- Optional `apply_l2_norm` can normalize both embeddings and head weights before dot/L2 scoring.
- Cosine scoring normalizes even if `apply_l2_norm` is false, because cosine requires unit vectors.

The scorer is deliberately stateless. All partitioning, model compatibility, and config resolution happen upstream in the store and route logic.

### 6.6 Score API

`app/routes/score.py` receives a `ScoreRequest` shaped like a subset of OpenRTB:

- top-level request ID,
- `user.data[]`,
- each data block contains `segment[]`,
- each segment may contain `ext`.

The request model is defined in `app/models/ortb.py`. The scorer expects `segment.ext` to be a flat object with:

- `ver`
- `vector`
- `model`
- `dimension`
- `type`

There is intentionally very little request-side validation beyond Pydantic shape validation. In particular:

- the route does not verify that `len(vector)` equals `dimension`,
- the route does not verify that request `dimension` matches the registered model dimension,
- the route does not verify that the vector length matches the shape of the compatible campaign heads before calling the scorer.

As a result, malformed inputs can make it through request parsing and fail later inside NumPy or torch arithmetic rather than being rejected cleanly at the HTTP boundary.

Flow:

1. walk all `user.data[].segment[]`,
2. ignore segments with no `ext`,
3. for each embedding, look up model config by `ext.model`,
4. default to `metric="cosine"` and `apply_l2_norm=false` if the model is unknown,
5. resolve type aliases,
6. fetch compatible heads from the store,
7. score,
8. record analytics for each returned scored head,
9. return one `ScoreResponse` per embedding found in the request.

Important behaviors:

- if no embeddings are found at all, the route returns HTTP 400,
- if embeddings are present but no compatible heads exist, the route returns an empty `scores` array rather than an error,
- results are rounded to 6 decimal places in the HTTP response,
- analytics records one record per top-k match, not one record per incoming request.

There is also no batching optimization across embeddings in the same request. If a request contains multiple segments with embeddings, the handler loops over them sequentially and performs a separate top-k call for each one.

### 6.7 Campaign head management API

`app/routes/campaigns.py` implements CRUD-like operations over campaign heads.

`POST /campaigns/heads`

- batch registration with upsert semantics,
- 400 on validation errors such as dimension mismatch.

`PUT /campaigns/heads`

- update existing heads only,
- atomic pre-check: if any head ID does not exist, the whole request fails with 404,
- 400 on invalid shapes or dimension mismatches.

`DELETE /campaigns/heads/{campaign_head_id}`

- deletes one head,
- returns 404 if absent,
- if the deleted head was the last head for a model, that model’s stored config is also removed.

That last behavior is important because it allows the same model name to be reintroduced later with a different dimension.

### 6.8 Analytics subsystem

`app/engine/analytics.py` tracks scored embeddings per campaign head using a bounded `deque`.

Each record stores:

- the scored embedding as float32,
- the resulting score.

`get_analytics()` can filter by:

- `campaign_id`
- `campaign_head_id`

Bucket logic:

- compute percentiles from the configured `score_buckets`,
- fit PCA with `n_components = min(configured_dims, n_features)`,
- project embeddings into PCA space if there are enough samples,
- compute a centroid per percentile bucket.

Output model:

- `CampaignAnalytics`
- list of `ScoreBucket`
- each bucket has a label like `p25-p50`, a count, and an optional PCA centroid.

This is not production observability; it is interpretability tooling. It is trying to answer “what kinds of embeddings end up scoring in the low, medium, and high bands for this campaign head?”

There is one small output nuance worth calling out: `AnalyticsResponse.reduced_dimensions` always reports the configured PCA target dimension, not necessarily the actual dimensionality used for a given head. If the embedding dimension is smaller than the configured PCA dimension, the actual PCA output will use fewer components even though the response metadata still reports the configured value.

### 6.9 Examples and tests

The service includes:

- `examples/sample_campaign_heads.json`
- `examples/sample_score_request.json`

These examples are internally consistent with the FastAPI request models, not with the Prebid adapter’s nested `ext.aa` shape.

The tests cover:

- scoring correctness for cosine, dot, and L2,
- top-k behavior,
- torch and NumPy execution paths,
- cache behavior,
- type alias resolution,
- compatible model matching,
- update atomicity,
- analytics endpoint output shape.

I executed the test suite locally in a repo-scoped virtualenv with:

`./.venv/bin/python -m pytest src/user-embedding-to-campaign-scoring/tests -q`

Observed result:

- `34 passed, 2 skipped in 0.14s`

The skipped cases are the torch-specific tests, which are written with `pytest.importorskip("torch")` and therefore only run when PyTorch is installed.

### 6.10 Concurrency and state model

The scoring service is not backed by an external database in this repository. All registered heads, model configs, caches, and analytics records live in process memory.

Practical consequences:

- state disappears when the process restarts,
- horizontal scaling would require an external replication or registration strategy outside this codebase,
- analytics are per-process rather than globally aggregated,
- asyncio locks protect in-process mutation, but there is no cross-process coordination.

This is consistent with the repo’s “reference sidecar” framing, but it is an important operational constraint.

## 7. Prebid RTD example: what it does

The `prebid-module/` subtree is explicitly labeled as an example, not the canonical maintained implementation.

### 7.1 Purpose

The module shows how to:

- read Agentic Audiences data from browser storage,
- transform each stored embedding entry into an OpenRTB segment,
- merge those segments into `reqBidsConfigObj.ortb2Fragments.global.user.data`.

### 7.2 Module mechanics

`modules/agenticAudienceAdapter.js` defines:

- `MODULE_NAME = "agenticAudience"`
- `DEFAULT_STORAGE_KEY = "_agentic_audience_"`
- a storage manager via Prebid’s `getStorageManager`
- helper readers for localStorage and cookies
- `mapEntryToOpenRtbSegment()`
- `getBidRequestData()`
- a trivial `init()` that always returns `true`

Expected storage format:

- base64-encoded JSON string,
- top-level object with `entries`,
- each entry may include `id`, `name`, `ver`, `vector`, `dimension`, `model`, `type`.

The module:

1. chooses the storage key from `params.storageKey` or the default,
2. reads from localStorage first, then cookies,
3. base64-decodes and JSON-parses the blob,
4. maps `entries[]` into OpenRTB segments,
5. merges a `user.data[0]` object into the global ORTB2 fragment.

The merge behavior uses Prebid’s `mergeDeep`, so the adapter is additive rather than replacing the whole `user` block outright. That makes it safer to coexist with other RTD modules or publisher-provided ORTB2 fragments.

### 7.3 OpenRTB shape produced by the adapter

The adapter writes each embedding into:

`user.data[].segment[].ext.aa`

Specifically:

- `ext.aa.ver`
- `ext.aa.vector`
- `ext.aa.dimension`
- `ext.aa.model`
- `ext.aa.type`

This matches the comments and unit tests in the Prebid example.

### 7.4 Example/test observations

The test suite under `prebid-module/test/spec/modules/agenticAudienceAdapter_spec.js` verifies:

- storage-key selection behavior,
- localStorage and cookie fallback,
- callback behavior when no data exists,
- exact pass-through of `vector` and `type`,
- shape under `user.data[0].segment[0].ext.aa`.

The tests intentionally allow `vector` to be either a base64 payload or a raw array. The adapter does not validate or coerce the embedding contents.

The example also does not gate behavior on consent state beyond exposing the standard `userConsent` parameter in the submodule hooks. `init()` unconditionally returns `true`, and `getBidRequestData()` reads storage whenever data is present. In other words, this example is focused on payload plumbing, not privacy enforcement logic.

### 7.5 Important integration mismatch

There is a direct interface mismatch between the Prebid example and the scoring service:

- Prebid example emits embedding fields under `segment.ext.aa`,
- FastAPI scorer expects them directly at `segment.ext`.

That means the example browser transport and the example scorer are conceptually aligned but not plug-and-play compatible without a translation layer or request-model update.

There is also a second mismatch:

- the Prebid tests allow `type` to be numeric or array-shaped in some cases,
- the scorer expects `type` to be a string that can be resolved through the alias map.

So the browser-side example is intentionally permissive, while the server-side scorer is stricter and closer to the spec text.

## 8. Data model alignment across repo layers

One of the most valuable ways to read this repo is as three partially overlapping data models.

### 8.1 Spec layer

The formal spec uses embedding types such as:

- `context`
- `creative`
- `user_intent`
- `inventory`
- `query`

It also introduces `embedding_space_id`, consent, security, and richer model descriptors.

### 8.2 Prebid transport layer

The Prebid example uses OpenRTB `segment.ext.aa` and only carries the minimum operational fields:

- version,
- vector,
- dimension,
- model,
- type.

It does not carry consent, security, or rich model metadata.

### 8.3 Scoring service layer

The scoring service expects a flattened `segment.ext` and resolves wire names into internal types:

- `contextual`
- `intent`
- `capi`
- `identity`
- `reinforcement`

It stores but does not operationally use `embedding_space_id` beyond registration payload acceptance.

The result is a typical standards-repo pattern:

- the documents define the desired future contract,
- the transport example demonstrates one insertion path,
- the reference scorer implements a narrower operational subset.

Another way to say this is:

- the spec layer is envelope-rich and policy-aware,
- the Prebid layer is transport-oriented and minimal,
- the scorer is math-oriented and only consumes the fields needed for matching.

## 9. Tests as behavioral specification

The tests are useful because they reveal real guarantees that the documentation only hints at.

Key behaviors proven by tests:

- cache hits return the identical cached object instance,
- registering a new head invalidates the cache,
- empty partitions are not cached,
- compatible models can score against one another,
- type alias `context -> contextual` works end to end,
- `PUT /campaigns/heads` is atomic with respect to missing IDs,
- deleting the last head for a model removes model config and permits re-registration with another dimension,
- analytics returns PCA centroids when there are enough samples.

The tests also expose the intended mental model:

- campaign heads are treated as vector rows in a matrix,
- scoring is partitioned by model and canonical type,
- analytics is head-centric rather than campaign-centric.

The tests do not currently exercise several cases that matter in production:

- mismatched request vector length versus registered head dimension,
- conflicting per-head metric declarations under the same model,
- conflicting per-head compatibility declarations under the same model,
- ingestion of the nested `segment.ext.aa` shape produced by the Prebid example.

## 10. What is mature vs. draft in this repository

### 10.1 Most mature

- FastAPI scoring service
- service tests
- type aliasing and model compatibility logic
- Docker packaging
- example Prebid injection flow

### 10.2 Medium maturity

- embedding schema
- architecture docs
- taxonomy

### 10.3 Clearly draft or placeholder

- agent interface schema
- buyer/seller example payloads
- roadmap file
- community docs
- code of conduct file

That distribution matters for anyone presenting the repo: this is not a finished end-to-end product. It is a standards proposal plus a fairly credible scoring-service prototype.

## 11. Operational interpretation

If you were to deploy only the implemented code from this repo, the runtime story would be:

1. train or otherwise generate campaign head vectors elsewhere,
2. register them into the scoring service,
3. send user embeddings to `/score`,
4. read back the top-k scored heads,
5. optionally inspect `/analytics` for coarse embedding-cluster interpretation.

What you would still need outside the repo:

- actual embedding generation infrastructure,
- a training pipeline for campaign heads,
- a clean OpenRTB extension contract shared by transport and scorer,
- authn/authz and privacy enforcement beyond the draft docs,
- bidder-side decision logic.

## 12. Key architectural strengths

- The repo keeps the scoring service narrow and auditable.
- It treats model-space incompatibility seriously instead of assuming vector interchangeability.
- It offers a real implementation rather than only prose specs.
- The analytics layer gives a practical debugging handle on black-box vector scoring.
- The Prebid example anchors the standards work in a familiar ad-tech transport surface.

## 13. Key gaps and caveats

- Prebid example shape and scorer request shape do not currently match.
- `embedding_space_id` is emphasized in docs/specs but only lightly used in code.
- first-head-wins model config behavior is narrower than the docs suggest.
- score requests do not currently enforce vector-length and dimension consistency at the API boundary.
- service state is entirely in-memory and process-local.
- agent-interface work is largely placeholder today.
- several community/process files exist but are empty.
- the repository references a broader ML whitepaper, but the executable code focuses only on scoring, not model generation or training.

## 14. Recommended narrative for an explainer video or presentation

A strong way to present this repository is:

1. start with the problem: sparse IDs are a weak interface for AI-native advertising,
2. explain embeddings as the new interoperable signal carrier,
3. show the end-to-end system from `docs/systems-and-models.md`,
4. drill into the scoring service as the main implemented artifact,
5. show how Prebid can transport embeddings into OpenRTB,
6. call out compatibility, taxonomy, and privacy metadata as the standards layer,
7. end with the honest current-state message: strong prototype plus evolving spec, not yet a fully unified production stack.

## 15. Bottom line

This repository is best understood as an open standards and reference-implementation bridge between two worlds:

- today’s RTB ecosystem and agent workflows,
- tomorrow’s embedding-native advertising stack.

The code that most concretely embodies that bridge is the `user-embedding-to-campaign-scoring` service. The docs and schemas explain the larger interoperable future around it. The Prebid example demonstrates how embeddings can enter the bidstream. The gaps between those pieces are not accidental noise; they are exactly where the repository is still evolving from proposal into standardized, end-to-end implementation.
