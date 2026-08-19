# `aiCost` Analytics Breakdown — Implementation Plan (Plan 6)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Publish a per-category LLM cost breakdown to analytics as a new `aiCost` property, keyed by the provider's own response field names and nested to mirror the provider's response body.

**Architecture:** `genericCalculateCost` is split into a component struct plus a `total()` that reproduces today's sum exactly, so the breakdown is a by-product of pricing rather than a second calculation. `llmusage` gains a public accessor for the template's declared response paths. The policy joins the two — component value plus declared path — into a nested map, and the policy-engine analytics pipeline emits it alongside `aiMetadata` and `aiTokenUsage`.

**Tech Stack:** Go, `sdk/ai/llmusage`, `dev-policies/llm-cost-v2`, `gateway-runtime/policy-engine`.

## Global Constraints

- **Never run `git commit`, `git add`, or any committing command.** No task ends in a commit. Applies to subagents equally. If you think a commit is needed, stop and ask.
- **Never run `git checkout <file>` or `git restore <file>`.** Back up with `cp` before editing anything you may need to revert.
- **`gateway-controllers/policies/llm-cost` (v1) must not be modified.** It is the parity reference.
- **Never modify these owner-held files:** `gateway/build-manifest.yaml`, `gateway/configs/config.toml`, `gateway/configs/config-template.toml`, `gateway/configs/llm-pricing/model_prices.json`. Read freely.
- **Comments must never describe a change, fix, or history.** Explain what the code does for a reader who never saw the old version.
- **All Go verification runs with `GOWORK=off`.**
- **Run `go clean -testcache` before asserting tests pass.**
- **Provider facts come only from official published specs.** Never infer a provider's field names from gateway code or the pricing file.
- **Do not name or allude to any third-party pricing-data source** anywhere, including comments.
- **Parity baselines that must be unchanged at the end:** `0.0002700000`, `0.0061500000`, `0.0072750000`. Also unchanged: `0.0065000000` / `0.0117000000` (Gemini standard/priority) and `0.0530000000` / `0.0954000000`.

## Design decisions already settled

1. **Costs sit at the root of `aiCost`; nesting appears only where the provider nests.** The tree is built by splitting each template `identifier` on `.` after stripping the leading `$.`, then **dropping the first segment** — the response's usage container (`usage`, `usageMetadata`), which carries no information once inside `aiCost`. Remaining segments nest. A single-segment path keeps its position at the root. This still removes key collisions without a collision rule: OpenAI's two `audio_tokens` fields separate under `prompt_tokens_details` and `completion_tokens_details` because the provider itself separates them.

1a. **`aiCost` contains no total and no reconciliation field.** It is the breakdown only. The request's total remains `llmCost`, computed by the policy exactly as today including each calculator's `Adjust`. Nothing in this plan changes how the total is produced or reported; Task 2's refactor is arithmetically identical and is diff-verified against a baseline recorded beforehand.
2. **Key naming.** Take the path's leaf; if it ends in `Count`, replace that with `Cost`; otherwise append `_cost` when the leaf contains `_`, else `Cost`.
3. **Template-derivable only.** A cost component is emitted only when a template field declares where its token count lives. Components with no template field behind them are **ignored**: image-output cost, audio-seconds cost, web-search cost, tool-use cost, and any Bedrock 5m/1h split that the template does not declare separately.
4. **Consequence, accepted:** `aiCost` will not sum to `llmCost` when an ignored component is non-zero. This is inherent to decision 3, not a defect.
5. **Service tier is included**, at the top level of `aiCost` rather than inside the nested tree, because it describes the whole pricing decision rather than one token category. The **normalized** tier is emitted (the value that actually selected the rate table), with the standard tier rendered as the string `standard` rather than an empty string.

## Expected output

Verified against the shipped templates; every provider below produced zero key collisions.

```jsonc
// Gemini — $.usageMetadata.* flattens to the root
"aiCost": {
  "promptTokenCost": 0.00005,
  "candidatesTokenCost": 0.00012,
  "cachedContentTokenCost": 0.000001,
  "thoughtsTokenCost": 0.00002,
  "serviceTier": "priority"
}

// OpenAI — the two audio fields stay separate under their own parents
"aiCost": {
  "prompt_tokens_cost": 0.00005,
  "completion_tokens_cost": 0.00012,
  "prompt_tokens_details":     { "cached_tokens_cost": 0.0, "audio_tokens_cost": 0.0 },
  "completion_tokens_details": { "reasoning_tokens_cost": 0.0, "audio_tokens_cost": 0.0 },
  "serviceTier": "standard"
}

// Anthropic — both declared cache TTLs reported
"aiCost": {
  "input_tokens_cost": 0.00005,
  "output_tokens_cost": 0.00012,
  "cache_read_input_tokens_cost": 0.000001,
  "cache_creation": {
    "ephemeral_5m_input_tokens_cost": 0.0,
    "ephemeral_1h_input_tokens_cost": 0.0
  },
  "serviceTier": "standard"
}

// Bedrock — flat, camelCase preserved
"aiCost": {
  "inputTokensCost": 0.00005,
  "outputTokensCost": 0.00012,
  "cacheReadInputTokensCost": 0.0,
  "cacheWriteInputTokensCost": 0.0,
  "serviceTier": "standard"
}
```

## File Structure

| File | Responsibility |
|---|---|
| `sdk/ai/llmusage/template.go` | new exported `FieldPaths` accessor |
| `dev-policies/llm-cost-v2/pricing.go` | `costComponents` struct; `genericCalculateCost` delegates to it |
| `dev-policies/llm-cost-v2/aicost.go` | **new** — builds the nested tree from paths + components |
| `dev-policies/llm-cost-v2/llm_cost_v2.go` | attach `aiCost` to analytics metadata on both response paths |
| `policy-engine/internal/constants/constants.go` | `AICostMetadataKey` |
| `policy-engine/internal/analytics/analytics.go` | emit `aiCost` into `event.Properties` |

---

### Task 1: `llmusage` exposes the declared response paths

The policy needs each provider's own field names, which exist only in the template. `resolveFields` already resolves them per route including `resourceMappings` overrides; this exposes that result.

**Files:**
- Modify: `api-platform/sdk/ai/llmusage/template.go`
- Test: `api-platform/sdk/ai/llmusage/template_test.go`

**Interfaces:**
- Produces: `func FieldPaths(sc *policy.SharedContext, requestPath string) map[string]string` — template field name → declared identifier, restricted to `payload` fields. Empty map when the route has no template. Consumed by Task 3.

- [ ] **Step 1: Write the failing test**

Append to `template_test.go`:

```go
func TestFieldPaths_ReturnsPayloadIdentifiers(t *testing.T) {
	sc := &policy.SharedContext{Metadata: map[string]interface{}{}}
	storeTemplateForTest(t, sc, buildTemplate())

	paths := FieldPaths(sc, "/chat/completions")

	if paths["promptTokens"] != "$.usage.input_tokens" {
		t.Errorf("promptTokens = %q, want $.usage.input_tokens", paths["promptTokens"])
	}
	if paths["completionTokens"] != "$.usage.output_tokens" {
		t.Errorf("completionTokens = %q, want $.usage.output_tokens", paths["completionTokens"])
	}
}

func TestFieldPaths_HonoursResourceMappings(t *testing.T) {
	sc := &policy.SharedContext{Metadata: map[string]interface{}{}}
	storeTemplateForTest(t, sc, buildTemplate())

	paths := FieldPaths(sc, "/responses")

	if paths["promptTokens"] != "$.usage.prompt_tokens" {
		t.Errorf("promptTokens = %q, want the /responses override $.usage.prompt_tokens",
			paths["promptTokens"])
	}
}

func TestFieldPaths_NoTemplateReturnsEmpty(t *testing.T) {
	sc := &policy.SharedContext{Metadata: map[string]interface{}{}}

	if got := FieldPaths(sc, "/chat/completions"); len(got) != 0 {
		t.Errorf("FieldPaths = %v, want empty", got)
	}
}

func TestFieldPaths_NilContextReturnsEmpty(t *testing.T) {
	if got := FieldPaths(nil, "/chat/completions"); len(got) != 0 {
		t.Errorf("FieldPaths = %v, want empty", got)
	}
}
```

`storeTemplateForTest` does not exist yet. Add it to `template_test.go` too, modelled on how `extract_test.go` puts a template in the store — check that file first for the exact store API:

```bash
cd api-platform/sdk/ai/llmusage
grep -n -B3 -A14 "GetLazyResourceStoreInstance" extract_test.go | head -30
```

Then write the helper to match, storing under a handle and setting `sc.Metadata[MetadataTemplateHandle]`.

- [ ] **Step 2: Run to verify it fails**

```bash
cd api-platform/sdk/ai/llmusage
GOWORK=off go clean -testcache
GOWORK=off go test -run TestFieldPaths -v ./... 2>&1 | head -20
```

Expected: compile failure — `undefined: FieldPaths`.

- [ ] **Step 3: Implement**

Append to `template.go`:

```go
// FieldPaths returns the response path each usage field is declared at for this
// route, so a consumer can label per-field output with the provider's own field
// names. Only payload fields are returned; header and path-parameter
// identifiers do not name a location in the response body. The result reflects
// any resourceMappings override that applies to requestPath.
func FieldPaths(sc *policy.SharedContext, requestPath string) map[string]string {
	if sc == nil {
		return map[string]string{}
	}

	template, err := templateForRoute(sc)
	if err != nil {
		return map[string]string{}
	}

	fields, _ := resolveFields(template, requestPath)
	paths := make(map[string]string, len(fields))
	for name, spec := range fields {
		if spec.Location == locationPayload && spec.Identifier != "" {
			paths[name] = spec.Identifier
		}
	}
	return paths
}
```

Confirm the payload location constant's name before using it:

```bash
cd api-platform/sdk/ai/llmusage
grep -rn "locationPayload\s*=" *.go
```

- [ ] **Step 4: Run to verify it passes**

```bash
cd api-platform/sdk/ai/llmusage
GOWORK=off go clean -testcache
GOWORK=off go test -run TestFieldPaths -v ./... 2>&1 | grep -E "^(--- |ok|FAIL)"
```

Expected: all four PASS.

- [ ] **Step 5: Whole library still green**

```bash
cd api-platform/sdk/ai/llmusage
GOWORK=off go test ./... 2>&1 | tail -3
GOWORK=off go vet ./...
```

Expected: `ok`, vet silent.

---

### Task 2: decompose the cost calculation

The breakdown must be a by-product of the existing arithmetic, not a parallel implementation, or the two will drift. `genericCalculateCost` keeps its signature and returns `components.total()`, so any change to the total is a test failure.

**Files:**
- Modify: `api-platform/gateway/dev-policies/llm-cost-v2/pricing.go:475-575`
- Test: `api-platform/gateway/dev-policies/llm-cost-v2/pricing_test.go`

**Interfaces:**
- Produces: `type costComponents struct` with fields `Prompt, Completion, CacheRead, CacheWrite5m, CacheWrite1h, Reasoning, AudioInput, AudioOutput, ImageOutput, AudioSeconds, WebSearch, ToolUse float64`; `func (c costComponents) total() float64`; `func calculateCostComponents(usage Usage, pricing ModelPricing) costComponents`. Consumed by Task 3.
- Note: today's single `cacheWriteCost` is **split** into `CacheWrite5m` and `CacheWrite1h` so Anthropic's two declared TTL fields can be reported separately. Their sum must equal the old combined value.

- [ ] **Step 1: Write the failing test**

Append to `pricing_test.go` (check the file exists first; if not, create it with the same package clause as `pricing.go`):

```go
func TestCalculateCostComponents_TotalMatchesGenericCalculateCost(t *testing.T) {
	usage := Usage{
		PromptTokens:        10000,
		CompletionTokens:    2000,
		CachedReadTokens:    3000,
		CacheWriteTokens:    500,
		CacheWrite1hrTokens: 250,
		ReasoningTokens:     400,
		AudioInputTokens:    100,
		AudioOutputTokens:   50,
		ImageOutputTokens:   20,
	}
	pricing := ModelPricing{
		InputCostPerToken:                 1e-06,
		OutputCostPerToken:                2e-06,
		CacheReadInputTokenCost:           5e-07,
		CacheCreationInputTokenCost:       1.2e-06,
		CacheCreationInputTokenCostAbove1hr: 2e-06,
		OutputCostPerReasoningToken:       3e-06,
		InputCostPerAudioToken:            4e-06,
		OutputCostPerAudioToken:           5e-06,
		OutputCostPerImageToken:           6e-06,
	}

	components := calculateCostComponents(usage, pricing)
	total := genericCalculateCost(usage, pricing)

	if diff := components.total() - total; diff > 1e-12 || diff < -1e-12 {
		t.Errorf("components.total() = %.12f, genericCalculateCost = %.12f", components.total(), total)
	}
}

func TestCalculateCostComponents_SplitsCacheWriteByTTL(t *testing.T) {
	usage := Usage{CacheWriteTokens: 1000, CacheWrite1hrTokens: 2000}
	pricing := ModelPricing{
		InputCostPerToken:                   1e-06,
		CacheCreationInputTokenCost:         1e-06,
		CacheCreationInputTokenCostAbove1hr: 3e-06,
	}

	c := calculateCostComponents(usage, pricing)

	if c.CacheWrite5m <= 0 || c.CacheWrite1h <= 0 {
		t.Fatalf("both TTL components must be populated, got 5m=%v 1h=%v", c.CacheWrite5m, c.CacheWrite1h)
	}
	if c.CacheWrite1h <= c.CacheWrite5m {
		t.Errorf("1h component %v should exceed 5m component %v at these rates", c.CacheWrite1h, c.CacheWrite5m)
	}
}
```

Before running, confirm the exact `ModelPricing` field names used above — several are guesses at the 1-hour cache rate field:

```bash
cd api-platform/gateway/dev-policies/llm-cost-v2
grep -nE "CacheCreationInputTokenCost|cacheWrite1h|cacheWrite5m" pricing.go | head
grep -n -A10 "func resolveRates" pricing.go | head -20
```

Fix the literal field names in the test to match before proceeding.

- [ ] **Step 2: Run to verify it fails**

```bash
cd api-platform/gateway/dev-policies/llm-cost-v2
GOWORK=off go clean -testcache
GOWORK=off go test -run TestCalculateCostComponents -v ./... 2>&1 | head -12
```

Expected: compile failure — `undefined: calculateCostComponents`.

- [ ] **Step 3: Record the exact parity baseline before refactoring**

```bash
cd api-platform/gateway/dev-policies/llm-cost-v2
GOWORK=off go test -v ./... 2>&1 | grep -oE "0\.[0-9]{10}" | sort -u > /tmp/p6-baseline.txt
cat /tmp/p6-baseline.txt
```

- [ ] **Step 4: Extract the components struct**

In `pricing.go`, add above `genericCalculateCost`:

```go
// costComponents holds each separately-rated part of one request's cost. The
// parts are reported individually for analytics and summed for billing, so both
// come from the same arithmetic.
type costComponents struct {
	Prompt       float64
	Completion   float64
	CacheRead    float64
	CacheWrite5m float64
	CacheWrite1h float64
	Reasoning    float64
	AudioInput   float64
	AudioOutput  float64
	ImageOutput  float64
	AudioSeconds float64
	WebSearch    float64
	ToolUse      float64
}

func (c costComponents) total() float64 {
	return c.Prompt + c.Completion + c.CacheRead + c.CacheWrite5m + c.CacheWrite1h +
		c.Reasoning + c.WebSearch + c.ToolUse + c.AudioInput + c.AudioOutput +
		c.ImageOutput + c.AudioSeconds
}
```

Then rename the existing `genericCalculateCost` body into `calculateCostComponents`, changing only:

- its signature to `func calculateCostComponents(usage Usage, pricing ModelPricing) costComponents`
- the single `cacheWriteCost := …` line into two assignments:

```go
	cacheWrite5mCost := float64(usage.CacheWriteTokens) * r.cacheWrite5m
	cacheWrite1hCost := float64(usage.CacheWrite1hrTokens) * r.cacheWrite1h
```

- its final `return` into a `costComponents{…}` literal assigning every local cost variable to its field

Leave every rate lookup, exclusion subtraction and clamp exactly as it is. Then replace `genericCalculateCost` with:

```go
// genericCalculateCost prices one request using the provider-agnostic rules.
func genericCalculateCost(usage Usage, pricing ModelPricing) float64 {
	return calculateCostComponents(usage, pricing).total()
}
```

- [ ] **Step 5: Run the new tests**

```bash
cd api-platform/gateway/dev-policies/llm-cost-v2
GOWORK=off go clean -testcache
GOWORK=off go test -run TestCalculateCostComponents -v ./... 2>&1 | grep -E "^(--- |ok|FAIL)"
```

Expected: both PASS.

- [ ] **Step 6: Prove the refactor changed no cost**

```bash
cd api-platform/gateway/dev-policies/llm-cost-v2
GOWORK=off go test ./... 2>&1 | tail -3
GOWORK=off go vet ./...
GOWORK=off go test -v ./... 2>&1 | grep -oE "0\.[0-9]{10}" | sort -u > /tmp/p6-after.txt
diff /tmp/p6-baseline.txt /tmp/p6-after.txt && echo "  IDENTICAL — refactor is cost-neutral"
```

Expected: `ok`, vet silent, `diff` empty.

---

### Task 3: build the nested `aiCost` tree

**Files:**
- Create: `api-platform/gateway/dev-policies/llm-cost-v2/aicost.go`
- Test: `api-platform/gateway/dev-policies/llm-cost-v2/aicost_test.go`

**Interfaces:**
- Consumes: `FieldPaths` (Task 1), `costComponents` (Task 2).
- Produces: `func buildAICost(paths map[string]string, c costComponents, serviceTier string) map[string]interface{}`. Returns nil when nothing is attributable, so the caller can omit the property entirely.

- [ ] **Step 1: Write the failing test**

Create `aicost_test.go`:

```go
package llmcostv2

import "testing"

func TestBuildAICost_UsageContainerDroppedCostsAtRoot(t *testing.T) {
	paths := map[string]string{
		"promptTokens":     "$.usageMetadata.promptTokenCount",
		"completionTokens": "$.usageMetadata.candidatesTokenCount",
		"reasoningTokens":  "$.usageMetadata.thoughtsTokenCount",
	}
	c := costComponents{Prompt: 0.001, Completion: 0.002, Reasoning: 0.003}

	got := buildAICost(paths, c, "priority")

	if _, present := got["usageMetadata"]; present {
		t.Errorf("usageMetadata container should be dropped: %#v", got)
	}
	if got["serviceTier"] != "priority" {
		t.Errorf("serviceTier = %v, want priority", got["serviceTier"])
	}
	for key, want := range map[string]float64{
		"promptTokenCost":     0.001,
		"candidatesTokenCost": 0.002,
		"thoughtsTokenCost":   0.003,
	} {
		if got[key] != want {
			t.Errorf("%s = %v, want %v", key, got[key], want)
		}
	}
}

func TestBuildAICost_NestsOnlyWhereProviderNests(t *testing.T) {
	paths := map[string]string{
		"promptTokens":      "$.usage.prompt_tokens",
		"audioInputTokens":  "$.usage.prompt_tokens_details.audio_tokens",
		"audioOutputTokens": "$.usage.completion_tokens_details.audio_tokens",
	}
	c := costComponents{Prompt: 1.5, AudioInput: 0.5, AudioOutput: 0.25}

	got := buildAICost(paths, c, "")

	if got["prompt_tokens_cost"] != 1.5 {
		t.Errorf("prompt_tokens_cost = %v, want 1.5 at the root", got["prompt_tokens_cost"])
	}
	in, ok := got["prompt_tokens_details"].(map[string]interface{})
	if !ok {
		t.Fatalf("prompt_tokens_details missing: %#v", got)
	}
	out, ok := got["completion_tokens_details"].(map[string]interface{})
	if !ok {
		t.Fatalf("completion_tokens_details missing: %#v", got)
	}
	if in["audio_tokens_cost"] != 0.5 {
		t.Errorf("input audio = %v, want 0.5", in["audio_tokens_cost"])
	}
	if out["audio_tokens_cost"] != 0.25 {
		t.Errorf("output audio = %v, want 0.25", out["audio_tokens_cost"])
	}
}

func TestBuildAICost_StandardTierRenderedExplicitly(t *testing.T) {
	paths := map[string]string{"promptTokens": "$.usage.prompt_tokens"}
	got := buildAICost(paths, costComponents{Prompt: 1}, "")

	if got["serviceTier"] != "standard" {
		t.Errorf("serviceTier = %v, want standard", got["serviceTier"])
	}
}

func TestBuildAICost_IgnoresComponentsWithNoTemplateField(t *testing.T) {
	paths := map[string]string{"promptTokens": "$.usage.prompt_tokens"}
	c := costComponents{Prompt: 1, ImageOutput: 99, WebSearch: 99, ToolUse: 99, AudioSeconds: 99}

	got := buildAICost(paths, c, "")

	// prompt_tokens_cost plus serviceTier, and nothing derived from the
	// components that no template field locates.
	if len(got) != 2 {
		t.Errorf("aiCost = %#v, want only prompt_tokens_cost and serviceTier", got)
	}
	if got["prompt_tokens_cost"] != float64(1) {
		t.Errorf("prompt_tokens_cost = %v, want 1", got["prompt_tokens_cost"])
	}
}

func TestBuildAICost_NoTotalOrReconciliationField(t *testing.T) {
	paths := map[string]string{
		"promptTokens":     "$.usage.prompt_tokens",
		"completionTokens": "$.usage.completion_tokens",
	}
	got := buildAICost(paths, costComponents{Prompt: 1, Completion: 2}, "")

	for _, forbidden := range []string{"total", "totalCost", "total_cost", "llmCost", "sum"} {
		if _, present := got[forbidden]; present {
			t.Errorf("aiCost must carry no total; found %q in %#v", forbidden, got)
		}
	}
}

func TestBuildAICost_NoPathsReturnsNil(t *testing.T) {
	if got := buildAICost(map[string]string{}, costComponents{Prompt: 1}, "priority"); got != nil {
		t.Errorf("buildAICost = %#v, want nil", got)
	}
}

func TestBuildAICost_ZeroComponentStillReported(t *testing.T) {
	paths := map[string]string{"cachedTokens": "$.usage.prompt_tokens_details.cached_tokens"}
	got := buildAICost(paths, costComponents{}, "")

	details, ok := got["prompt_tokens_details"].(map[string]interface{})
	if !ok {
		t.Fatalf("prompt_tokens_details missing: %#v", got)
	}
	if details["cached_tokens_cost"] != float64(0) {
		t.Errorf("cached_tokens_cost = %v, want 0", details["cached_tokens_cost"])
	}
}

func TestCostKeyNaming(t *testing.T) {
	for leaf, want := range map[string]string{
		"promptTokenCount": "promptTokenCost",
		"prompt_tokens":    "prompt_tokens_cost",
		"inputTokens":      "inputTokensCost",
		"audio_tokens":     "audio_tokens_cost",
	} {
		if got := costKey(leaf); got != want {
			t.Errorf("costKey(%q) = %q, want %q", leaf, got, want)
		}
	}
}
```

- [ ] **Step 2: Run to verify it fails**

```bash
cd api-platform/gateway/dev-policies/llm-cost-v2
GOWORK=off go clean -testcache
GOWORK=off go test -run "BuildAICost|CostKeyNaming" -v ./... 2>&1 | head -12
```

Expected: compile failure — `undefined: buildAICost`, `undefined: costKey`.

- [ ] **Step 3: Implement**

Create `aicost.go`:

```go
package llmcostv2

import "strings"

// componentForField maps a template field to the cost component charged for the
// tokens that field locates. Components with no template field behind them —
// image output, audio duration, web search and tool use — are absent, so they
// are not reported.
func componentForField(name string, c costComponents) (float64, bool) {
	switch name {
	case "promptTokens":
		return c.Prompt, true
	case "completionTokens":
		return c.Completion, true
	case "cachedTokens":
		return c.CacheRead, true
	case "cacheWriteTokens":
		return c.CacheWrite5m, true
	case "cacheWrite1hTokens":
		return c.CacheWrite1h, true
	case "reasoningTokens":
		return c.Reasoning, true
	case "audioInputTokens":
		return c.AudioInput, true
	case "audioOutputTokens":
		return c.AudioOutput, true
	}
	return 0, false
}

// costKey turns the leaf of a response path into the name its cost is reported
// under, keeping the provider's own casing. A leaf naming a count becomes the
// matching cost; anything else gains a cost suffix.
func costKey(leaf string) string {
	if strings.HasSuffix(leaf, "Count") {
		return strings.TrimSuffix(leaf, "Count") + "Cost"
	}
	if strings.Contains(leaf, "_") {
		return leaf + "_cost"
	}
	return leaf + "Cost"
}

// buildAICost reports the cost of each token category under the name the
// provider returns that category's count at, so the breakdown reads like the
// response body. The usage container the counts sit in carries no meaning here
// and is dropped, leaving costs at the root and nesting only where the provider
// itself nests. Categories the template does not locate are omitted. Returns
// nil when no category is attributable, so the caller can leave the property
// out entirely.
func buildAICost(paths map[string]string, c costComponents, serviceTier string) map[string]interface{} {
	tree := map[string]interface{}{}

	for field, path := range paths {
		cost, owned := componentForField(field, c)
		if !owned {
			continue
		}
		segments := strings.Split(strings.TrimPrefix(path, "$."), ".")
		if len(segments) > 1 {
			segments = segments[1:]
		}
		if len(segments) == 0 || segments[0] == "" {
			continue
		}

		node := tree
		for _, segment := range segments[:len(segments)-1] {
			child, ok := node[segment].(map[string]interface{})
			if !ok {
				child = map[string]interface{}{}
				node[segment] = child
			}
			node = child
		}
		node[costKey(segments[len(segments)-1])] = cost
	}

	if len(tree) == 0 {
		return nil
	}

	tier := serviceTier
	if tier == "" {
		tier = "standard"
	}
	tree["serviceTier"] = tier
	return tree
}
```

- [ ] **Step 4: Run to verify it passes**

```bash
cd api-platform/gateway/dev-policies/llm-cost-v2
GOWORK=off go clean -testcache
GOWORK=off go test -run "BuildAICost|CostKeyNaming" -v ./... 2>&1 | grep -E "^(--- |ok|FAIL)"
```

Expected: all seven PASS.

- [ ] **Step 5: Package still green**

```bash
cd api-platform/gateway/dev-policies/llm-cost-v2
GOWORK=off go test ./... 2>&1 | tail -3
GOWORK=off go vet ./...
```

---

### Task 4: the policy publishes `aiCost`

**Files:**
- Modify: `api-platform/gateway/dev-policies/llm-cost-v2/llm_cost_v2.go`
- Test: `api-platform/gateway/dev-policies/llm-cost-v2/llm_cost_v2_test.go`

**Interfaces:**
- Consumes: `buildAICost` (Task 3), `FieldPaths` (Task 1).
- Produces: analytics metadata key `aiCost` holding the nested map. `costResult` gains `components costComponents` and `paths map[string]string`.

- [ ] **Step 1: Write the failing test**

Append to `llm_cost_v2_test.go`. `runResponse` returns only cost and status, so this test drives `OnResponseBodyChunk` directly to read the returned action:

```go
func TestOnResponseBodyChunk_PublishesNestedAICost(t *testing.T) {
	loadShippedTemplate(t, "gemini", "gemini-template.yaml")
	p := newTestPolicy(t)

	sc := &policy.SharedContext{Metadata: map[string]interface{}{
		llmusage.MetadataTemplateHandle: "gemini",
	}}
	respCtx := &policy.ResponseStreamContext{
		SharedContext: sc,
		RequestPath:   "/v1beta/models/gemini-3-flash-preview:generateContent",
	}
	body := []byte(`{"modelVersion":"gemini-3-flash-preview","usageMetadata":{` +
		`"promptTokenCount":10000,"candidatesTokenCount":500,` +
		`"thoughtsTokenCount":100,"serviceTier":"priority"}}`)

	action := p.OnResponseBodyChunk(context.Background(), respCtx,
		&policy.StreamBody{Chunk: body, EndOfStream: true, Index: 0}, nil)

	forward, ok := action.(policy.ForwardResponseChunk)
	if !ok {
		t.Fatalf("action type = %T, want ForwardResponseChunk", action)
	}
	aiCost, ok := forward.AnalyticsMetadata["aiCost"].(map[string]interface{})
	if !ok {
		t.Fatalf("aiCost missing or wrong type: %#v", forward.AnalyticsMetadata)
	}

	if aiCost["serviceTier"] != "priority" {
		t.Errorf("serviceTier = %v, want priority", aiCost["serviceTier"])
	}
	if _, present := aiCost["usageMetadata"]; present {
		t.Errorf("usageMetadata container should not appear: %#v", aiCost)
	}
	for _, key := range []string{"promptTokenCost", "candidatesTokenCost", "thoughtsTokenCost"} {
		v, present := aiCost[key]
		if !present {
			t.Errorf("%s absent from %#v", key, aiCost)
			continue
		}
		if cost, ok := v.(float64); !ok || cost <= 0 {
			t.Errorf("%s = %v, want a positive cost", key, v)
		}
	}
}
```

- [ ] **Step 2: Run to verify it fails**

```bash
cd api-platform/gateway/dev-policies/llm-cost-v2
GOWORK=off go clean -testcache
GOWORK=off go test -run TestOnResponseBodyChunk_PublishesNestedAICost -v ./... 2>&1 | head -12
```

Expected: FAIL — `aiCost missing`.

- [ ] **Step 3: Carry the components and paths out of pricing**

In `llm_cost_v2.go`, extend `costResult`:

```go
	components costComponents
	paths      map[string]string
```

In `price`, capture both. Replace the two pricing lines:

```go
	cost := genericCalculateCost(usage, pricing)
	if calc != nil {
		cost = calc.Adjust(cost, usage, pricing)
	}
```

with:

```go
	components := calculateCostComponents(usage, pricing)
	cost := components.total()
	if calc != nil {
		cost = calc.Adjust(cost, usage, pricing)
	}
```

and add to the returned `costResult` literal:

```go
		components: components,
		paths:      llmusage.FieldPaths(sc, requestPath),
```

`Adjust` applies multipliers to the total that the components do not carry, which is one of the reasons `aiCost` is documented as not summing to `llmCost`.

- [ ] **Step 4: Attach it in `analyticsFor`**

In `analyticsFor`, inside the `result.calculated` branch, after the token keys:

```go
		if aiCost := buildAICost(result.paths, result.components, result.serviceTier); aiCost != nil {
			metadata["aiCost"] = aiCost
		}
```

`costResult` has no `serviceTier` field yet — add `serviceTier string` to the struct and set it in `price` from `usage.ServiceTier` (the pricing-side `Usage`, which `toPricingUsage` already populates from the template).

- [ ] **Step 5: Run to verify it passes**

```bash
cd api-platform/gateway/dev-policies/llm-cost-v2
GOWORK=off go clean -testcache
GOWORK=off go test -run TestOnResponseBodyChunk_PublishesNestedAICost -v ./... 2>&1 | grep -E "^(--- |ok|FAIL)"
```

Expected: PASS.

- [ ] **Step 6: Full package, costs unchanged**

```bash
cd api-platform/gateway/dev-policies/llm-cost-v2
GOWORK=off go test ./... 2>&1 | tail -3
GOWORK=off go vet ./...
GOWORK=off go test -v ./... 2>&1 | grep -oE "0\.[0-9]{10}" | sort -u | diff /tmp/p6-baseline.txt - && echo "  costs IDENTICAL"
```

Expected: `ok`, vet silent, diff empty.

---

### Task 5: the analytics pipeline emits `aiCost`

The pipeline forwards only properties it names explicitly; the loop over the metadata map at `analytics.go:275` logs and copies nothing. `aiCost` therefore needs an explicit assignment, following `aiMetadata` (line 493) and `aiTokenUsage` (line 532). Unlike those two it is a dynamic map rather than a typed DTO, because its shape follows each provider's response body.

**Files:**
- Modify: `api-platform/gateway/gateway-runtime/policy-engine/internal/constants/constants.go:112-113`
- Modify: `api-platform/gateway/gateway-runtime/policy-engine/internal/analytics/analytics.go` (near line 493)
- Test: `api-platform/gateway/gateway-runtime/policy-engine/internal/analytics/analytics_test.go`

**Interfaces:**
- Consumes: `analytics_data` field `aiCost`, arriving via `typedValuePairsFromMetadata` as `map[string]interface{}`.
- Produces: `event.Properties["aiCost"]`.

- [ ] **Step 1: Write the failing test**

Append to `internal/analytics/analytics_test.go`. This uses `createLogEntryWithMetadataValues` (the `*structpb.Value` variant already used by `TestPrepareAnalyticEvent_WithGuardrailMetadata` at line 845), because a nested object cannot be expressed through the `map[string]string` helper:

```go
func TestPrepareAnalyticEvent_WithAICost(t *testing.T) {
	cfg := &config.Config{}
	analytics := NewAnalytics(cfg)

	aiCost, err := structpb.NewStruct(map[string]interface{}{
		"serviceTier": "priority",
		"usageMetadata": map[string]interface{}{
			"promptTokenCost":     0.001,
			"candidatesTokenCost": 0.002,
		},
	})
	require.NoError(t, err)

	logEntry := createLogEntryWithMetadataValues(map[string]*structpb.Value{
		AIProviderNameMetadataKey:    structpb.NewStringValue("gemini"),
		ModelIDMetadataKey:           structpb.NewStringValue("gemini-3-flash-preview"),
		constants.AICostMetadataKey:  structpb.NewStructValue(aiCost),
	})

	event := analytics.prepareAnalyticEvent(logEntry)

	require.NotNil(t, event)
	raw, ok := event.Properties[constants.AICostPropertyKey]
	require.True(t, ok, "aiCost absent from Properties")
	got, ok := raw.(map[string]interface{})
	require.True(t, ok, "aiCost is %T, want map[string]interface{}", raw)

	assert.Equal(t, "priority", got["serviceTier"])
	usageMetadata, ok := got["usageMetadata"].(map[string]interface{})
	require.True(t, ok, "nesting lost: %#v", got)
	assert.Equal(t, 0.001, usageMetadata["promptTokenCost"])
	assert.Equal(t, 0.002, usageMetadata["candidatesTokenCost"])
}
```

Note the metadata keys here are set as top-level filter-metadata fields, matching the guardrail test. If `createLogEntryWithMetadataValues` puts them outside the `analytics_data` struct, the implementation in Step 4 must read from the same map the guardrail keys are read from — check which of `typedValuePairsFromMetadata` the guardrail test relies on before finalising Step 4, since both paths populate it.

- [ ] **Step 2: Run to verify it fails**

```bash
cd api-platform/gateway/gateway-runtime/policy-engine
GOWORK=off go clean -testcache
GOWORK=off go test ./internal/analytics/ -run TestPrepareAnalyticEvent_WithAICost -v 2>&1 | head -12
```

Expected: FAIL — `aiCost` absent from Properties.

- [ ] **Step 3: Add the constant**

In `internal/constants/constants.go`, beside the existing LLM keys:

```go
	AICostMetadataKey        = "aiCost"
	AICostPropertyKey        = "aiCost"
```

- [ ] **Step 4: Emit it**

In `analytics.go`, inside the AI-metadata block and after `event.Properties["aiMetadata"] = aiMetadata`:

```go
		// The cost breakdown mirrors the provider's response body, so its shape
		// varies by provider and it is forwarded as it arrives rather than
		// through a fixed type.
		if raw, exists := typedValuePairsFromMetadata[constants.AICostMetadataKey]; exists {
			if aiCost, ok := raw.(map[string]interface{}); ok && len(aiCost) > 0 {
				event.Properties[constants.AICostPropertyKey] = aiCost
			}
		}
```

- [ ] **Step 5: Run to verify it passes**

```bash
cd api-platform/gateway/gateway-runtime/policy-engine
GOWORK=off go clean -testcache
GOWORK=off go test ./internal/analytics/ -run TestPrepareAnalyticEvent_WithAICost -v 2>&1 | grep -E "^(--- |ok|FAIL)"
```

Expected: PASS.

- [ ] **Step 6: Whole policy-engine green**

```bash
cd api-platform/gateway/gateway-runtime/policy-engine
GOWORK=off go test ./... 2>&1 | grep -vE "^ok|no test files" | head -8
GOWORK=off go vet ./... 2>&1 | head -3
```

Expected: no failures, vet silent.

---

### Task 6: end-to-end verification and rebuild

- [ ] **Step 1: Every touched module**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform
for m in sdk/ai/llmusage gateway/dev-policies/llm-cost-v2 gateway/gateway-runtime/policy-engine; do
  printf "%-42s " "$m"
  out=$(cd "$m" && GOWORK=off go clean -testcache 2>&1 && GOWORK=off go vet ./... 2>&1 && GOWORK=off go test ./... 2>&1)
  echo "$out" | grep -qE "^(FAIL|# )|\.go:[0-9]+:[0-9]+:" && { echo PROBLEM; echo "$out" | grep -E "^(FAIL|# )|\.go:" | head -3; } || echo "vet+test OK"
done
```

- [ ] **Step 2: Parity**

```bash
cd api-platform/gateway/dev-policies/llm-cost-v2
GOWORK=off go test -v ./... 2>&1 | grep -oE "0\.[0-9]{10}" | sort -u
```

Expected: contains `0.0002700000`, `0.0061500000`, `0.0072750000`, `0.0065000000`, `0.0117000000`; no `0.0000000000` for a priced model.

- [ ] **Step 3: Protected files untouched**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/gateway-controllers && git status --porcelain policies/llm-cost/
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform && git status --porcelain \
  gateway/build-manifest.yaml gateway/configs/config.toml gateway/configs/config-template.toml \
  gateway/configs/llm-pricing/model_prices.json
```

Expected: the first empty; the second showing only the owner's pre-existing modifications, with no `aiCost` in any diff.

- [ ] **Step 4: Nothing committed**

```bash
for r in /Users/irashperera/Documents/APIM/api-platform/dev/api-platform /Users/irashperera/Documents/APIM/api-platform/dev/gateway-controllers; do
  (cd "$r" && echo "$(basename $(pwd)): $(git log --oneline -1) | staged: $(git diff --cached --stat | tail -1)")
done
```

- [ ] **Step 5: Rebuild the runtime**

`aiCost` changes the policy-engine, so the **gateway-runtime** image must be rebuilt. The templates did not change, so the controller does not need rebuilding for this plan. The Python-policy clones stall intermittently; retry, and only treat a Go compile error as real:

```bash
cd api-platform/gateway
for i in 1 2 3 4 5; do
  echo "=== attempt $i ==="
  if make build-gateway-runtime > /tmp/p6_build_$i.log 2>&1; then echo SUCCESS; break; fi
  grep -qE "\.go:[0-9]+:[0-9]+:|undefined:|cannot use" /tmp/p6_build_$i.log && { echo "REAL COMPILE ERROR"; grep -nE "\.go:[0-9]+:[0-9]+:" /tmp/p6_build_$i.log | head; break; }
  echo "  network stall, retrying"
done
```

- [ ] **Step 6: Report**

State: which providers produce which `aiCost` keys, that parity held, that the analytics pipeline was modified (previously out of scope, now authorised), and anything skipped or failed — explicitly, not summarised around.

---

## Known limitations after this plan

- **`aiCost` does not sum to `llmCost`** when an ignored component is non-zero (image output, audio duration, web search, tool use) or when a provider calculator's `Adjust` applies a multiplier. By design, per decision 3.
- **Bedrock's cache TTL split is not broken out**, because its template declares only a single `cacheWriteTokens` total; the 5m/1h detail exists only inside `cacheDetails[]`, which needs a filter the library does not support. Anthropic's split *is* reported, because its template declares both TTLs as separate paths.
- **`aiCost` is a dynamic map, not a typed DTO**, unlike `aiMetadata` and `aiTokenUsage`. Its shape follows the provider's response body, so a fixed struct cannot express it.
- **The analytics system policy still does not read `valueMap` or `fallbackIdentifiers`** — unchanged by this plan; it has its own extractor over six template fields.
