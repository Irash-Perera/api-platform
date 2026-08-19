# Design: template-driven LLM usage extraction

Status: **draft for review.** Scope is the `llm-cost` policy only.

## 1. Problem

`llm-cost` reads token usage out of upstream LLM responses to calculate cost. The JSON
field names it looks for are written into Go source, one calculator per provider:

```go
// calculator_openai.go
Usage struct {
    PromptTokens        int64 `json:"prompt_tokens"`
    CompletionTokens    int64 `json:"completion_tokens"`
    PromptTokensDetails struct {
        CachedTokens int64 `json:"cached_tokens"`
    } `json:"prompt_tokens_details"`
} `json:"usage"`
```

Five calculators repeat this pattern with different field names. Three consequences:

**A provider changing a field name requires a code change and a gateway rebuild.** There is
no way for an operator to correct it.

**The same numbers are read twice, two different ways.** `LlmProviderTemplate` already
declares where token counts live, and `token-based-ratelimit` reads them from there. But
`llm-cost` ignores the template and parses independently. Editing a template today changes
rate limiting and leaves cost untouched — the two can silently disagree.

**Nothing else can reuse the result.** A new policy needing token counts must reimplement
per-provider parsing, including SSE merging and Bedrock's binary event-stream framing.

The templates are the natural place for this. They already exist, are already delivered to
the policy engine over xDS, and are already selected per route. They are simply missing the
fields cost needs.

## 2. Goals and non-goals

**Goals**

- Field paths for token usage live in `LlmProviderTemplate` YAML, not in Go.
- Extraction happens once per request; the result is available to any policy.
- Observable output is byte-identical to today, proven by differential test.

**Non-goals**

- `azure-llm-cost` is unchanged by this work. It keeps its own parser and its own metadata
  writes. Migrating it is separate, later work (see section 9).
- No change to how analytics, `llm-cost-based-ratelimit`, or `token-based-ratelimit`
  behave.
- Fee-based charges are not moved into templates (see section 4).

## 3. Architecture

Today one component both finds the numbers and prices them. The design splits it at the
boundary between "what the upstream reported" and "what it costs".

```
   ┌────────┐   request   ┌──────────────┐   request   ┌──────────┐
   │ Client │────────────▶│   Gateway    │────────────▶│ Upstream │
   └────────┘             └──────────────┘             └────┬─────┘
        ▲                                                   │
        │                        response ◀─────────────────┘
        │
        │   ┌──────────────────────────────────────────────────────────┐
        │   │  sdk/ai/llmusage            (NEW, shared)                │
        │   │                                                          │
        │   │   1. decode transport    SSE merge / Bedrock eventstream  │
        │   │   2. resolve template    template_handle → lazy store     │
        │   │   3. extract fields      JSONPath from template           │
        │   │   4. normalize           apply cacheAccounting            │
        │   │   5. publish             SharedContext.Metadata           │
        │   └───────────────────────────┬──────────────────────────────┘
        │                               │  llm:usage:v1
        │           ┌───────────────────┼───────────────────┐
        │           ▼                   ▼                   ▼
        │    ┌────────────┐   ┌──────────────────┐  ┌──────────────┐
        │    │  llm-cost  │   │ token-based-     │  │ future       │
        │    │            │   │ ratelimit        │  │ consumers    │
        └────┤  pricing   │   │ (unchanged)      │  └──────────────┘
             │  + fees    │   └──────────────────┘
             └────────────┘
```

`llmusage` answers "what did the upstream report" — reusable by any policy. `llm-cost`
answers "what does that cost" — its concern alone.

### What moves

| From | To |
|---|---|
| `responseBodyForNormalization`, `mergeSSEEvents`, `bedrock_eventstream.go` | `llmusage` |
| model-name probing (`$.model` → `$.modelVersion` → `$.message.model`) | `llmusage`, driven by template |
| the **scalar token-field extraction** inside each `Normalize()` | `llmusage`, driven by template |

`Normalize()` is not removed wholesale. What leaves it is the scalar field reads — the part a
JSONPath can express. What remains is anything requiring logic over the payload, per
section 4: Gemini's modality arrays and every provider's fee detection. For openai, mistral
and anthropic that leaves `Normalize()` empty and it is deleted; for gemini and bedrock a
reduced version survives.

### What stays in `llm-cost`

The pricing file and its loader, `lookupPricingWithKey`, `selectCalculator` (for fees and
residual normalization), `Adjust()`, `genericCalculateCost`, and all metadata writes.

### Library surface

```go
package llmusage

// Get returns normalized usage for this request, extracting on first call and
// returning the stored value afterwards.
func Get(sc *policy.SharedContext, body []byte) (Usage, error)

// Accumulate appends a stream chunk to the shared buffer and returns it.
// Deduped by chunk.Index so several policies accumulating produce one buffer.
func Accumulate(sc *policy.SharedContext, chunk *policy.StreamBody) []byte
```

### Module placement

`sdk/ai/llmusage` as a **nested module** with its own `go.mod`, depending only on
`sdk/core`. This keeps it in the AI-specific module without making `llm-cost` inherit the
Milvus/etcd/OpenTelemetry graph that `sdk/ai`'s `vectordb` and `embeddings` packages pull
in.

No SDK bump is required: `StreamBody.Index`, `lazy_resource.go` and
`utils.ExtractStringValueFromJsonpath` all exist in `sdk/core` v0.2.4, which `llm-cost`
already pins.

## 4. Boundary: what belongs in a template

A template field is a *location*. Some of what `llm-cost` reads is not a location, and
those stay in Go.

```
  TEMPLATE  (declarative, reusable downstream)
  ─────────────────────────────────────────────
  prompt / completion / total tokens
  cached read tokens
  cache write tokens, 5m and 1h
  reasoning tokens
  audio tokens in / out          scalar form, e.g. OpenAI
  service tier
  request model / response model

  llm-cost  (behavioural, billing-only)
  ─────────────────────────────────────────────
  web search fee     scan choices[].message.annotations[] for url_citation
  grounding fee      length of candidates[].groundingMetadata.webSearchQueries
  modality tokens    match promptTokensDetails[] / candidatesTokensDetails[]
                     on modality == AUDIO | IMAGE, then subtract the cached
                     modality entries  (Gemini)
  audio seconds      unit conversion, billed per minute
  speed              read from the request body
  search context     read from the request body
```

Note the asymmetry in audio tokens. OpenAI reports them as a scalar at
`$.usage.prompt_tokens_details.audio_tokens`, which a path expresses. Gemini reports them as
an entry inside a per-modality array keyed on `modality`, which a path cannot select. So the
template carries the scalar form and Gemini's modality handling stays in Go.

The split is not arbitrary. Everything on the left is a number the upstream reported, which
a rate limiter or quota policy could reuse. Everything on the right is a pricing rule that
only `llm-cost` cares about, and expressing it as a path would require adding array
predicates, aggregations and value maps to a CRD — a mini expression language for five
niche features.

## 5. Template schema extension

Seven new optional fields and one flag. All optional, so every existing template stays
valid.

Field names below are the **OpenAI** spellings, shown as an example. Each provider's template
declares its own paths for the same logical field.

```yaml
cachedTokens:       { location: payload, identifier: $.usage.prompt_tokens_details.cached_tokens }
cacheWriteTokens:   { location: payload, identifier: $.usage.prompt_tokens_details.cache_write_tokens }
cacheWrite1hTokens: { location: payload, identifier: $.usage.cache_creation.ephemeral_1h_input_tokens }
reasoningTokens:    { location: payload, identifier: $.usage.completion_tokens_details.reasoning_tokens }
audioInputTokens:   { location: payload, identifier: $.usage.prompt_tokens_details.audio_tokens }
audioOutputTokens:  { location: payload, identifier: $.usage.completion_tokens_details.audio_tokens }
serviceTier:        { location: payload, identifier: $.service_tier }

cacheAccounting: inclusive | additive      # default: inclusive
```

Anthropic's equivalents are `$.usage.cache_read_input_tokens`,
`$.usage.cache_creation.ephemeral_5m_input_tokens` and `$.usage.service_tier` — note the
tier sits *inside* `usage` there, which is why per-provider paths are necessary rather than
one fixed location.

There is deliberately no `imageOutputTokens`. No provider reports it as a scalar path — the
only source is Gemini's modality array, which stays in Go per section 4.

### Field names verified against official specs

Paths were confirmed against each provider's published API reference, not against the
existing calculators. That surfaced one gap in the current implementation:

| Provider | Finding |
|---|---|
| openai | `prompt_tokens_details.cache_write_tokens` exists (GPT-5.6+) and is **not read today**. OpenAI bills cache writes at **1.25× the uncached input rate** on those models. See the gap analysis below. |
| anthropic | Field shapes confirmed correct. Official: *"Total input tokens is the summation of `input_tokens`, `cache_creation_input_tokens`, and `cache_read_input_tokens`"* — additive, as implemented. `service_tier` is inside `usage`, values `standard\|priority\|batch`, and is **not read**; latent only, since no anthropic entry in the shipped pricing file carries a priority rate. |
| gemini | Confirmed `cachedContentTokenCount` is a subset of `promptTokenCount` (inclusive). Modality data is four arrays of `ModalityTokenCount`, correctly excluded. `trafficType` enum verified — `llm-cost`'s mapping is wrong, see below. |
| mistral | Core fields `prompt_tokens` / `completion_tokens` / `total_tokens` confirmed. `prompt_audio_seconds` is **absent from every official reference** (chat and FIM); needs a live Voxtral response to settle. |
| bedrock | Confirmed `usage` = `inputTokens`, `outputTokens`, `totalTokens`, `cacheReadInputTokens`, `cacheWriteInputTokens`, `cacheDetails[]` of `{inputTokens, ttl}` where **`ttl` is exactly `5m \| 1h`** — matching what `llm-cost` switches on. `serviceTier.type` sits at the **response top level**. `cacheDetails[]` is an array, so its per-TTL split stays in Go. |

### The `cache_write_tokens` gap is two-part

Reading the field is necessary but not sufficient:

1. `llm-cost` does not extract `prompt_tokens_details.cache_write_tokens`.
2. The shipped pricing file has **no cache-write rate** for the GPT-5.6 family. Only 10 of
   224 openai entries carry one, all realtime models.

So this is not a case of code failing to apply a rate it already has. Gateway cost omits a
charge OpenAI genuinely bills, but closing it needs both a code change and pricing data — and
the pricing file is customer-maintained, so the data half may be an operator action rather
than ours.

### Gemini's tier mapping in `llm-cost` matches values that do not exist

Verified against the Vertex AI reference: `usageMetadata.trafficType` has exactly five values —
`TRAFFIC_TYPE_UNSPECIFIED`, `ON_DEMAND`, `ON_DEMAND_PRIORITY`, `ON_DEMAND_FLEX`,
`PROVISIONED_THROUGHPUT`.

`llm-cost` maps `ON_DEMAND_PRIORITY` correctly but then matches `"FLEX"` and `"BATCH"` — **neither is
a real value**. Its own comment documents them as real, which is how the error survived. The genuine
flex value, `ON_DEMAND_FLEX`, is therefore never matched.

Impact today is nil: no Gemini, `vertex_ai` or `vertex_ai-language-models` entry in the shipped
pricing file carries a flex rate (0 of 143), so there is nothing to apply. Priority rates do exist
(12 entries) and priority works.

`llm-cost-v2` implements the **correct** mapping regardless — `ON_DEMAND_PRIORITY` → priority,
`ON_DEMAND_FLEX` → flex, everything else standard. That is spec-correct and costs nothing in parity
terms, because no flex rate exists to diverge on. It becomes right the moment an operator adds one.

### Mistral's `prompt_audio_seconds` is undocumented

`llm-cost` reads `$.usage.prompt_audio_seconds` to bill Voxtral audio per minute. The field appears
in **no** official Mistral reference — neither the chat nor the FIM endpoint documents anything about
audio duration in the usage object. It may be real but undocumented, or wrong. This cannot be settled
from published specs and needs a live Voxtral response to confirm.

### Bedrock's service tier is read by `llm-cost-v2` but not by `llm-cost`

`calculator_bedrock.go` never reads `serviceTier` (0 occurrences). The v2 template declares
`$.serviceTier.type`, confirmed against the Converse reference as sitting at the response top level.
This is a deliberate improvement rather than parity, and it is latent — no bedrock entry carries a
priority rate.

### Service tier is unread for two providers

Anthropic and Bedrock both report a service tier that `llm-cost` ignores. Neither currently
misprices, because the shipped pricing file carries priority rates only for openai (28),
azure (32), azure_ai (10), gemini (5) and vertex_ai (4) — none for anthropic (0 of 25) or
bedrock (0 of 138).

These are therefore **latent**: an operator adding priority rates for those providers would
see them silently ignored. Adding `serviceTier` to both templates costs nothing here and
closes the gap before it can bite. Note Bedrock's tier is at the response top level, so its
path is `$.serviceTier.type`, not under `usage`.

Azure OpenAI and Azure AI Foundry must get the same treatment when `azure-llm-cost` migrates.
Azure serves GPT-5.6 models, so it shares the `cache_write_tokens` exposure, and unlike the
others its pricing entries **do** carry priority rates — so a tier mistake there would
misprice immediately rather than latently.

### `cacheAccounting`

The one thing that is not a location. Providers disagree on whether the cached-token count
is already inside the input total or additional to it:

| Provider | Accounting | Why |
|---|---|---|
| openai | `inclusive` | `cached_tokens` is a subset of `prompt_tokens` |
| gemini | `inclusive` | `cachedContentTokenCount` is a subset of `promptTokenCount` |
| anthropic | `additive` | `promptTokens = input + cache_creation + cache_read` |
| bedrock | varies by response shape | Converse and Anthropic-InvokeModel differ |
| mistral | n/a | no caching |

Because Bedrock varies *within* one provider, the flag sits alongside the extraction fields
so a `resourceMappings` entry can override it. It is not a once-per-template setting.

### Schema change surface

Every layer below prunes unknown fields, so all of them must change together. A field
added only to the YAML is silently dropped.

```
gateway/gateway-controller/api/management-openapi.yaml   → make generate-server-code
                                                            → pkg/api/management/generated.go
kubernetes/gateway-operator/api/v1/llmprovidertemplate_types.go
kubernetes/gateway-operator/api/v1alpha1/llmprovidertemplate_types.go
                                                         → make generate (deepcopy)
                                                         → make manifests (config/crd/bases)
kubernetes/helm/operator-helm-chart/crds/...yaml         → refresh copy of config/crd/bases
gateway/gateway-controller/pkg/config/llm_validator.go
gateway/gateway-controller/default-llm-provider-templates/*.yaml
```

Notes established by inspection:

- **`platform-api` IS a fourth pruning layer.** An earlier draft of this spec claimed otherwise,
  on the basis that `gateway-internal-api.yaml` mentions `LLMProviderTemplateData` only in a
  comment and models the artifact `spec` as a generic object. That was wrong: the spec is
  generic, but the Go code is not. `artifact_import_llm_template.go` calls
  `utils.DecodeSpec(ctx.Configuration.Spec, &specTmpl)` into `model.LLMProviderTemplate`
  (`platform-api/internal/model/llm.go`), a typed struct carrying only the original six fields
  at both levels — so anything else is dropped. The gateway pushes templates here via
  `pushTemplateToControlPlane`, so a template created through the gateway management API is
  mirrored into the control plane **without** the new fields.

  This is mirror and UX divergence rather than runtime data loss, since no control-plane →
  gateway template push was found: the gateway keeps its own correct copy and is what the
  policy reads. But `GET /llm-provider-templates/{id}` on the control plane returns a template
  that disagrees with the gateway's, AI Workspace renders and copies from the lossy version,
  and the doc comment on `templateRuntimeArtifact` in `platform-api/internal/service/llm.go`
  — which asserts it captures "the subset of an LLM provider template that the gateway
  actually consumes at request time" — becomes false once these fields are consumed.

  **This must be closed before the cost policy ships**, and it also means the affected
  surfaces include `platform-api/resources/default-llm-provider-templates/` (a duplicate copy
  of the shipped templates) and the AI Workspace field manifests under
  `portals/ai-workspace/src/utils/`.
- **No conversion code is needed.** `v1alpha1.LlmProviderTemplate.ConvertTo/ConvertFrom` use
  `convertViaJSON`, so conversion follows the JSON tags automatically. The fields must still be
  added to `v1alpha1` as well, or a round-trip through the older version silently drops them.
- **The helm CRDs are byte-identical copies** of `config/crd/bases/`, so they must be
  refreshed after `make manifests` or the chart ships a stale schema that prunes the new
  fields.

## 6. Extraction

### Resolution order

```
1. template_handle   ← SharedContext.Metadata, set by the kernel
2. template          ← lazy store, GetResourceByIDAndType(handle, "LlmProviderTemplate")
3. mapping           ← selectTemplateResourceMapping(template, requestPath)
4. field path        ← mapping field, else template root field   (per field)
5. decode            ← SSE merge or Bedrock eventstream, chosen by path
6. extract           ← ExtractStringValueFromJsonpath per declared field
7. normalize         ← apply cacheAccounting, derive totals
8. publish           ← SharedContext.Metadata["llm:usage:v1"]
```

Step 4 falls back **per field**, not per object: a mapping overriding only `promptTokens`
still inherits `serviceTier` from the template root. This matches how the existing
`/responses` mapping is written.

### Shared matching rule

`pathsMatch`, `selectTemplateResourceMapping` and `shouldPreferTemplateResourceMapping`
currently live in `gateway-controller/pkg/utils/llm_transformer.go`, which policies cannot
import. They move to `sdk/core/utils`, and both the controller-side transformer and
`llmusage` call the same functions.

Reimplementing the rule in the library would let the controller and the extractor select
different mappings for the same path — a divergence that would be very hard to diagnose.

### Streaming

`Accumulate` appends only when `chunk.Index > lastSeenIndex`, storing both the buffer and
the last index in `SharedContext`. Extraction runs once at `EndOfStream`. The buffer is
deleted at `EndOfStream` as the existing policies already do.

Result: one buffer per request regardless of how many policies accumulate, instead of one
buffer per policy.

### Model resolution

The template declares `responseModel` and `requestModel`, and `location: pathParam` already
supports a regex for extracting a model from the path. `llmusage` publishes the resolved
model **and the ordered candidate list**, because a consumer may need the fallback chain
rather than only the winner.

## 7. Compatibility

The observable output must not change. `llm-cost` continues to write `x-llm-cost`,
`x-llm-cost-status` and the four `aitoken:*` keys with the same values and the same
formatting; only the source of the numbers changes.

| Surface | Change |
|---|---|
| `x-llm-cost` / `x-llm-cost-status` | none |
| `aitoken:*` keys | none — same values, same format |
| `llmCost` in analytics | none |
| `llm-cost-based-ratelimit` | none |
| `token-based-ratelimit` | none |
| `azure-llm-cost` | none |
| `llm:usage:v1` | new, additive |

Publishing to `SharedContext.Metadata` has no analytics impact. The analytics system policy
snapshots that map at `OnResponseHeaders`, before the response body phase, so body-phase
writes never reach the generic metadata property. Analytics receives only what a policy
explicitly returns in `AnalyticsMetadata`, which is unchanged.

## 8. Verification

### Frozen reference implementation

The current calculators are copied verbatim into a test-only package that is never edited:

```
policies/llm-cost/
  internal/reference/          frozen copy, test-only
    calculator_openai.go
    calculator_anthropic.go
    calculator_bedrock.go
    calculator_gemini.go
    calculator_mistral.go
    pricing.go
  differential_test.go
```

```
        ┌────────────── same response bytes ──────────────┐
        ▼                                                 ▼
  internal/reference                              template + llmusage
  (frozen)                                        (new path)
        │                                                 │
        └────────────▶  cost_old == cost_new  ◀────────────┘
                        exact equality, build fails on drift
```

Corpus: the 42 usage objects and 152 test cases already in `llm_cost_test.go`, plus real
captures added per provider. A template with a wrong path fails the build rather than
silently mispricing.

`internal/reference/` is deleted in a final PR once all five providers pass.

### Expected divergences

The frozen reference encodes today's behaviour, including its bugs. Verifying against
official specs already found one (`cache_write_tokens`, section 5), and the remaining
providers may surface more. So exact equality cannot be the only rule, or the gate would
block every correction.

Divergences are declared explicitly, per case, with a reason:

```go
// Cases where new behaviour intentionally differs from the frozen reference.
// Anything not listed here must match exactly.
var expectedDivergences = []divergence{
    {
        fixture: "openai_gpt56_cache_write",
        reason:  "reference omits prompt_tokens_details.cache_write_tokens; " +
                 "OpenAI bills cache writes at 1.25x uncached input on GPT-5.6+",
    },
}
```

An undeclared difference fails the build. A declared one must state which official spec
justifies it. This keeps the gate strict while allowing deliberate fixes, and leaves a record
of every behaviour change for review.

Note that adding `serviceTier` to the anthropic and bedrock templates produces **no**
divergence against the shipped pricing file, since neither has priority rates for those
providers. It changes behaviour only for an operator who adds such rates — which is the point
of doing it. So it needs no divergence entry, but its test should assert the tier is
extracted, not merely that cost is unchanged.

Fixing `cache_write_tokens` also needs a pricing-side decision — whether the 1.25×
multiplier is derived in `llm-cost` or carried as a cache-write rate in the pricing file. That
is a billing change, so it lands as its own PR after the migration, not folded into it.

### Failure observability

The failure mode to avoid is a wrong path silently yielding zero.

1. **Declared-but-empty warns.** If a template declares a field and the path yields nothing
   while the response carries a usage object, log at WARN with the template, field and
   path. Deduped per template+field so it cannot flood.
2. **Status key.** `llm:usage:status` = `extracted` | `no_usage` | `template_missing` |
   `path_miss`, mirroring the existing `x-llm-cost-status` pattern.
3. **Deploy-time validation.** `llm_validator.go` rejects a template declaring
   `cacheWriteTokens` without `cacheAccounting`, and rejects unparseable JSONPath, so
   errors surface before billing does.

Cost behaviour on failure is unchanged: unpriced, request unaffected.

## 9. Rollout

Eight PRs, each independently revertable, blast radius one provider.

| PR | Content |
|---|---|
| 1 | Schema extension across all layers + validators |
| 2 | `sdk/ai/llmusage` nested module, `pathsMatch` lifted to `sdk/core/utils`, tests |
| 3 | openai — template updated, `Normalize()` deleted, differential test green |
| 4 | mistral — confirm `prompt_audio_seconds` against a live response, then template, `Normalize()` deleted |
| 5 | anthropic — first `additive` accounting, add `serviceTier`, `Normalize()` deleted |
| 6 | gemini — template covers scalars, `Normalize()` reduced to modality arrays |
| 7 | bedrock — add `serviceTier` at `$.serviceTier.type`; binary framing and per-TTL `cacheDetails[]` keep a reduced `Normalize()` |
| 8 | delete `internal/reference/` |

Simplest providers first so the architecture is proven before the awkward ones. All five
providers have now been checked against their official specs; only Mistral's
`prompt_audio_seconds` remains unconfirmed, which PR 4 resolves against a live response.

The `cache_write_tokens` billing correction is deliberately **not** in this sequence. It is a
billing change rather than a refactor, and belongs in its own PR after the migration so the
two are reviewable separately.

## 10. Known limitations of this design

Recorded for a later discussion. None of these block the current work, but each is a case where an
operator would still need a code change, and none is obvious from the outside.

### The field vocabulary is closed

A template says *where a known value lives*. It cannot introduce a new kind of value. The library
looks for exactly twelve names (`promptTokens`, `completionTokens`, `totalTokens`, `cachedTokens`,
`cacheWriteTokens`, `cacheWrite1hTokens`, `reasoningTokens`, `audioInputTokens`,
`audioOutputTokens`, `serviceTier`, `requestModel`, `responseModel`).

So this design solves the **common** failure — a provider renaming, moving, or per-endpoint
varying a field, all fixable by editing YAML and effective live. It does **not** solve a provider
adding a genuinely new billable dimension: that still needs the full schema change across the
OpenAPI spec, both CRD versions, the validator, and the library. This is not hypothetical —
reasoning tokens, the 5-minute/1-hour cache split, and audio tokens all appeared within the last
few years.

The reason a closed set is defensible is that cost is `numbers × rates × rules`, and only the
numbers are data. A counter the calculator has no billing rule for cannot be priced, so an
arbitrary named counter would move the code change from the schema into the pricing logic rather
than removing it.

If this becomes painful, the middle path is an open counter list where each entry declares a
**billing role** (`input`, `output`, `cache_read`, `cache_write`, `passthrough`) plus its
accounting mode and rate key. A new counter fitting an existing role would then need no code.
That should be designed against the first real provider change that forces a schema edit, not
against a hypothetical one.

### Only the response body is read

The `ExtractionIdentifier` schema allows `location: header`, `pathParam` and `queryParam`, but the
library reads a field only when the location is `payload`:

```go
if !ok || spec.Location != locationPayload {
    return ""
}
```

A template declaring a header location is therefore **silently ignored and yields zero** — the
worst failure shape, since it looks like a free request. If a provider moves a value out of the
body into a response header, that needs code.

Two things should happen regardless of when header support is added: the library should log at
WARN when it encounters a location it cannot handle, and the validator should reject a non-payload
location on the usage fields until the library supports one.

### JSONPath can index arrays but cannot filter them

`utils.ExtractStringValueFromJsonpath` supports fixed indices, including negative ones, so
`$.choices[0].message.content` and `$.usage.details[-1].tokenCount` both resolve. What it has no
support for is a predicate — "the array entry whose `modality` is `AUDIO`".

That is why Gemini's per-modality arrays stay in Go: the entries are order-dependent, so a fixed
index is not safe. The same limit applies to any value that must be *computed* rather than located
— an array length (`llm-cost`'s grounding fee), a conditional count (its web-search fee), or a unit
conversion (Mistral's audio seconds).

## 11. Known future work

Not part of this design, recorded so it is not rediscovered later.

**Foundry needs model-based mapping selection.** `resourceMappings` selects by request
path. On Azure AI Foundry, several model families share `/models/chat/completions` with
different usage behaviour, so path alone cannot distinguish them. The likely extension is an
optional `modelPattern` beside `resource`, reusing the same exact-beats-wildcard preference.
To keep that cheap, mapping selection in `llmusage` should be **one pluggable function**
rather than inlined path matching.

**A proxy's `additionalProviders` collapses onto one template.**
`extractTemplateHandle` resolves only `spec.provider.id`, so a request routed to a secondary
provider receives the primary's template.

**Analytics token coverage is unchanged.** Because `llmusage` runs only when a policy calls
it, routes with no cost policy still report no tokens. Fixing that would require running
extraction unconditionally in the kernel, which this design does not do.

**`azure-llm-cost` migration.** Once this is proven, that policy can drop `usage.go`,
`extractModelName` and its SSE merge, keeping its deployment mapping, region tiers,
`azure/` vs `azure_ai/` catalog selection and pricing math.

### Three fields vanished between the two layers, and one was an active undercharge

The template/Go split has a failure mode worth naming: a field can end up declared in **neither**
layer and simply disappear. Nothing errors, no test fails, and the cost comes out plausible but
wrong. It happened three times.

| Field | Severity | Cause |
|---|---|---|
| Gemini `ServiceTier` | latent | needs mapping (`ON_DEMAND_PRIORITY` → `priority`), so not a path; no `fees` set it |
| Bedrock `CacheWrite1hrTokens` | latent | the per-TTL split needs an array walk; the only code doing it was unreachable |
| Gemini `AudioInputTokens` / `AudioOutputTokens` | **active** | lives in a modality array, so not a path; no `fees` set it |

The third was a real undercharge. 53 gemini and vertex_ai entries carry audio rates that differ
from their text rates — `gemini-2.0-flash` bills audio input at `7e-07` against `1e-07` for text, a
factor of seven. With `AudioInputTokens` left at zero those tokens were billed at the text rate:

```
per 1000 audio input tokens on gemini-2.0-flash
  correct      0.0007000000
  as billed    0.0001000000
  shortfall    0.0006000000
```

The first two were latent only because the shipped pricing file happens to carry no matching rates.

**The guard.** A per-provider coverage test now enumerates every field the original `llm-cost`
calculators set and asserts each arrives via either the template or a `fees` method, by driving a
real response through the policy. Removing a single assignment fails it:

```
AudioInputTokens is zero; the response populated it but it did not reach Usage
```

It must have **no exemption list**. An early version carried a skip-list for the two audio fields,
which suppressed the test for precisely the fields that were broken.

### What `llm:usage:v1` contains, and what it does not

`Publish` is called inside `Get`, before the policy runs `fees`. Nothing publishes afterwards. So
the shared snapshot is the **template extraction**, not the final billing view.

A downstream consumer therefore sees neither the fee-derived fields nor the *refinements* `fees`
makes to template values:

```
Bedrock   template read cacheWriteTokens = 500
          fees split it into 300 @5m + 200 @1h
          llm:usage:v1 still says 500, undifferentiated

Gemini    fees derived ServiceTier = "priority" from trafficType
          llm:usage:v1 has no tier at all
```

`x-llm-cost` is always correct — the policy uses its own local copy for the arithmetic. Only the
shared view of the *inputs* is coarser. A cost-based rate limiter is unaffected; a future token
quota policy reading `llm:usage:v1` would undercount Gemini audio and see Bedrock's cache as one
lump.

Closing it is one line — publish again after `fees` — but `Publish` also sets the status, so a
second call needs care not to overwrite a meaningful one. Recorded as open question 5.

## 12. Decisions taken

**`totalTokens` is read, not derived.** The template declares `totalTokens` and the value
reported by the provider is used, matching current behaviour. Deriving it as input + output
would change output on any provider whose reported total differs from the sum — which we
already know happens, for example Grok on Azure AI Foundry. Changing it would be a separate,
deliberate decision.

**A template declaring no token fields is allowed, not rejected.** The validator does not
require them, because a template may legitimately exist for routing or model extraction
alone. The runtime reports `llm:usage:status = path_miss` instead, so the condition is
visible without blocking deployment.

## 13. Open questions

1. Metadata key name: `llm:usage:v1` is proposed, with the version suffix allowing the shape
   to change later. Existing conventions are mixed (`x-llm-cost`, `aitoken:*`,
   `provider_name`, `template_handle`), so this needs a convention call.
2. Whether `cache_write_tokens` is inclusive in `prompt_tokens` or additional to it. The
   OpenAI reference describes it as "the unadjusted number of prompt tokens written to
   cache" but does not state the relationship explicitly. This must be confirmed before the
   billing fix, since it decides whether those tokens are also billed as plain input.
3. Whether `server_tool_use` in Anthropic's usage object represents a billable item the
   current implementation misses. It appears in the official spec; its treatment has not
   been checked.
4. Whether closing the `cache_write_tokens` gap requires shipping cache-write rates for the
   GPT-5.6 family in the default pricing file, or whether that is left to operators. The
   file is customer-maintained, so this is a product decision rather than an engineering one.
5. Whether `llm:usage:v1` should be republished after `fees` so downstream consumers see the
   refined values rather than the raw template extraction. It affects the published contract, not
   correctness of cost.
