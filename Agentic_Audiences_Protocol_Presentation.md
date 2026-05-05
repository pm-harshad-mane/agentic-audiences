# Agentic Audiences Protocol Presentation Script

This file contains a neutral, technical presenter script for `Agentic_Audiences_Protocol.pdf`. It follows the actual 15-slide deck, but uses `explainer_by_codex.md` to fill in implementation details and limitations that are not fully visible in the slides.

## Slide 1: The Agentic Audiences Blueprint

“This presentation is a technical walkthrough of the Agentic Audiences repository.

The repo combines several different kinds of artifacts: architecture and standards documents, a draft JSON schema and embedding exchange spec, an embedding taxonomy, a FastAPI scoring service under `src/user-embedding-to-campaign-scoring`, and a Prebid Real-Time Data module example under `prebid-module`.

At the most operational level, Agentic Audiences is a proposed protocol plus reference code for transporting user and context embeddings through advertising infrastructure and scoring them against campaign vectors in real time.

So the right way to interpret this repository is not as a finished product, and not as just a specification. It is a mixed artifact: part standards proposal, part reference implementation, and part exploration of how embedding-oriented signal exchange could fit into existing advertising infrastructure.

The core architectural idea is straightforward. Instead of moving only sparse identifiers and segment booleans through the stack, the system moves embeddings that encode identity, context, and behavioral or reinforcement signals. Those embeddings are generated outside the critical RTB path, transported through standard ad-tech surfaces such as OpenRTB, and scored in real time against campaign vectors.

To make the rest of the deck concrete, I’ll use one running example: a user lands on a page about electric vehicles, a publisher-side model generates a contextual embedding for that page and session, that embedding is inserted into OpenRTB, and a DSP-side scorer compares it against campaign heads for something like EV loans, pickup trucks, and general auto insurance. That example is not hardcoded in the repo, but it is the kind of flow the repo is designed around.

As we go through the deck, I’ll keep separating what is already implemented in code from what is still only specified or implied.”

## Slide 2: Sparse Identifiers Are Failing as Interfaces for AI-Native Advertising Models

“This slide states the motivation in representation terms.

Traditional ad-tech interfaces are sparse. They carry identifiers, segment membership, geography flags, and key-value attributes. Those interfaces are serviceable for deterministic filtering, but they are a weak fit for learned models that benefit from dense continuous representations.

The future side of the slide shows the proposed alternative: dense vector representations. The repository’s argument is that embeddings can act as a compact interface layer for models because they compress higher-dimensional identity, contextual, and behavioral data into a form that supports fast similarity math.

This is a design claim, not an empirically proven result inside this repo. The repo does not contain benchmark evidence that this approach outperforms existing approaches in production. What it does contain is a specification and a prototype that assume this style of interface is useful for AI-native workflows.

That is why the recurring idea throughout the repo is a shift from exact boolean targeting toward geometric similarity, while still staying inside sub-100 millisecond RTB timing constraints.”

## Slide 3: Phase 1, Phase 2, Phase 3

“This slide describes the repository’s staged model of adoption.

Phase 1 is structured agent interoperability. In practical terms, that means moving from free-form prompt exchange toward schema-bound payloads between agents.

Phase 2 is learned context modeling. That is the step where exchanged contextual, identity, and behavioral data become training inputs for models that learn embeddings.

Phase 3 is full embedding intelligence exchange. That is the state where vectors, rather than text-heavy payloads, become the main carrier of learned state between systems.

A useful framing detail from the repository is that Agentic Audiences is positioned as a data plane, not a full control plane. The control-plane functions associated with buyer agents, seller agents, or other orchestration layers are assumed to continue existing. This repository is mainly concerned with the representation and exchange of signals.

So this slide should be read as an adoption path for the data format and system interfaces, not as a promise that the repo already implements all three phases.”

## Slide 4: Embeddings Decouple Signal Generation from Real-Time Bidding Constraints

“This slide is one of the key systems diagrams in the repository.

The left side, labeled asynchronous marketplace, corresponds to the audience model. In the architecture docs, this model consumes identity, context, and behavior and generates embeddings. Importantly, the docs say this generation should happen asynchronously and not inline with RTB requests.

The center is the browser or edge layer. That is where embeddings can be stored in first-party storage or otherwise surfaced close to the request path. The docs mention ATS.js as one possible implementation, but the protocol is not tied to ATS.js specifically.

The right side is the campaign scoring sidecar. That sidecar is the main concrete code artifact in the repo. Its job is to perform similarity math between an incoming user embedding and registered campaign head vectors.

Using the running example, this means the EV article visit is turned into an embedding before the bid request happens. The browser or edge layer carries that representation forward. Then, during the actual request, the scorer only has to compare that embedding to a set of stored campaign heads and return the top matches.

The loop at the bottom corresponds to retraining. Marketplace signals, CRM, CAPI, and reinforcement data are assumed to feed training. But the important point is separation of concerns: model inference and training are off-path; vector scoring is on-path.

That separation is one of the cleaner architectural decisions in the repo, because it keeps the real-time service narrow.”

## Slide 5: The Architecture Requires a Federated Ecosystem Rather Than a Monolithic Product

“This slide turns the previous system diagram into an ownership model.

The publisher or publisher-side provider controls the browser storage layer. A specialized provider may operate the inference server that creates embeddings. The DSP runs the campaign scoring sidecar. Advertiser or clean-room environments are expected to handle campaign training. A signal aggregator is assumed to collect feedback for future retraining.

So the architecture is explicitly federated. The protocol boundary is meant to sit between independently operated systems, rather than collapsing the entire stack into one vendor-controlled service.

The repository’s documents also make a specific argument for first-party storage: it can improve latency by avoiding extra fetches during request handling, and it may improve privacy posture because reduced representations can be exchanged instead of raw user-level feature tables.

Whether those advantages hold in a specific deployment is an implementation question. But structurally, the repo is clearly optimizing for a distributed ecosystem model rather than a monolith.”

## Slide 6: The Repository as a Standard and a Prototype

“This slide is best read as a repo map.

There are top-level narrative artifacts such as `README.md`, `explainer.md`, `explainer_by_codex.md`, and `docs/systems-and-models.md`. There is a formal schema at `specs/v1.0/schema/embedding_format.schema.json`. There are draft spec documents like `specs/v1.0/embedding-exchange.md` and `specs/v1.0/embedding-taxonomy.md`.

There is executable code under `src/user-embedding-to-campaign-scoring`. That is the strongest implementation area in the repo. There is also a Prebid RTD example under `prebid-module`.

The maturity differences across these areas are real. Some files are placeholders or empty: `specs/v1.0/schema/agent_interface.schema.json`, `specs/v1.0/examples/buyer_agent_request.json`, `specs/v1.0/examples/seller_agent_response.json`, `specs/roadmap.md`, `community/README.md`, and `CODE_OF_CONDUCT.md`.

So when the slide contrasts blueprint zones and executable structures, that is not just metaphorical. It accurately reflects the repository state: some parts are draft specifications, while the scorer and some surrounding implementation pieces are much more concrete.”

## Slide 7: The Specification Layer

“This slide covers the formal contract layer.

The schema in `embedding_format.schema.json` defines a strict top-level envelope with required fields `spec_version`, `message_id`, `timestamp`, `model`, `context`, and `embeddings`. The `model` block itself requires `id`, `version`, `type`, `architecture`, `dimension`, `metric`, and `embedding_space_id`. The `context` block requires `language`. If `consent` is present, it requires `purposes` and `ttl_seconds`.

There are also optional top-level blocks for `producer`, `identity`, `consent`, `security`, and `extensions`. This is why the repo’s formal layer is more than just a vector payload format. It is trying to describe provenance, compatibility metadata, and policy controls.

The `embedding-exchange.md` document adds transport and interoperability rules. It specifies HTTPS JSON as the primary wire format, NDJSON as an optional streaming format, and CBOR as an optional binary representation. It also states that embeddings are only comparable when embedding space, dimension, metric, normalization, and projection assumptions are compatible.

The taxonomy document complements the schema by classifying embeddings by signal type, temporal scope, and composition. Signal types include identity, contextual, reinforcement, creative, inventory, and query or intent. The repo’s interoperability argument depends heavily on that taxonomy, because it assumes alignment between models is only meaningful when the models were trained over comparable signal families.

One limitation to keep in mind is that the runtime code does not fully implement all of this metadata awareness. The formal spec is richer than the current scorer implementation.”

## Slide 8: The Prebid RTD Module

“This slide focuses on the transport example in `prebid-module`.

The main file is `modules/agenticAudienceAdapter.js`. It defines a Prebid Real-Time Data submodule that reads a base64-encoded JSON blob from localStorage or cookies, parses it, maps each stored entry into an OpenRTB segment, and merges those segments into `reqBidsConfigObj.ortb2Fragments.global.user.data`.

The important output shape is `segment.ext.aa`. Inside that `aa` block, the module writes `ver`, `vector`, `dimension`, `model`, and `type`.

If I keep using the EV-page example, this is the moment where a stored contextual embedding for that page gets pulled out of browser storage and inserted into the outgoing OpenRTB request. In the example module, that payload ultimately lands under `user.data[].segment[].ext.aa`.

The module uses `mergeDeep`, which means it is trying to coexist safely with other RTD modules rather than overwriting the entire user object. That is a useful implementation detail because it shows the example is written with Prebid’s additive configuration model in mind.

The tests under `prebid-module/test/spec/modules/agenticAudienceAdapter_spec.js` verify the storage-key behavior, the localStorage and cookie fallback path, the exact shape under `ext.aa`, and the fact that the module passes through values without much coercion.

That last point matters. The example is intentionally permissive. It does not strongly validate vectors. It does not enforce consent in its code path. Its `init()` function returns `true` unconditionally. So this is a transport example, not a hardened policy enforcement layer.”

## Slide 9: The FastAPI Sidecar

“This slide covers the most complete executable component in the repository.

The scoring service is implemented with FastAPI and Pydantic v2. It exposes `POST /campaigns/heads` for registration, `PUT /campaigns/heads` for updating registered heads, `DELETE /campaigns/heads/{campaign_head_id}` for deletion, `POST /score` for matching, `GET /analytics` for interpretability data, and `GET /health` for backend status.

Internally, startup happens in `app/main.py`. The app loads config, constructs a `CampaignHeadStore`, constructs an `AnalyticsTracker`, and stores both on `app.state`.

The `CampaignHeadStore` is the central in-memory state holder. It stores heads partitioned by `model:canonical_type`. Each stored head holds `campaign_id`, `campaign_head_id`, and a float16 weight vector. It also stores a model-level config object containing `dimension`, `metric`, `apply_l2_norm`, and `compatible_with`.

The service is deliberately narrow. It does not bid, pace, or optimize budgets. It only performs vector matching. That narrowness is a strength from an auditability perspective, but it also means the repo does not claim to solve bidder logic end to end.

In the running example, the scorer would receive the EV-context embedding, look up all registered campaign heads whose model and canonical type are compatible, and then return something like: EV loan campaign scores highest, auto insurance scores second, pickup truck campaign scores lower.

Another implementation detail worth noting is that the service is stateful but process-local. The store, cache, and analytics all live in memory. There is no database layer in the repo.”

## Slide 10: Float16 Optimization and Bounded PCA Analytics

“This slide is split into two parts: scoring math and interpretability.

The scoring engine in `app/engine/scorer.py` supports cosine similarity, dot product, and negative L2 distance. The NumPy path normalizes vectors when needed, uses matrix operations for scoring, and uses `np.argpartition` for efficient top-k extraction. If torch is installed, the service can also take a torch path using `torch.topk` and device-aware tensors.

The float16 point on the slide maps directly to the store implementation. Campaign head weights are stored as `np.float16` to reduce memory pressure. During compute, those arrays are converted to float32 for the arithmetic path.

The analytics side is implemented in `app/engine/analytics.py`. For each scored head, the tracker stores `ScoringRecord` objects containing the incoming embedding and the resulting score. Those records are stored in a bounded deque, with max length controlled by config.

When the analytics endpoint is queried, the tracker computes percentile cut points from configured score buckets, fits PCA with `n_components` bounded by sample count and feature count, and computes a reduced centroid for each percentile bucket.

So in the EV-page example, if that type of embedding repeatedly scores in the top bucket for an EV financing campaign head, the analytics output would show that high-scoring bucket accumulating similar contextual embeddings and would expose a centroid for that region in reduced space.

So the analytics endpoint is not trying to be full observability. It is trying to give a debugging and interpretation surface for the scorer: what do embeddings in low, medium, and high score ranges look like after dimensionality reduction.”

## Slide 11: Vector Compatibility Is Conditional

“This slide is a direct reflection of one of the repo’s most important design positions: embeddings are not assumed to be interoperable just because they have the same length.

In the concrete scorer implementation, there are three main matching mechanisms.

First, exact model-name match. If the incoming embedding model name and the stored head model name are the same, the store treats them as compatible.

Second, declared compatibility. The store’s `is_compatible` method checks whether one model declares the other in `compatible_with`, or vice versa.

Third, type aliasing. The config layer resolves wire-format types into canonical internal names. By default, `context` maps to `contextual`, `creative` maps to `capi`, `user_intent` and `query` map to `intent`, and `inventory` maps to `contextual`.

There is also an implementation caveat that matters operationally. The first registered head for a model establishes that model’s dimension and scoring config in memory. Later heads for the same model must match the dimension, but they do not replace that stored config. So the effective runtime behavior is ‘first head wins’ at the model level.

That means compatibility is explicit, but it is also stateful and sensitive to registration order in the current implementation.”

## Slide 12: Intentional Data Model Mismatches Across Layers

“This slide is important because it explains why the repo can feel internally inconsistent if you only read one layer.

The formal spec is the richest representation. It expects a policy-aware envelope with fields like `embedding_space_id`, consent structures, and security blocks. The Prebid layer is transport-oriented and emits a much thinner payload under `segment.ext.aa`. The scoring service is narrower still and expects a flat `segment.ext` object with just the fields it needs for runtime math.

This is partly intentional. Each layer has a different job. The formal spec defines the long-term contract. The Prebid example shows one path for OpenRTB insertion. The scorer only implements the matching primitive.

Using the running example, the intended end-to-end flow is conceptually simple: generate EV-page embedding, insert it into OpenRTB, score it against campaign heads. But the current repo does not yet make that flow mechanically seamless because the Prebid example emits `ext.aa` while the scorer expects a flattened `ext`.

But there is also a genuine integration mismatch here. The Prebid example emits fields inside `ext.aa`, while the scorer’s request model expects those fields directly under `ext`. So as the code exists today, those two components are not directly interoperable without translation or request-model changes.

There is another mismatch around metadata usage. The scorer accepts `embedding_space_id` in campaign head registration, but does not use that field in partitioning or runtime matching. So some of the formal spec’s compatibility model is not yet realized in the running service.

The repository is therefore useful as a technical prototype, but not yet internally unified as one end-to-end wire contract.”

## Slide 13: Maturity Map

“This slide is a reasonable technical summary of repo maturity.

The strongest part of the repository is the core scorer path: registration, in-memory storage, compatibility filtering, NumPy and optional torch compute, analytics, and tests.

The middle tier is the architecture and schema work. Those documents are substantive and coherent. They define a plausible interoperability model. But they are still draft-level from a standards perspective.

The least mature layer is multi-agent interface standardization. The repo contains placeholders where a fuller agent interface and example buyer or seller payloads would eventually live.

I also verified the scorer tests locally. The service test suite under `src/user-embedding-to-campaign-scoring/tests` ran with 34 passing tests and 2 skipped tests. The skipped tests are torch-specific and only run when torch is installed.

So the cleanest summary is: mature scorer implementation, evolving spec and documentation layer, and still-draft broader agent negotiation surface.”

## Slide 14: What Is Missing for Deployment

“This slide is where the repository is most explicit about its boundaries.

The repo does not include embedding generation infrastructure. It does not include campaign head training pipelines. It does not include bidder-side decision logic. And it does not provide a complete production auth, privacy, and consent enforcement stack.

There are also runtime limitations inside the scorer. All state is process-local. Registered heads, model configs, caches, and analytics are stored in memory. That means restart loses state, and horizontal scaling would require external coordination or a backing store that is not part of this repository.

There are request validation gaps too. The `/score` path parses the shape of incoming embeddings, but it does not fully verify that `len(vector)` equals `dimension` before the math path runs, nor does it fully enforce model-dimension consistency at the HTTP boundary. So malformed requests can fail deeper in the compute path rather than being rejected early.

Another missing piece is policy enforcement. The spec documents spend real effort on consent, TTL, and security metadata. The transport example and scorer do not yet enforce those semantics as a complete runtime system.

So the right takeaway is not that the repo is incomplete in a vague sense. It is more specific: the repository contains the standard and the scorer primitive, but the surrounding production system still has to be built around it.”

## Slide 15: Practical Interoperability Bridge

“The final slide summarizes what the repository is actually useful for today.

It is useful as an auditable scoring primitive. The service has narrow scope, which makes it easy to inspect and reason about. That is valuable if a DSP wants to evaluate the math path without adopting a full opaque product.

It is useful as a practical OpenRTB-adjacent transport experiment. The Prebid example shows one way to move embeddings through current ad-tech plumbing without inventing an entirely separate request surface.

It is useful as a standards discussion starter. The schema, taxonomy, and architecture documents define a fairly concrete model for embedding comparability, signal taxonomy, and role boundaries across the ecosystem.

And it is useful as a way to surface the real engineering problems in this area. The repo does not hide the hard parts. It exposes vector-space incompatibility, runtime-versus-spec mismatches, missing production infrastructure, and the difference between a transport example and a fully enforced policy system.

So the neutral conclusion is this: Agentic Audiences is a technically interesting repository because it shows how an embedding-oriented advertising interface could be structured, and because it includes a real scorer implementation. At the same time, it remains a partial system. The strongest code is the scoring service. The formal standard is broader than the implementation. And the end-to-end production story still requires substantial external infrastructure and standardization work.”
