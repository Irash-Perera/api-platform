# LLM Usage Extraction — Plan 3 of 3: `llm-cost-v2` as a Local Dev Policy

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `llm-cost-v2` as a local dev policy that computes LLM cost from
**template-declared** field paths, producing costs identical to `llm-cost` — which is left
completely untouched. This is for local verification, not for shipping.

**Architecture:** The policy is authored in `gateway/dev-policies/llm-cost-v2` inside the
api-platform repo, so it can import `sdk/ai/llmusage` directly from the same tree — no publishing
and no vendored copy. Token extraction comes from the route's `LlmProviderTemplate`; pricing
lookup, cost arithmetic and the fee logic that cannot be expressed as a path are ported from
`llm-cost` unchanged.

**Tech Stack:** Go 1.26.2, with `sdk/core` and `sdk/ai/llmusage` resolved through relative
`replace` directives (no workspace changes).

**Reference spec:** `gateway/spec/llm-usage-extraction-design.md`.
**Depends on:** Plan 1 (template schema fields) and Plan 2 (`sdk/ai/llmusage`), both uncommitted in
the working tree.

## Two repositories

```
PLATFORM   /Users/irashperera/Documents/APIM/api-platform/dev/api-platform      ← all work happens here
POLICIES   /Users/irashperera/Documents/APIM/api-platform/dev/gateway-controllers ← READ ONLY this plan
```

`POLICIES` is read-only for this plan: `llm-cost` is copied *from* it and must not be modified.

## Verified facts this plan relies on

Each was established by experiment, not assumption:

- **No `go.work` entry is needed.** Two relative `replace` directives in the policy's own `go.mod`
  make it self-contained — it builds and tests with `GOWORK=off`, exactly like the existing
  `azure-llm-cost` dev policy.
- **One replace is enough.** `sdk/ai/llmusage` is unpublished so it needs a replace, but it carries
  its own unexported path-matching helpers and otherwise imports only published `sdk/core` symbols.
  Verified: it builds and its 29 tests pass against the published `sdk/core` with `GOWORK=off`.
- **Go ignores `replace` in dependency modules.** The policy is a dependency of the policy-engine,
  so the policy's own replaces do nothing for a gateway image build. Simulating the builder —
  including the `plugin_registry.go` import it generates — fails with
  `unknown revision sdk/ai/llmusage/v0.0.0` until the **same replace** is added to the
  policy-engine's `go.mod`. It already uses that pattern for `common`.
- **`sdk/core` is never replaced and never modified.** An earlier revision put the path-matching
  helpers there and had the gateway-controller delegate to them. That broke `make build-controller`:
  the controller compiles in Docker against its pinned published `sdk/core` v0.2.18, which has no
  such file. The helpers now live inside `llmusage`, the controller is untouched, and nothing needs
  `sdk/core` replaced anywhere.
- **The whole arrangement builds.** `make build` completes with exit 0 and both images are produced,
  with the log showing `llm-cost-v2` resolved via `filePath` and discovered at index 21.
- `llm-cost` builds and passes its full suite on `sdk/core` v0.3.4 (it currently pins v0.2.4), so
  the newer SDK is safe for identical logic.
- `genericCalculateCost` computes
  `PromptTokens - CachedReadTokens - CacheWriteTokens - CacheWrite1hrTokens - AudioInputTokens`,
  so `PromptTokens` must carry the **full** input total and the pricing code does the subtraction.

## Global Constraints

- **Never commit and never stage, in either repo.** Do not run `git commit`, `git add`,
  `git stash`, `git checkout`, `git restore`, or `git reset`. The repository owner commits.
  (`git checkout <file>` has already destroyed needed work once in this project — do not use it.)
- **`llm-cost` must not be modified.** Not one byte. It is the reference this plan is measured
  against and stays in production use.
- Do not touch the owner's unrelated uncommitted files: `gateway/build-manifest.yaml`,
  `gateway/configs/config.toml`, `gateway/configs/config-template.toml`,
  `gateway/configs/llm-pricing/model_prices.json`. `gateway/build.yaml` is modified by Task 1 only.
- `llm-cost-v2` writes the **same** metadata keys as `llm-cost` (`x-llm-cost`,
  `x-llm-cost-status`, `aitoken:*`), so **the two must never be attached to the same route** —
  both would write `x-llm-cost` and the last to run would win.
- Cost must be **identical** to `llm-cost` for every ported case. A difference is a defect in the
  new policy, not a reason to change the expectation — unless it is one of the three documented
  divergences in Task 5.
- Comments: short and explanatory. NEVER write a comment describing a fix, a change, or history
  (no "fixed this", "ported from X", "previously this did Y"). Copied files keep their original
  comments; do not annotate them as copies.
- Every task ends with tests passing under `go clean -testcache`.

## Temporary entries, to be removed before any commit

This policy is a local dev artifact. These exist only because `sdk/ai/llmusage` and
`sdk/core/utils/pathmatch.go` are unpublished; when they are released, every one of them goes away.
Record them in your report at every task so they are not forgotten:

1. `gateway/dev-policies/llm-cost-v2/go.mod` — one replace (`sdk/ai/llmusage`)
2. `gateway/gateway-runtime/policy-engine/go.mod` — the **same** replace, needed only for
   `make build`
3. `gateway/build.yaml` — the `filePath` entry (`dev-policies/README.md` already says to remove it)

No `go.work` change is required or wanted.

## File Structure

```
PLATFORM/gateway/dev-policies/llm-cost-v2/
  go.mod
  pricing.go             copied from llm-cost: pricing load, lookup, cost arithmetic
  calculator_*.go        copied from llm-cost, reduced to fee logic only (5 files)
  bedrock_eventstream.go copied from llm-cost
  fees.go                NEW: the usage bridge + fee application
  llm_cost_v2.go         NEW: the policy
  policy-definition.yaml NEW
  testdata/model_prices.json   copied from llm-cost
  fees_test.go           NEW
  llm_cost_v2_test.go    NEW: parity tests

PLATFORM/gateway/gateway-runtime/policy-engine/go.mod    one replace (build only)
PLATFORM/gateway/build.yaml                              one filePath entry
PLATFORM/gateway/gateway-controller/default-llm-provider-templates/*.yaml   new fields (5 files)
```

No `docs/` or catalog work: a dev policy is not in the shipped policy catalog.

---

### Task 1: Prove the wiring end to end before writing real logic

The point of this task is to fail fast. It stands up a stub policy that imports the library and
confirms the whole toolchain — module resolution via the replace, the policy-engine replace, and
an actual gateway build — before any real code is written against those assumptions.

**Files:**
- Create: `gateway/dev-policies/llm-cost-v2/go.mod`
- Create: `gateway/dev-policies/llm-cost-v2/llm_cost_v2.go` (stub, replaced in Task 3)
- Create: `gateway/dev-policies/llm-cost-v2/policy-definition.yaml`
- Modify: `gateway/gateway-runtime/policy-engine/go.mod`
- Modify: `gateway/build.yaml`

**Interfaces:**
- Consumes: `sdk/ai/llmusage` from Plan 2.
- Produces: a buildable policy module named
  `github.com/wso2/gateway-controllers/policies/llm-cost-v2`. Tasks 2-6 build inside it.

- [ ] **Step 1: Create the module**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/gateway/dev-policies
mkdir -p llm-cost-v2
cat > llm-cost-v2/go.mod <<'EOF'
module github.com/wso2/gateway-controllers/policies/llm-cost-v2

go 1.26.2

require (
	github.com/wso2/api-platform/sdk/ai/llmusage v0.0.0
	github.com/wso2/api-platform/sdk/core v0.3.4
)

replace github.com/wso2/api-platform/sdk/ai/llmusage => ../../../sdk/ai/llmusage
EOF
```

The module path deliberately keeps the `gateway-controllers/policies/` prefix so that moving this
policy into the policies repo later needs no import rewrites.

One replace is enough: it reaches the unpublished library, which carries its own path-matching
helpers and otherwise uses only published `sdk/core` symbols. That makes the module self-contained,
so **no `go.work` entry is required** — the same arrangement the existing `azure-llm-cost` dev
policy uses.

- [ ] **Step 2: Write a stub policy that actually uses the library**

Create `llm_cost_v2.go`. It must reference the library so the import is genuinely exercised:

```go
package llmcostv2

import (
	"context"
	"log/slog"

	policy "github.com/wso2/api-platform/sdk/core/policy/v1alpha2"
	"github.com/wso2/api-platform/sdk/ai/llmusage"
)

// LLMCostV2Policy prices LLM calls using the token locations declared in the
// route's provider template.
type LLMCostV2Policy struct{}

// GetPolicy builds the policy from its system parameters.
func GetPolicy(_ policy.PolicyMetadata, _ map[string]interface{}) (policy.Policy, error) {
	return &LLMCostV2Policy{}, nil
}

// Mode declares the processing requirements. The request body is buffered so the
// model name it carries is readable in the response phase; the response arrives
// as chunks so streaming and buffered responses take one path.
func (p *LLMCostV2Policy) Mode() policy.ProcessingMode {
	return policy.ProcessingMode{
		RequestHeaderMode:  policy.HeaderModeSkip,
		RequestBodyMode:    policy.BodyModeBuffer,
		ResponseHeaderMode: policy.HeaderModeSkip,
		ResponseBodyMode:   policy.BodyModeStream,
	}
}

// NeedsMoreResponseData always returns false; chunks are accumulated manually.
func (p *LLMCostV2Policy) NeedsMoreResponseData(_ []byte) bool {
	return false
}

// OnResponseBodyChunk accumulates the response and reports extracted usage.
func (p *LLMCostV2Policy) OnResponseBodyChunk(
	_ context.Context,
	respCtx *policy.ResponseStreamContext,
	chunk *policy.StreamBody,
	_ map[string]interface{},
) policy.StreamingResponseAction {
	accumulated := llmusage.Accumulate(respCtx.SharedContext, chunk)
	if !chunk.EndOfStream {
		return policy.ForwardResponseChunk{}
	}

	usage, err := llmusage.Get(respCtx.SharedContext, accumulated, nil, respCtx.RequestPath)
	if err != nil {
		slog.Warn("llm-cost-v2: could not extract usage", "error", err)
		return policy.ForwardResponseChunk{}
	}

	slog.Info("llm-cost-v2: extracted usage",
		"model", usage.Model,
		"inputTokens", usage.TotalInputTokens,
		"outputTokens", usage.OutputTokens)
	return policy.ForwardResponseChunk{}
}
```

- [ ] **Step 3: Write a minimal policy definition**

Create `policy-definition.yaml`:

```yaml
name: llm-cost-v2
version: v0.1.0
description: |
  Calculates the monetary cost of LLM API calls at response time using the token
  field locations declared in the route's LlmProviderTemplate, and stores the
  result in SharedContext.Metadata under "x-llm-cost".

  Writes the same metadata keys as the llm-cost policy, so the two must never be
  attached to the same route.

parameters:
  type: object
  additionalProperties: false
  properties: {}
  required: []

systemParameters:
  type: object
  additionalProperties: false
  properties:
    pricing_file:
      type: string
      description: >
        Path to the model pricing JSON file shipped with the gateway image.
      "wso2/defaultValue": "${config.policy_configurations.llm_cost_v2.pricing_file}"
```

- [ ] **Step 4: Confirm the policy builds and the library import resolves**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/gateway/dev-policies/llm-cost-v2
go build ./... && echo "  policy builds, library import resolves"
```

Expected: the success line. A resolution error here means the replace in Step 1 is wrong — check
that the relative path resolves to `PLATFORM/sdk/ai/llmusage`.

- [ ] **Step 5: Add the policy-engine replace so the image build can resolve the library**

Go ignores `replace` directives in dependency modules, so the policy's own replace does nothing
here — the policy-engine is the main module and needs its own. Without it the builder's
`go mod tidy` fails with `unknown revision sdk/ai/llmusage/v0.0.0`.

In `gateway/gateway-runtime/policy-engine/go.mod`, add it beside the existing replace for `common`:

```go
replace github.com/wso2/api-platform/sdk/ai/llmusage => ../../../sdk/ai/llmusage
```

Do **not** add a replace for `sdk/core`. The library needs only published `sdk/core` symbols, and
replacing it would force the policy-engine off the version it pins for no benefit.

- [ ] **Step 6: Register the policy in the gateway build**

In `gateway/build.yaml`, add an entry immediately after the existing `llm-cost` entry, preserving
alphabetical order:

```yaml
  - name: llm-cost-v2
    filePath: ./dev-policies/llm-cost-v2
```

Note that `gateway/build.yaml` already carries an uncommitted `filePath` entry for
`azure-llm-cost` belonging to the owner. Add your entry without disturbing theirs.

- [ ] **Step 7: Verify the policy-engine still resolves and builds**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/gateway/gateway-runtime/policy-engine
go build ./... 2>&1 | head -5 && echo "  policy-engine builds"
```

Expected: the success line.

- [ ] **Step 8: Attempt a real gateway build**

This is the step this task exists for. Run the gateway build and record exactly what happens:

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/gateway
make build 2>&1 | tail -40
```

If it succeeds, the whole approach is proven and later tasks can proceed with confidence.

If it fails, **do not improvise a fix.** Capture the full error, note which stage failed
(discovery, go.mod update, tidy, compile, or packaging), and report BLOCKED. The remedy may be a
build-flag change or publishing the SDK, and that is the owner's call — not something to work
around silently.

- [ ] **Step 9: Confirm nothing unexpected changed**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform
git status --short
git -C ../gateway-controllers status --short
```

Expected: your new files plus the three temporary registrations, the owner's pre-existing files
unchanged, and the policies repo clean. Report the full lists.

- [ ] **Step 10: Leave uncommitted and report**

Do not commit or stage. State explicitly in your report whether `make build` succeeded, and list
the three temporary entries.

---

### Task 2: Port pricing lookup and cost arithmetic

**Files:**
- Create: `gateway/dev-policies/llm-cost-v2/pricing.go` (copied)
- Create: `gateway/dev-policies/llm-cost-v2/testdata/model_prices.json` (copied)

**Interfaces:**
- Consumes: nothing.
- Produces: `ModelPricing`, `loadPricingFromFile`, `lookupPricingWithKey`, `genericCalculateCost`,
  `Usage`, `selectCalculator`, `providerCalculator` — all keeping their existing names. Tasks 3, 4
  and 5 use them.

- [ ] **Step 1: Copy the file and the pricing fixture verbatim**

```bash
SRC=/Users/irashperera/Documents/APIM/api-platform/dev/gateway-controllers/policies/llm-cost
DST=/Users/irashperera/Documents/APIM/api-platform/dev/api-platform/gateway/dev-policies/llm-cost-v2
cp "$SRC"/pricing.go "$DST"/pricing.go
mkdir -p "$DST"/testdata && cp "$SRC"/testdata/model_prices.json "$DST"/testdata/
```

- [ ] **Step 2: Change only the package clause**

`pricing.go` declares `package llmcost`. Change it to `package llmcostv2`. Make **no other edit** —
a reviewer must be able to diff it against `llm-cost/pricing.go` and see only that one line differ.

- [ ] **Step 3: Record exactly what does not compile yet**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/gateway/dev-policies/llm-cost-v2
go build ./... 2>&1 | head -20
```

Expected: errors naming the five calculator types that `selectCalculator` returns
(`OpenAICalculator`, `AnthropicCalculator`, `GeminiCalculator`, `MistralCalculator`,
`BedrockCalculator`) as undefined. They arrive in Task 4.

Record the exact error list in your report — it is the checklist Task 4 must satisfy. Do not stub
them out and do not delete `selectCalculator`.

- [ ] **Step 4: Record the full `Usage` field list**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/gateway/dev-policies/llm-cost-v2
sed -n '/^type Usage struct/,/^}/p' pricing.go
```

Copy the complete field list into your report. Task 4 must account for every fee-related field, and
a field missed there means a charge silently disappears.

- [ ] **Step 5: Leave uncommitted and report**

Do not commit or stage. Confirm `git -C ../../../../gateway-controllers status --short` shows the
policies repo still clean.

---

### Task 3: The policy — hooks, extraction, metadata

**Files:**
- Modify: `gateway/dev-policies/llm-cost-v2/llm_cost_v2.go` (replaces Task 1's stub)

**Interfaces:**
- Consumes: `llmusage` (Plan 2); `ModelPricing`, `loadPricingFromFile`, `lookupPricingWithKey`,
  `genericCalculateCost`, `selectCalculator` (Task 2).
- Produces: `GetPolicy`, `LLMCostV2Policy`, `costResult`, `setCostMetadata`, and the metadata key
  constants. Task 5 tests through `GetPolicy`.

- [ ] **Step 1: Read the original for the contract to reproduce**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/gateway-controllers/policies/llm-cost
sed -n '40,70p' llm_cost.go
sed -n '226,290p' llm_cost.go
```

The observable contract: `x-llm-cost` formatted `"%.10f"`; `x-llm-cost-status` either `calculated`
or `not_calculated`; and `AnalyticsMetadata` carrying `x-llm-cost` plus the four `aitoken:*` keys
when a cost was calculated.

- [ ] **Step 2: Replace the stub with the real policy**

Rewrite `llm_cost_v2.go`:

```go
package llmcostv2

import (
	"context"
	"fmt"
	"log/slog"
	"strconv"

	"github.com/wso2/api-platform/sdk/ai/llmusage"
	policy "github.com/wso2/api-platform/sdk/core/policy/v1alpha2"
)

// Metadata keys match llm-cost so this policy is a drop-in. The two must never
// be attached to the same route.
const (
	MetadataLLMCost       = "x-llm-cost"
	MetadataLLMCostStatus = "x-llm-cost-status"

	costStatusCalculated    = "calculated"
	costStatusNotCalculated = "not_calculated"

	metadataPromptTokenCount     = "aitoken:prompttokencount"
	metadataCompletionTokenCount = "aitoken:completiontokencount"
	metadataTotalTokenCount      = "aitoken:totaltokencount"
	metadataModelID              = "aitoken:modelid"
)

// LLMCostV2Policy prices LLM calls using the token locations declared in the
// route's provider template.
type LLMCostV2Policy struct {
	pricingMap map[string]ModelPricing
}

// GetPolicy builds the policy from its system parameters. The pricing file is
// read once at startup.
func GetPolicy(_ policy.PolicyMetadata, params map[string]interface{}) (policy.Policy, error) {
	pricingFile, ok := params["pricing_file"].(string)
	if !ok || pricingFile == "" {
		return nil, fmt.Errorf("llm-cost-v2: pricing_file is required")
	}

	pricingMap, err := loadPricingFromFile(pricingFile)
	if err != nil {
		return nil, fmt.Errorf("llm-cost-v2: failed to load pricing file %q: %w", pricingFile, err)
	}

	slog.Info("llm-cost-v2: pricing map loaded", "path", pricingFile, "entries", len(pricingMap))
	return &LLMCostV2Policy{pricingMap: pricingMap}, nil
}

// Mode declares the processing requirements. The request body is buffered so the
// model name it carries is readable in the response phase; the response arrives
// as chunks so streaming and buffered responses take one path.
func (p *LLMCostV2Policy) Mode() policy.ProcessingMode {
	return policy.ProcessingMode{
		RequestHeaderMode:  policy.HeaderModeSkip,
		RequestBodyMode:    policy.BodyModeBuffer,
		ResponseHeaderMode: policy.HeaderModeSkip,
		ResponseBodyMode:   policy.BodyModeStream,
	}
}

// NeedsMoreResponseData always returns false; chunks are accumulated manually.
func (p *LLMCostV2Policy) NeedsMoreResponseData(_ []byte) bool {
	return false
}

// OnResponseBodyChunk accumulates the response and prices it at end of stream.
func (p *LLMCostV2Policy) OnResponseBodyChunk(
	_ context.Context,
	respCtx *policy.ResponseStreamContext,
	chunk *policy.StreamBody,
	_ map[string]interface{},
) policy.StreamingResponseAction {
	accumulated := llmusage.Accumulate(respCtx.SharedContext, chunk)

	if !chunk.EndOfStream {
		return policy.ForwardResponseChunk{}
	}

	var requestBody []byte
	if respCtx.RequestBody != nil && respCtx.RequestBody.Present {
		requestBody = respCtx.RequestBody.Content
	}

	result := p.price(respCtx.SharedContext, accumulated, requestBody, respCtx.RequestPath)
	setCostMetadata(respCtx.SharedContext, result)

	analyticsMetadata := map[string]any{MetadataLLMCost: result.cost}
	if result.calculated {
		analyticsMetadata[metadataModelID] = result.modelKey
		analyticsMetadata[metadataPromptTokenCount] = strconv.FormatInt(result.promptTokens, 10)
		analyticsMetadata[metadataCompletionTokenCount] = strconv.FormatInt(result.completionTokens, 10)
		analyticsMetadata[metadataTotalTokenCount] = strconv.FormatInt(result.totalTokens, 10)
	}
	return policy.ForwardResponseChunk{AnalyticsMetadata: analyticsMetadata}
}

// costResult carries the outcome of pricing one response.
type costResult struct {
	cost             float64
	modelKey         string
	promptTokens     int64
	completionTokens int64
	totalTokens      int64
	calculated       bool
}

// price resolves usage from the route's template, looks up the model, and
// computes the cost. Every failure yields an uncalculated result and leaves the
// response untouched.
func (p *LLMCostV2Policy) price(sc *policy.SharedContext, body, requestBody []byte, requestPath string) costResult {
	if len(body) == 0 {
		slog.Warn("llm-cost-v2: empty response body, skipping cost calculation")
		return costResult{}
	}

	extracted, err := llmusage.Get(sc, body, requestBody, requestPath)
	if err != nil {
		slog.Warn("llm-cost-v2: could not extract usage", "path", requestPath, "error", err)
		return costResult{}
	}
	if extracted.Model == "" {
		slog.Warn("llm-cost-v2: no model name in response or request", "path", requestPath)
		return costResult{}
	}

	pricing, modelKey, found := lookupPricingWithKey(p.pricingMap, extracted.Model)
	if !found {
		slog.Warn("llm-cost-v2: no pricing entry for model, setting cost to 0",
			"model", extracted.Model, "candidates", extracted.ModelCandidates)
		return costResult{}
	}

	usage := toPricingUsage(extracted)

	calc := selectCalculator(pricing.Provider)
	if calc != nil {
		usage = applyFees(calc, usage, body, requestBody)
	}

	cost := genericCalculateCost(usage, pricing)
	if calc != nil {
		cost = calc.Adjust(cost, usage, pricing)
	}

	return costResult{
		cost:             cost,
		modelKey:         modelKey,
		promptTokens:     usage.PromptTokens,
		completionTokens: usage.CompletionTokens,
		totalTokens:      usage.TotalTokens,
		calculated:       true,
	}
}

// setCostMetadata publishes the cost and its status for downstream policies.
func setCostMetadata(sc *policy.SharedContext, result costResult) {
	if sc == nil {
		return
	}
	if sc.Metadata == nil {
		sc.Metadata = make(map[string]interface{})
	}

	status := costStatusNotCalculated
	if result.calculated {
		status = costStatusCalculated
	}
	sc.Metadata[MetadataLLMCost] = fmt.Sprintf("%.10f", result.cost)
	sc.Metadata[MetadataLLMCostStatus] = status
}
```

`toPricingUsage` and `applyFees` arrive in Task 4, so this will not compile yet.

- [ ] **Step 3: Confirm only the two Task 4 functions and the five calculators are missing**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/gateway/dev-policies/llm-cost-v2
go build ./... 2>&1 | head -20
```

Expected: `undefined: toPricingUsage`, `undefined: applyFees`, plus Task 2's five calculator types.
Anything else is a mistake in this file — fix it before moving on.

- [ ] **Step 4: Leave uncommitted and report**

Do not commit or stage. Confirm the policies repo is still clean.

---

### Task 4: Bridge extracted usage into pricing, and port the fee logic

This task decides whether costs match, so it deserves the most care. Two jobs: map the
template-extracted usage onto the struct `genericCalculateCost` expects, and port the per-provider
charges that cannot be expressed as a path.

**Files:**
- Create: `gateway/dev-policies/llm-cost-v2/fees.go`
- Create: `gateway/dev-policies/llm-cost-v2/fees_test.go`
- Create: the five `calculator_*.go` files and `bedrock_eventstream.go` (copied, then reduced)

**Interfaces:**
- Consumes: `llmusage.Usage`; `Usage`, `ModelPricing`, `providerCalculator` (Task 2).
- Produces: `toPricingUsage(u llmusage.Usage) Usage`,
  `applyFees(calc providerCalculator, usage Usage, body, requestBody []byte) Usage`, and the five
  calculator types.

- [ ] **Step 1: Establish which fields each calculator sets that a template cannot**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/gateway-controllers/policies/llm-cost
grep -n "return Usage{" -A 25 calculator_openai.go
grep -n "WebSearchRequests\|SearchContextSize\|GeminiWebSearchRequests\|AudioInputSeconds\|ToolUsePromptTokens\|Speed\|InferenceGeo\|ImageOutputTokens\|CachedAudioInputTokens" calculator_*.go
```

Record the full list. Every field on the right of the design spec's section 4 boundary must be
accounted for: web search (openai), grounding count (gemini), audio seconds (mistral), `speed` and
`inference_geo` (anthropic), per-modality tokens (gemini), and Bedrock's per-TTL `cacheDetails[]`.

- [ ] **Step 2: Write the failing test**

Create `fees_test.go`:

```go
package llmcostv2

import (
	"testing"

	"github.com/wso2/api-platform/sdk/ai/llmusage"
)

func TestToPricingUsage_MapsEveryExtractedField(t *testing.T) {
	extracted := llmusage.Usage{
		TotalInputTokens:    1000,
		UncachedInputTokens: 200,
		CachedReadTokens:    800,
		CacheWriteTokens:    50,
		CacheWrite1hTokens:  25,
		OutputTokens:        300,
		ReasoningTokens:     40,
		AudioInputTokens:    10,
		AudioOutputTokens:   5,
		TotalTokens:         1300,
		IsPriority:          true,
		Model:               "gpt-4o",
	}

	got := toPricingUsage(extracted)

	// PromptTokens must be the FULL input total: genericCalculateCost subtracts
	// the cached, cache-write and audio categories itself.
	if got.PromptTokens != 1000 {
		t.Errorf("PromptTokens = %d, want 1000", got.PromptTokens)
	}
	if got.CompletionTokens != 300 {
		t.Errorf("CompletionTokens = %d, want 300", got.CompletionTokens)
	}
	if got.TotalTokens != 1300 {
		t.Errorf("TotalTokens = %d, want 1300", got.TotalTokens)
	}
	if got.CachedReadTokens != 800 {
		t.Errorf("CachedReadTokens = %d, want 800", got.CachedReadTokens)
	}
	if got.CacheWriteTokens != 50 {
		t.Errorf("CacheWriteTokens = %d, want 50", got.CacheWriteTokens)
	}
	if got.CacheWrite1hrTokens != 25 {
		t.Errorf("CacheWrite1hrTokens = %d, want 25", got.CacheWrite1hrTokens)
	}
	if got.ReasoningTokens != 40 {
		t.Errorf("ReasoningTokens = %d, want 40", got.ReasoningTokens)
	}
	if got.AudioInputTokens != 10 {
		t.Errorf("AudioInputTokens = %d, want 10", got.AudioInputTokens)
	}
	if got.AudioOutputTokens != 5 {
		t.Errorf("AudioOutputTokens = %d, want 5", got.AudioOutputTokens)
	}
	if got.ServiceTier != "priority" {
		t.Errorf("ServiceTier = %q, want priority", got.ServiceTier)
	}
	if got.InputTokensForTiering != 1000 {
		t.Errorf("InputTokensForTiering = %d, want 1000", got.InputTokensForTiering)
	}
}

func TestToPricingUsage_NonPriorityTierIsEmpty(t *testing.T) {
	got := toPricingUsage(llmusage.Usage{TotalInputTokens: 10, IsPriority: false})

	if got.ServiceTier != "" {
		t.Errorf("ServiceTier = %q, want empty for a non-priority tier", got.ServiceTier)
	}
}

func TestApplyFees_OpenAIWebSearchDetected(t *testing.T) {
	body := []byte(`{"choices":[{"message":{"annotations":[{"type":"url_citation"}]}}]}`)
	requestBody := []byte(`{"web_search_options":{"search_context_size":"high"}}`)

	got := applyFees(&OpenAICalculator{}, Usage{PromptTokens: 10}, body, requestBody)

	if got.WebSearchRequests != 1 {
		t.Errorf("WebSearchRequests = %d, want 1", got.WebSearchRequests)
	}
	if got.SearchContextSize != "high" {
		t.Errorf("SearchContextSize = %q, want high", got.SearchContextSize)
	}
}

func TestApplyFees_OpenAINoSearchLeavesZero(t *testing.T) {
	got := applyFees(&OpenAICalculator{}, Usage{PromptTokens: 10},
		[]byte(`{"choices":[{"message":{}}]}`), nil)

	if got.WebSearchRequests != 0 {
		t.Errorf("WebSearchRequests = %d, want 0", got.WebSearchRequests)
	}
}

func TestApplyFees_PreservesExtractedCounts(t *testing.T) {
	// Fee detection must never disturb the token counts the template produced.
	body := []byte(`{"choices":[{"message":{"annotations":[{"type":"url_citation"}]}}]}`)
	in := Usage{PromptTokens: 1000, CompletionTokens: 300, CachedReadTokens: 800}

	got := applyFees(&OpenAICalculator{}, in, body, nil)

	if got.PromptTokens != 1000 || got.CompletionTokens != 300 || got.CachedReadTokens != 800 {
		t.Errorf("token counts altered by fee detection: %+v", got)
	}
}
```

- [ ] **Step 3: Run it to verify it fails**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/gateway/dev-policies/llm-cost-v2
go test ./... 2>&1 | head -10
```

Expected: a compile failure naming `toPricingUsage`, `applyFees` and `OpenAICalculator`.

- [ ] **Step 4: Copy the calculators and reduce them to fee logic**

```bash
SRC=/Users/irashperera/Documents/APIM/api-platform/dev/gateway-controllers/policies/llm-cost
DST=/Users/irashperera/Documents/APIM/api-platform/dev/api-platform/gateway/dev-policies/llm-cost-v2
cp "$SRC"/calculator_openai.go "$SRC"/calculator_anthropic.go "$SRC"/calculator_gemini.go \
   "$SRC"/calculator_mistral.go "$SRC"/calculator_bedrock.go "$SRC"/bedrock_eventstream.go "$DST"/
```

Change `package llmcost` to `package llmcostv2` in each. Then in each calculator, **keep**:

- the calculator type declaration
- `Adjust(...)` exactly as it is
- every helper computing a fee or non-path value: web-search detection from
  `choices[].message.annotations[]`, `search_context_size` from the request body, Gemini's
  grounding-query count and per-modality token walk, Mistral's `prompt_audio_seconds`, Anthropic's
  `speed` / `inference_geo`, Bedrock's `bedrockCacheWritesByTTL` and its event-stream decoding

and **delete from `Normalize()`** only the scalar token reads the template now supplies
(`prompt_tokens`, `completion_tokens`, `total_tokens`, `cached_tokens`, `reasoning_tokens`, the
scalar audio fields, `service_tier`, `model`).

Rename each surviving `Normalize` to `fees(body, requestBody []byte) Usage`, returning a `Usage`
carrying **only** fee and non-path fields. A calculator with nothing left keeps its type and
`Adjust` and gets a `fees` returning an empty `Usage`.

- [ ] **Step 5: Write the bridge**

Create `fees.go`:

```go
package llmcostv2

import "github.com/wso2/api-platform/sdk/ai/llmusage"

// priorityServiceTier is the tier value that selects priority rates.
const priorityServiceTier = "priority"

// toPricingUsage maps template-extracted usage onto the struct the pricing
// arithmetic consumes. PromptTokens carries the full input total because
// genericCalculateCost subtracts the cached, cache-write and audio categories.
func toPricingUsage(u llmusage.Usage) Usage {
	usage := Usage{
		PromptTokens:          u.TotalInputTokens,
		CompletionTokens:      u.OutputTokens,
		TotalTokens:           u.TotalTokens,
		InputTokensForTiering: u.TotalInputTokens,
		CachedReadTokens:      u.CachedReadTokens,
		CacheWriteTokens:      u.CacheWriteTokens,
		CacheWrite1hrTokens:   u.CacheWrite1hTokens,
		ReasoningTokens:       u.ReasoningTokens,
		AudioInputTokens:      u.AudioInputTokens,
		AudioOutputTokens:     u.AudioOutputTokens,
	}
	if u.IsPriority {
		usage.ServiceTier = priorityServiceTier
	}
	return usage
}

// applyFees adds the provider-specific charges that cannot be expressed as a
// field path, leaving every token count the template produced untouched.
func applyFees(calc providerCalculator, usage Usage, body, requestBody []byte) Usage {
	feeCarrier, ok := calc.(interface {
		fees(body, requestBody []byte) Usage
	})
	if !ok {
		return usage
	}

	extra := feeCarrier.fees(body, requestBody)

	usage.WebSearchRequests = extra.WebSearchRequests
	usage.SearchContextSize = extra.SearchContextSize
	usage.GeminiWebSearchRequests = extra.GeminiWebSearchRequests
	usage.ToolUsePromptTokens = extra.ToolUsePromptTokens
	usage.AudioInputSeconds = extra.AudioInputSeconds
	usage.ImageOutputTokens = extra.ImageOutputTokens
	usage.CachedAudioInputTokens = extra.CachedAudioInputTokens
	usage.InferenceGeo = extra.InferenceGeo
	usage.Speed = extra.Speed

	return usage
}
```

Cross-check against Task 2 Step 4's recorded `Usage` field list. Any fee field present there but
missing above must be added, or its charge silently disappears.

- [ ] **Step 6: Run the tests**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/gateway/dev-policies/llm-cost-v2
go clean -testcache && go build ./... && go test ./... -v 2>&1 | tail -20
```

Expected: `ok`, with the 5 fee tests passing.

- [ ] **Step 7: Leave uncommitted and report**

Do not commit or stage. Confirm the policies repo is still clean.

---

### Task 5: Fill in the provider templates and prove costs match

Combined because each template's correctness is only demonstrated by the parity test for that
provider — splitting them would leave the templates unverified.

**Files:**
- Modify: `gateway/gateway-controller/default-llm-provider-templates/{openai,anthropic,gemini,mistral,awsbedrock}-template.yaml`
- Create: `gateway/dev-policies/llm-cost-v2/llm_cost_v2_test.go`

**Interfaces:**
- Consumes: everything from Tasks 1-4.
- Produces: nothing consumed later.

- [ ] **Step 1: Add the new fields to each template**

These edits are **invisible to `llm-cost`**, which uses hardcoded paths and ignores template
fields — so this is additive and cannot affect the existing policy.

Append to each template's `spec` block, at the same indentation as the existing `promptTokens`.

`openai-template.yaml`:

```yaml
  cachedTokens:
    location: payload
    identifier: $.usage.prompt_tokens_details.cached_tokens
  reasoningTokens:
    location: payload
    identifier: $.usage.completion_tokens_details.reasoning_tokens
  audioInputTokens:
    location: payload
    identifier: $.usage.prompt_tokens_details.audio_tokens
  audioOutputTokens:
    location: payload
    identifier: $.usage.completion_tokens_details.audio_tokens
  serviceTier:
    location: payload
    identifier: $.service_tier
  cacheAccounting: inclusive
```

`anthropic-template.yaml` — note the tier lives **inside** `usage`, and accounting is **additive**
because Anthropic's `input_tokens` excludes cached and cache-creation tokens:

```yaml
  cachedTokens:
    location: payload
    identifier: $.usage.cache_read_input_tokens
  cacheWriteTokens:
    location: payload
    identifier: $.usage.cache_creation.ephemeral_5m_input_tokens
  cacheWrite1hTokens:
    location: payload
    identifier: $.usage.cache_creation.ephemeral_1h_input_tokens
  serviceTier:
    location: payload
    identifier: $.usage.service_tier
  cacheAccounting: additive
```

`gemini-template.yaml` — no audio or image fields, because Gemini reports those inside per-modality
arrays a path cannot select; no `serviceTier`, because Gemini has none:

```yaml
  cachedTokens:
    location: payload
    identifier: $.usageMetadata.cachedContentTokenCount
  reasoningTokens:
    location: payload
    identifier: $.usageMetadata.thoughtsTokenCount
  cacheAccounting: inclusive
```

`mistral-template.yaml` — OpenAI-shaped with no caching:

```yaml
  cacheAccounting: inclusive
```

`awsbedrock-template.yaml` — the tier is at the **response top level**, not under `usage`:

```yaml
  cachedTokens:
    location: payload
    identifier: $.usage.cacheReadInputTokens
  cacheWriteTokens:
    location: payload
    identifier: $.usage.cacheWriteInputTokens
  serviceTier:
    location: payload
    identifier: $.serviceTier.type
  cacheAccounting: additive
```

- [ ] **Step 2: Verify the templates still parse and validate**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/gateway/gateway-controller
go clean -testcache && go test ./pkg/config/ ./pkg/utils/ 2>&1 | grep -E "^(ok|FAIL|---)"
```

Expected: `ok` for both. `pkg/utils` loads the shipped templates, so a YAML or schema mistake
surfaces here.

- [ ] **Step 3: Confirm `llm-cost` is unaffected**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/gateway-controllers/policies/llm-cost
go clean -testcache && go test ./... 2>&1 | grep -E "^(ok|FAIL)"
```

Expected: `ok`. The original reads no templates, so this must be unchanged.

- [ ] **Step 4: Write the parity harness and the OpenAI cases**

Create `llm_cost_v2_test.go`:

```go
package llmcostv2

import (
	"context"
	"testing"

	"github.com/wso2/api-platform/sdk/ai/llmusage"
	policy "github.com/wso2/api-platform/sdk/core/policy/v1alpha2"
)

// storeTemplate seeds the lazy-resource store the extraction library reads and
// removes the entry when the test ends.
func storeTemplate(t *testing.T, handle string, spec map[string]interface{}) {
	t.Helper()

	store := policy.GetLazyResourceStoreInstance()
	if err := store.StoreResource(&policy.LazyResource{
		ID:           handle,
		ResourceType: llmusage.ResourceTypeLLMProviderTemplate,
		Resource:     spec,
	}); err != nil {
		t.Fatalf("failed to store template: %v", err)
	}
	t.Cleanup(func() {
		_ = store.RemoveResourceByIDAndType(handle, llmusage.ResourceTypeLLMProviderTemplate)
	})
}

func ident(path string) map[string]interface{} {
	return map[string]interface{}{"location": "payload", "identifier": path}
}

// openAISpec mirrors the shipped openai-template.yaml after Step 1.
func openAISpec() map[string]interface{} {
	return map[string]interface{}{
		"cacheAccounting":   "inclusive",
		"promptTokens":      ident("$.usage.prompt_tokens"),
		"completionTokens":  ident("$.usage.completion_tokens"),
		"totalTokens":       ident("$.usage.total_tokens"),
		"cachedTokens":      ident("$.usage.prompt_tokens_details.cached_tokens"),
		"reasoningTokens":   ident("$.usage.completion_tokens_details.reasoning_tokens"),
		"audioInputTokens":  ident("$.usage.prompt_tokens_details.audio_tokens"),
		"audioOutputTokens": ident("$.usage.completion_tokens_details.audio_tokens"),
		"serviceTier":       ident("$.service_tier"),
		"requestModel":      ident("$.model"),
		"responseModel":     ident("$.model"),
	}
}

func newTestPolicy(t *testing.T) *LLMCostV2Policy {
	t.Helper()

	p, err := GetPolicy(policy.PolicyMetadata{}, map[string]interface{}{
		"pricing_file": "testdata/model_prices.json",
	})
	if err != nil {
		t.Fatalf("GetPolicy failed: %v", err)
	}
	return p.(*LLMCostV2Policy)
}

// runResponse drives one buffered response through the policy and returns the
// published cost string and status.
func runResponse(t *testing.T, p *LLMCostV2Policy, handle string, body, requestBody []byte, path string) (string, string) {
	t.Helper()

	sc := &policy.SharedContext{Metadata: map[string]interface{}{
		llmusage.MetadataTemplateHandle: handle,
	}}
	respCtx := &policy.ResponseStreamContext{SharedContext: sc, RequestPath: path}
	if requestBody != nil {
		respCtx.RequestBody = &policy.Body{Content: requestBody, Present: true}
	}

	p.OnResponseBodyChunk(context.Background(), respCtx,
		&policy.StreamBody{Chunk: body, EndOfStream: true, Index: 0}, nil)

	cost, _ := sc.Metadata[MetadataLLMCost].(string)
	status, _ := sc.Metadata[MetadataLLMCostStatus].(string)
	return cost, status
}

func TestParity_OpenAIBuffered(t *testing.T) {
	storeTemplate(t, "openai", openAISpec())
	p := newTestPolicy(t)

	body := []byte(`{"model":"gpt-4o-mini-2024-07-18","usage":{"prompt_tokens":1000,"completion_tokens":200,"total_tokens":1200}}`)

	cost, status := runResponse(t, p, "openai", body, nil, "/chat/completions")

	if status != costStatusCalculated {
		t.Fatalf("status = %q, want %q", status, costStatusCalculated)
	}
	if cost == "" || cost == "0.0000000000" {
		t.Fatalf("cost = %q, want a non-zero calculated cost", cost)
	}
	t.Logf("openai buffered cost = %s", cost)
}

func TestParity_OpenAICachedTokensDiscounted(t *testing.T) {
	storeTemplate(t, "openai", openAISpec())
	p := newTestPolicy(t)

	plain := []byte(`{"model":"gpt-4o-mini-2024-07-18","usage":{"prompt_tokens":1000,"completion_tokens":100}}`)
	cached := []byte(`{"model":"gpt-4o-mini-2024-07-18","usage":{"prompt_tokens":1000,"completion_tokens":100,"prompt_tokens_details":{"cached_tokens":800}}}`)

	plainCost, _ := runResponse(t, p, "openai", plain, nil, "/chat/completions")
	cachedCost, _ := runResponse(t, p, "openai", cached, nil, "/chat/completions")

	if plainCost == cachedCost {
		t.Errorf("cached and uncached cost are identical (%s); the cache discount was not applied", plainCost)
	}
	t.Logf("uncached=%s cached=%s", plainCost, cachedCost)
}

func TestParity_UnknownModelIsUnpriced(t *testing.T) {
	storeTemplate(t, "openai", openAISpec())
	p := newTestPolicy(t)

	body := []byte(`{"model":"not-a-real-model-xyz","usage":{"prompt_tokens":10,"completion_tokens":5}}`)

	cost, status := runResponse(t, p, "openai", body, nil, "/chat/completions")

	if status != costStatusNotCalculated {
		t.Errorf("status = %q, want %q", status, costStatusNotCalculated)
	}
	if cost != "0.0000000000" {
		t.Errorf("cost = %q, want 0.0000000000", cost)
	}
}

func TestParity_NoTemplateIsUnpricedNotFatal(t *testing.T) {
	p := newTestPolicy(t)

	sc := &policy.SharedContext{Metadata: map[string]interface{}{}}
	respCtx := &policy.ResponseStreamContext{SharedContext: sc, RequestPath: "/chat/completions"}
	action := p.OnResponseBodyChunk(context.Background(), respCtx,
		&policy.StreamBody{
			Chunk:       []byte(`{"model":"gpt-4o-mini-2024-07-18","usage":{"prompt_tokens":10}}`),
			EndOfStream: true, Index: 0,
		}, nil)

	if action == nil {
		t.Fatal("action is nil; the response must still be forwarded")
	}
	if got, _ := sc.Metadata[MetadataLLMCostStatus].(string); got != costStatusNotCalculated {
		t.Errorf("status = %q, want %q", got, costStatusNotCalculated)
	}
}
```

- [ ] **Step 5: Add one parity case per remaining provider**

For anthropic, gemini, mistral and bedrock, add a `*Spec()` helper mirroring that provider's
template from Step 1, plus a test using a response body taken from `llm-cost`'s own tests.

**How to get each expected cost:** run the same body through `llm-cost` and record what it
produces. Do not compute a figure by hand — a hand-derived number proves nothing about parity.

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/gateway-controllers/policies/llm-cost
grep -n "assertCostMetadata" llm_cost_test.go | head -20
```

- [ ] **Step 6: Test the highest-risk divergence explicitly**

`llm-cost`'s anthropic calculator derives the input total from
`input_tokens + cache_creation_input_tokens + cache_read_input_tokens` — the **aggregate** field.
Step 1's template instead declares the **per-TTL breakdown**, which is correct when present, since
the aggregate and the breakdown describe the same tokens and declaring both would double-count.

The risk is a response carrying `cache_creation_input_tokens` with **no** `cache_creation` object:
the template then extracts zero cache-write tokens, `PromptTokens` is short by that amount, and the
cost differs. Add:

```go
func TestParity_AnthropicAggregateCacheCreationOnly(t *testing.T) {
	storeTemplate(t, "anthropic", anthropicSpec())
	p := newTestPolicy(t)

	// Aggregate present, per-TTL breakdown absent.
	body := []byte(`{"model":"claude-sonnet-4-5","usage":{` +
		`"input_tokens":1000,"output_tokens":200,` +
		`"cache_read_input_tokens":500,"cache_creation_input_tokens":300}}`)

	cost, status := runResponse(t, p, "anthropic", body, nil, "/v1/messages")

	t.Logf("aggregate-only cache creation: status=%s cost=%s", status, cost)
	// Compare against llm-cost for this same body. If they differ, the fix is in
	// the template, not the code.
}
```

If it diverges, point `cacheWriteTokens` at `$.usage.cache_creation_input_tokens` instead and
record why.

- [ ] **Step 7: Record the two other expected divergences**

Both should leave cost unchanged with the shipped pricing file. Verify rather than assume:

1. **Anthropic and Bedrock `serviceTier`** — now extracted, previously ignored. No anthropic or
   bedrock pricing entry carries a priority rate, so cost should not move.
2. **OpenAI `cache_write_tokens`** — deliberately **not** declared in Step 1, because `llm-cost`
   does not read it and the shipped pricing file has no cache-write rate for the GPT-5.6 family.
   Adding it is a separate billing decision (design spec open question 4).

Any other difference is a defect. Report it rather than adjusting the expectation.

- [ ] **Step 8: Run everything**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/gateway/dev-policies/llm-cost-v2
go clean -testcache && go test ./... -v 2>&1 | tail -30
```

Expected: all pass.

- [ ] **Step 9: Leave uncommitted and report**

Do not commit or stage. Report the parity results per provider, any divergence found, and confirm
the policies repo is clean.

---

### Task 6: Build the gateway and confirm the policy loads

**Files:** none created. This task verifies.

- [ ] **Step 1: Confirm the required config block, and do not add it yourself**

The system parameter references `config.policy_configurations.llm_cost_v2.pricing_file`. Without
it the parameter resolves empty and `GetPolicy` fails at startup.

`gateway/configs/config.toml` is one of the owner's uncommitted files, so **do not edit it**.
Report that this block is needed:

```toml
[policy_configurations.llm_cost_v2]
pricing_file = "/etc/policy-engine/llm-pricing/model_prices.json"
```

- [ ] **Step 2: Build the gateway**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/gateway
make build 2>&1 | tail -40
```

Report the outcome. If it fails, capture the full error and which stage failed — do not improvise a
fix.

- [ ] **Step 3: Confirm the policy is in the build manifest**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/gateway
grep -n "llm-cost-v2" build-manifest.yaml || echo "  NOT in the manifest"
```

`build-manifest.yaml` is written by the builder. Note that it is one of the owner's modified files,
so report the change rather than treating it as yours.

- [ ] **Step 4: Final state check**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform
git status --short
git -C ../gateway-controllers status --short
```

Report both lists in full, and restate the three temporary entries that must be removed before any
commit.

---

## Definition of done

- [ ] `gateway/dev-policies/llm-cost-v2` builds and its full suite passes under
      `go clean -testcache`.
- [ ] It imports `sdk/ai/llmusage` directly — no copied library files anywhere.
- [ ] Parity tests cover all five providers, with every divergence explicitly tested and explained.
- [ ] `policies/llm-cost/` shows **no** modification, and `llm-cost`'s own suite still passes.
- [ ] All five provider templates carry the new fields and still parse and validate.
- [ ] `gateway/gateway-controller` `pkg/config` and `pkg/utils` suites pass.
- [ ] The gateway build outcome is reported, whatever it is.
- [ ] Nothing committed, nothing staged, in either repository.

## Follow-up work this plan does not do

- **Shipping the policy.** It lives in `dev-policies/`, so it is not in the policy catalog and is
  not a product policy. Making it one means moving it to
  `gateway-controllers/policies/llm-cost-v2` and depending on a published `sdk/ai/llmusage`.
- **Removing the three temporary entries.** They must come out before any commit;
  `dev-policies/README.md` says the same about the `build.yaml` entry.
- **Retiring `llm-cost`.** Out of scope. Whether `llm-cost-v2` replaces it is a decision for after
  real-traffic comparison.
- **The `platform-api` pruning gap** (design spec section 11) — templates mirrored to the control
  plane still lose the new fields.
- **The `cache_write_tokens` billing decision** — design spec open question 4.
- **`azure-llm-cost`** — untouched.
