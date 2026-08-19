# LLM Usage Extraction — Plan 4: URL Extraction and Removing the Duplicated `Normalize`

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give the extraction library the ability to read values out of the request URL, then delete
the duplicated `Normalize` implementations from `llm-cost-v2`'s calculators — restoring the two
fields that fell into the gap between template and Go while doing so.

**Architecture:** Two independent changes that must land in this order. First, `sdk/ai/llmusage`
gains `location: pathParam` support, which unblocks Bedrock (whose model exists only in the URL).
Second, `llm-cost-v2` narrows its calculator interface to `fees` + `Adjust` so `Normalize` has no
reason to exist, and the fee logic stranded inside it is moved out.

**Tech Stack:** Go 1.26.2, `sdk/core` (published), standard library.

**Reference spec:** `gateway/spec/llm-usage-extraction-design.md`.
**Builds on:** Plans 1-3, all uncommitted in the working tree.

## Two repositories

```
PLATFORM   /Users/irashperera/Documents/APIM/api-platform/dev/api-platform      ← all work here
POLICIES   /Users/irashperera/Documents/APIM/api-platform/dev/gateway-controllers ← READ ONLY
```

## Why this plan exists — the measurements

An audit of what `llm-cost`'s calculators read, against what `llm-cost-v2` covers, found three
problems:

**1. `Normalize` is dead weight.** 544 of 996 calculator lines, none of it reachable — the cost path
calls only `fees()`. It duplicates in Go exactly what the template now declares in YAML.

```
file          total   Normalize    fees
openai          159      82         47
anthropic       270     114         59
gemini          258     151         50
mistral          77      26         18
bedrock         232     171          0     ← no fee logic at all
                ────    ────       ────
                996     544        174
```

**2. Two fields vanished** — neither templated nor in `fees()`:

- **Gemini `ServiceTier`.** The original maps `trafficType` (`ON_DEMAND` → standard, else
  priority/flex). The template has no `serviceTier` for Gemini and `fees()` does not set it, so
  Gemini priority/flex traffic silently prices as standard.
- **Bedrock `CacheWrite1hrTokens`.** The original splits `cacheDetails[]` by TTL via
  `bedrockCacheWritesByTTL`. That function is reachable only from the dead `Normalize`, and the
  template declares just `cacheWriteTokens`, so the 5-minute/1-hour distinction is lost.

**3. Bedrock has no `fees()` and no `Adjust()`.** It embeds `OpenAICalculator`, so `applyFees`'s
type assertion silently matches **OpenAI's** web-search detection. It returns zeros only because
Bedrock responses have no `choices[].message.annotations` — correct by accident.

The root cause of 2 and 3: nothing verified that every field the original reads ends up on exactly
one side of the template/Go boundary. Task 7 adds that check as a test.

## What decides template vs Go

A field belongs in the template only if **all three** hold. Otherwise it is Go's job.

```
1. the schema has a name for it            (12 names, closed vocabulary)
2. it is a plain location, not a computation
3. it is in the response body               (location: payload)  — until Task 1
```

Every gap found is condition 2 or 3. After Task 1, condition 3 widens to include the request URL.

A third pattern also exists and Bedrock's model uses it: **the template locates a raw value and Go
refines it.** One field, both layers.

## Global Constraints

- **Never commit and never stage, in either repo.** Do not run `git commit`, `git add`,
  `git stash`, `git reset`, `git restore`, or **`git checkout <file>`**. The owner commits.
  (`git checkout <file>` has already destroyed needed work once in this project.)
- **`llm-cost` must not be modified.** It is the parity reference. Confirm the `POLICIES` repo is
  clean at the end of every task.
- Do not touch the owner's unrelated uncommitted files: `gateway/build-manifest.yaml`,
  `gateway/configs/config.toml`, `gateway/configs/config-template.toml`,
  `gateway/configs/llm-pricing/model_prices.json`.
- **Cost for the four providers that currently match `llm-cost` must not change.** OpenAI,
  Anthropic, Gemini and Mistral parity tests pass today; they must still pass after every task. A
  change there is a regression, not an improvement.
- Comments: short and explanatory. NEVER write a comment describing a fix, a change, or history
  (no "fixed this", "moved from Normalize", "previously this did Y", "now only fees").
- Use `GOWORK=off` for every build and test command. The Docker build compiles that way, and a
  workspace-enabled command has already masked a real defect in this project.
- Do not run `go build ./...` from the repo root — there is no `go.mod` there, only `go.work`.

## File Structure

```
PLATFORM/sdk/ai/llmusage/
  decode.go            MODIFIED: read pathParam as well as payload
  pathparam.go         NEW: regex-against-URL extraction + percent-decoding
  pathparam_test.go    NEW
  decode_test.go       MODIFIED: pathParam cases

PLATFORM/gateway/gateway-controller/default-llm-provider-templates/
  awsbedrock-template.yaml   MODIFIED: widen the model regex, add cacheWrite1hTokens
  gemini-template.yaml       unchanged (trafficType needs mapping, so it stays in Go)

PLATFORM/gateway/dev-policies/llm-cost-v2/
  pricing.go           MODIFIED: excise the calculator interface and selectCalculator
  calculators.go       NEW: the narrow interface + selectCalculator
  fees.go              MODIFIED: applyFees takes the narrow type
  calculator_*.go      MODIFIED: Normalize deleted; bedrock gains fees + Adjust;
                                 gemini gains the trafficType mapping
  completeness_test.go NEW: asserts every original field is covered
```

---

### Task 1: Teach the library to read values from the URL

**Files:**
- Create: `sdk/ai/llmusage/pathparam.go`
- Create: `sdk/ai/llmusage/pathparam_test.go`
- Modify: `sdk/ai/llmusage/decode.go` (`readString` accepts `pathParam`)

**Interfaces:**
- Consumes: `fieldSpec` from `template.go`.
- Produces: `func extractFromPath(requestPath, pattern string) string`. `readString` routes to it
  when `spec.Location == locationPathParam`.

**Design decision, stated because it is a judgement call:** the library **percent-decodes** what it
extracts. Reading a value out of a URL correctly includes undoing URL encoding — that is not
provider-specific interpretation, it is part of reading a URL at all. What the library does **not**
do is understand what the decoded value *means*: cutting an ARN down to a model ID stays in Go,
where `pricing.go`'s `bedrockModelAliases` already handles it.

- [ ] **Step 1: Write the failing test**

Create `sdk/ai/llmusage/pathparam_test.go`:

```go
package llmusage

import "testing"

func TestExtractFromPath(t *testing.T) {
	const bedrockPattern = `model/([A-Za-z0-9.:%-]+)/`

	tests := []struct {
		name        string
		requestPath string
		pattern     string
		want        string
	}{
		{
			name:        "plain bedrock model id",
			requestPath: "/model/anthropic.claude-3-sonnet-20240229-v1:0/converse",
			pattern:     bedrockPattern,
			want:        "anthropic.claude-3-sonnet-20240229-v1:0",
		},
		{
			name:        "cross-region inference profile",
			requestPath: "/model/us.anthropic.claude-3-5-sonnet-20240620-v1:0/converse-stream",
			pattern:     bedrockPattern,
			want:        "us.anthropic.claude-3-5-sonnet-20240620-v1:0",
		},
		{
			name:        "percent-encoded arn is decoded",
			requestPath: "/model/arn%3Aaws%3Abedrock%3Aus-east-1%3A123%3Ainference-profile%2Fus.anthropic.claude-3-5-sonnet-20240620-v1%3A0/converse",
			pattern:     bedrockPattern,
			want:        "arn:aws:bedrock:us-east-1:123:inference-profile/us.anthropic.claude-3-5-sonnet-20240620-v1:0",
		},
		{
			name:        "gemini lookbehind pattern",
			requestPath: "/v1beta/models/gemini-2.5-flash:streamGenerateContent",
			pattern:     `models/([a-zA-Z0-9.\-]+)`,
			want:        "gemini-2.5-flash",
		},
		{
			name:        "no match returns empty",
			requestPath: "/chat/completions",
			pattern:     bedrockPattern,
			want:        "",
		},
		{
			name:        "unparseable pattern returns empty rather than panicking",
			requestPath: "/model/x/converse",
			pattern:     `model/([A-Za-z`,
			want:        "",
		},
		{
			name:        "pattern with no capture group returns empty",
			requestPath: "/model/x/converse",
			pattern:     `model/`,
			want:        "",
		},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			if got := extractFromPath(tt.requestPath, tt.pattern); got != tt.want {
				t.Errorf("extractFromPath() = %q, want %q", got, tt.want)
			}
		})
	}
}

func TestExtractUsage_ModelFromPathParam(t *testing.T) {
	tmpl := map[string]interface{}{
		"promptTokens": map[string]interface{}{
			"location": "payload", "identifier": "$.usage.inputTokens",
		},
		"completionTokens": map[string]interface{}{
			"location": "payload", "identifier": "$.usage.outputTokens",
		},
		"responseModel": map[string]interface{}{
			"location": "pathParam", "identifier": `model/([A-Za-z0-9.:%-]+)/`,
		},
	}
	// A Bedrock Converse response carries no model field at all.
	body := []byte(`{"usage":{"inputTokens":100,"outputTokens":50}}`)

	got, err := extractUsage(tmpl, body, nil, "/model/anthropic.claude-3-sonnet-20240229-v1:0/converse")
	if err != nil {
		t.Fatalf("extractUsage returned error: %v", err)
	}

	if got.Model != "anthropic.claude-3-sonnet-20240229-v1:0" {
		t.Errorf("Model = %q, want the id from the request path", got.Model)
	}
	if got.TotalInputTokens != 100 {
		t.Errorf("TotalInputTokens = %d, want 100", got.TotalInputTokens)
	}
}
```

- [ ] **Step 2: Run it and confirm it fails**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/sdk/ai/llmusage
GOWORK=off go test ./... -run 'ExtractFromPath|ModelFromPathParam' 2>&1 | head -6
```

Expected: `undefined: extractFromPath`.

- [ ] **Step 3: Write the extractor**

Create `sdk/ai/llmusage/pathparam.go`:

```go
package llmusage

import (
	"net/url"
	"regexp"
	"sync"
)

// pathPatternCache holds compiled patterns so a template's regex is compiled
// once rather than on every request.
var pathPatternCache sync.Map

// extractFromPath returns the first capture group of pattern applied to the
// request path, percent-decoded. An unparseable pattern, a pattern with no
// capture group, or no match yields an empty string, which callers treat the
// same as an absent field.
func extractFromPath(requestPath, pattern string) string {
	re := compilePathPattern(pattern)
	if re == nil {
		return ""
	}

	match := re.FindStringSubmatch(requestPath)
	if len(match) < 2 {
		return ""
	}

	// Values taken from a URL may be percent-encoded; AWS encodes ARN model
	// identifiers because their resource component can contain a slash.
	if decoded, err := url.PathUnescape(match[1]); err == nil {
		return decoded
	}
	return match[1]
}

// compilePathPattern compiles and caches a pattern, returning nil when it is
// not valid so a malformed template cannot break request handling.
func compilePathPattern(pattern string) *regexp.Regexp {
	if cached, ok := pathPatternCache.Load(pattern); ok {
		re, _ := cached.(*regexp.Regexp)
		return re
	}

	re, err := regexp.Compile(pattern)
	if err != nil {
		pathPatternCache.Store(pattern, (*regexp.Regexp)(nil))
		return nil
	}
	pathPatternCache.Store(pattern, re)
	return re
}
```

- [ ] **Step 4: Route `pathParam` fields to it**

In `decode.go`, `readString` currently rejects any location other than `payload`:

```go
if !ok || spec.Location != locationPayload {
    return ""
}
```

`readString` has no access to the request path, so it must be threaded in. Change `readString` and
`readInt` to take `requestPath string`, and route by location:

```go
// readString reads a declared field. Payload fields are read from the given
// JSON document; pathParam fields are matched against the request path.
func readString(payload []byte, fields map[string]fieldSpec, name, requestPath string) string {
	spec, ok := fields[name]
	if !ok {
		return ""
	}

	switch spec.Location {
	case locationPayload:
		value, err := utils.ExtractStringValueFromJsonpath(payload, spec.Identifier)
		if err != nil {
			return ""
		}
		return strings.TrimSpace(value)
	case locationPathParam:
		return extractFromPath(requestPath, spec.Identifier)
	default:
		return ""
	}
}
```

Add the constant beside `locationPayload`:

```go
// locationPathParam is the ExtractionIdentifier location for the request path.
const locationPathParam = "pathParam"
```

Update every `readInt` / `readString` call in `extractUsage` and `resolveModel` to pass
`requestPath`. `resolveModel` also needs the parameter added to its signature.

Header and query-parameter locations remain unread. Leave them returning empty for now — Task 6
records that gap.

- [ ] **Step 5: Run the whole library suite**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/sdk/ai/llmusage
GOWORK=off go clean -testcache && GOWORK=off go build ./... && GOWORK=off go vet ./... \
  && GOWORK=off go test ./... -v 2>&1 | tail -20
```

Expected: all pass — the 31 existing tests plus 8 new ones.

- [ ] **Step 6: Confirm the four passing providers still pass**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/gateway/dev-policies/llm-cost-v2
GOWORK=off go clean -testcache && GOWORK=off go test ./... 2>&1 | grep -E "^(ok|FAIL)"
```

Expected: `ok`. Threading `requestPath` through touches every field read, so a regression here
would show as a parity failure.

- [ ] **Step 7: Leave uncommitted and report**

Do not commit or stage. Confirm the `POLICIES` repo is clean.

---

### Task 2: Make Bedrock resolve its model

**Files:**
- Modify: `gateway/gateway-controller/default-llm-provider-templates/awsbedrock-template.yaml`

**Interfaces:**
- Consumes: `pathParam` support from Task 1.
- Produces: a Bedrock template whose model resolves. Task 6's tests depend on it.

- [ ] **Step 1: Widen the model regex and declare the 1-hour cache field**

The current pattern is `model/([A-Za-z0-9.:-]+)/`. It matches plain model IDs but **not**
percent-encoded ARNs, because the character class excludes `%`. Verified: an
`inference-profile` ARN path returns no match at all.

In `awsbedrock-template.yaml`, change **both** `requestModel` and `responseModel` identifiers to:

```yaml
    identifier: model/([A-Za-z0-9.:%-]+)/
```

Add nothing else to those two fields — `location: pathParam` is already correct.

Also add `cacheWrite1hTokens` beside the existing `cacheWriteTokens`. Bedrock reports the per-TTL
split inside `cacheDetails[]`, an array a path cannot select from, so this declaration is a
placeholder that will read nothing on Converse responses — the real split is computed in Go in
Task 3. Do **not** add it if it would read a wrong value; check the Converse response shape first
and, if there is no scalar 1-hour field, **leave it out** and note that in your report.

- [ ] **Step 2: Verify the templates still parse and validate**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/gateway/gateway-controller
GOWORK=off go clean -testcache && GOWORK=off go test ./pkg/config/ ./pkg/utils/ 2>&1 | grep -E "^(ok|FAIL)"
```

Expected: `ok` for both.

- [ ] **Step 3: Confirm `llm-cost` is unaffected**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/gateway-controllers/policies/llm-cost
GOWORK=off go clean -testcache && GOWORK=off go test ./... 2>&1 | grep -E "^(ok|FAIL)"
```

Expected: `ok`.

- [ ] **Step 4: Leave uncommitted and report**

Do not commit or stage. State whether you added `cacheWrite1hTokens` and why.


---

### Task 3: Let a field declare fallback locations

Bedrock's `Normalize` is not declaring *where* data lives — it is **sniffing which of four response
shapes arrived**, reading all four field sets into one struct and using whichever populated:

```
Converse / Nova InvokeModel   inputTokens          outputTokens                cacheDetails[]
Anthropic InvokeModel         input_tokens         output_tokens               cache_read_input_tokens
OpenAI (via transformer)      prompt_tokens        completion_tokens           prompt_tokens_details
Amazon Titan InvokeModel      inputTextTokenCount  results[0].tokenCount       —
```

`resourceMappings` cannot discriminate these: `/converse` identifies one shape, but Anthropic, Nova
and Titan **all** arrive on `/invoke`, and the transformer case rewrites the body while leaving the
path unchanged.

The real limitation is narrower than "this needs Go": a field can declare only **one** location.
Given a fallback list, the sniffing becomes declarative and Bedrock's `Normalize` loses its last
reason to exist.

`ExtractionIdentifier` is a **shared** type used by all twelve fields, so one schema edit gives every
field this capability.

**Files:**
- Modify: `gateway/gateway-controller/api/management-openapi.yaml` (`ExtractionIdentifier`)
- Regenerate: `gateway/gateway-controller/pkg/api/management/generated.go`
- Modify: `kubernetes/gateway-operator/api/v1/llmprovidertemplate_types.go`
- Modify: `kubernetes/gateway-operator/api/v1alpha1/llmprovidertemplate_types.go`
- Regenerate: both `zz_generated.deepcopy.go`, `config/crd/bases/...yaml`; copy to helm
- Modify: `gateway/gateway-controller/pkg/config/llm_validator.go`
- Modify: `sdk/ai/llmusage/template.go`, `decode.go`
- Modify: `sdk/ai/llmusage/template_test.go`, `decode_test.go`

**Interfaces:**
- Produces: `fieldSpec.Identifiers []string` — the ordered candidate list, with the required
  `identifier` first. `readString` tries each until one yields a value.

**Design decision:** `identifier` stays **required** and remains the first candidate;
`fallbackIdentifiers` is optional and appended after it. This keeps every existing template valid
unchanged and leaves no ambiguity about precedence. The alternative — replacing `identifier` with a
list — would be a breaking schema change for no gain.

- [ ] **Step 1: Add the field to the OpenAPI spec**

In `management-openapi.yaml`, inside `ExtractionIdentifier` (around line 4193), after the existing
`identifier` property:

```yaml
        fallbackIdentifiers:
          type: array
          items:
            type: string
          description: |
            Additional locations to try when the primary identifier yields no
            value, in order. Used where one provider returns several response
            shapes on the same path.
          example:
            - $.usage.input_tokens
            - $.usage.prompt_tokens
```

Then regenerate:

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/gateway/gateway-controller
make generate-server-code
grep -n "FallbackIdentifiers" pkg/api/management/generated.go | head -3
```

Expected: `FallbackIdentifiers *[]string` on `ExtractionIdentifier`.

- [ ] **Step 2: Add it to both CRD versions and regenerate**

In `api/v1/llmprovidertemplate_types.go` and `api/v1alpha1/llmprovidertemplate_types.go`, add to
`ExtractionIdentifier`:

```go
	// FallbackIdentifiers are additional locations to try, in order, when the
	// primary identifier yields no value.
	// +optional
	FallbackIdentifiers []string `json:"fallbackIdentifiers,omitempty"`
```

Both versions need it — conversion goes through `convertViaJSON`, so a field missing from
`v1alpha1` is dropped on a round trip.

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/kubernetes/gateway-operator
make generate && make manifests
cd .. && cp gateway-operator/config/crd/bases/gateway.api-platform.wso2.com_llmprovidertemplates.yaml    helm/operator-helm-chart/crds/gateway.api-platform.wso2.com_llmprovidertemplates.yaml
diff -q gateway-operator/config/crd/bases/gateway.api-platform.wso2.com_llmprovidertemplates.yaml         helm/operator-helm-chart/crds/gateway.api-platform.wso2.com_llmprovidertemplates.yaml
```

Expected: `diff -q` silent. Report any file `make manifests` touched beyond the expected six.

- [ ] **Step 3: Validate the new field**

In `llm_validator.go`, extend `validateExtractionIdentifier` to reject an empty string inside
`fallbackIdentifiers`, since an empty JSONPath means "return the whole document" and would silently
match everything:

```go
	for i, fallback := range identifier.FallbackIdentifiers {
		if fallback == "" {
			errors = append(errors, ValidationError{
				Field:   fmt.Sprintf("%s.fallbackIdentifiers[%d]", fieldPrefix, i),
				Message: "fallback identifier cannot be empty",
			})
		}
	}
```

Add a test in `llm_validator_additional_test.go` asserting that exact field path.

- [ ] **Step 4: Write the failing library test**

Append to `sdk/ai/llmusage/decode_test.go`:

```go
func TestExtractUsage_FallbackIdentifiersTryInOrder(t *testing.T) {
	// One template covering three Bedrock response shapes.
	ident := func(primary string, fallbacks ...string) map[string]interface{} {
		spec := map[string]interface{}{"location": "payload", "identifier": primary}
		if len(fallbacks) > 0 {
			list := make([]interface{}, 0, len(fallbacks))
			for _, f := range fallbacks {
				list = append(list, f)
			}
			spec["fallbackIdentifiers"] = list
		}
		return spec
	}
	tmpl := map[string]interface{}{
		"promptTokens":     ident("$.usage.inputTokens", "$.usage.input_tokens", "$.usage.prompt_tokens", "$.inputTextTokenCount"),
		"completionTokens": ident("$.usage.outputTokens", "$.usage.output_tokens", "$.usage.completion_tokens", "$.results[0].tokenCount"),
	}

	tests := []struct {
		name      string
		body      string
		wantIn    int64
		wantOut   int64
	}{
		{"converse shape", `{"usage":{"inputTokens":100,"outputTokens":50}}`, 100, 50},
		{"anthropic invokemodel shape", `{"usage":{"input_tokens":200,"output_tokens":60}}`, 200, 60},
		{"openai transformed shape", `{"usage":{"prompt_tokens":300,"completion_tokens":70}}`, 300, 70},
		{"titan shape", `{"inputTextTokenCount":400,"results":[{"tokenCount":80}]}`, 400, 80},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got, err := extractUsage(tmpl, []byte(tt.body), nil, "/model/x/invoke")
			if err != nil {
				t.Fatalf("extractUsage returned error: %v", err)
			}
			if got.TotalInputTokens != tt.wantIn {
				t.Errorf("TotalInputTokens = %d, want %d", got.TotalInputTokens, tt.wantIn)
			}
			if got.OutputTokens != tt.wantOut {
				t.Errorf("OutputTokens = %d, want %d", got.OutputTokens, tt.wantOut)
			}
		})
	}
}

func TestExtractUsage_PrimaryIdentifierWinsOverFallback(t *testing.T) {
	tmpl := map[string]interface{}{
		"promptTokens": map[string]interface{}{
			"location":            "payload",
			"identifier":          "$.usage.inputTokens",
			"fallbackIdentifiers": []interface{}{"$.usage.prompt_tokens"},
		},
	}
	// Both present: the primary must win.
	body := []byte(`{"usage":{"inputTokens":100,"prompt_tokens":999}}`)

	got, err := extractUsage(tmpl, body, nil, "/x")
	if err != nil {
		t.Fatalf("extractUsage returned error: %v", err)
	}
	if got.TotalInputTokens != 100 {
		t.Errorf("TotalInputTokens = %d, want 100 from the primary identifier", got.TotalInputTokens)
	}
}
```

- [ ] **Step 5: Implement it in the library**

In `template.go`, extend `fieldSpec` and `readFieldSpec`:

```go
// fieldSpec is one extraction identifier from a template, with any fallback
// locations to try after it.
type fieldSpec struct {
	Location    string
	Identifier  string
	Identifiers []string
}
```

In `readFieldSpec`, build `Identifiers` as the primary followed by each `fallbackIdentifiers` entry,
skipping non-strings and empties. In `readString`, try each in order and return the first non-empty
result.

Keep the existing single-identifier behaviour intact: a spec with no fallbacks must behave exactly
as before.

- [ ] **Step 6: Run everything**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/sdk/ai/llmusage
GOWORK=off go clean -testcache && GOWORK=off go build ./... && GOWORK=off go vet ./...   && GOWORK=off go test ./... 2>&1 | grep -E "^(ok|FAIL)"
cd ../../../gateway/gateway-controller
GOWORK=off go clean -testcache && GOWORK=off go test ./pkg/config/ ./pkg/utils/ 2>&1 | grep -E "^(ok|FAIL)"
cd ../dev-policies/llm-cost-v2
GOWORK=off go clean -testcache && GOWORK=off go test ./... 2>&1 | grep -E "^(ok|FAIL)"
```

Expected: all `ok`, with every existing parity cost unchanged.

- [ ] **Step 7: Leave uncommitted and report**

Do not commit or stage. Report the regenerated files and confirm the `POLICIES` repo is clean.

---

### Task 4: Declare Bedrock's shapes, and retire the library's hardcoded model chain

**Files:**
- Modify: `gateway/gateway-controller/default-llm-provider-templates/awsbedrock-template.yaml`
- Modify: `gateway/gateway-controller/default-llm-provider-templates/anthropic-template.yaml`
- Modify: `sdk/ai/llmusage/decode.go` (`resolveModel`)

**Interfaces:**
- Consumes: `fallbackIdentifiers` from Task 3.

- [ ] **Step 1: Declare Bedrock's four shapes**

In `awsbedrock-template.yaml`, add fallbacks so one template covers every shape. Confirm each path
against `llm-cost`'s `calculator_bedrock.go` before writing it — a wrong path here silently reads
zero:

```yaml
  promptTokens:
    location: payload
    identifier: $.usage.inputTokens
    fallbackIdentifiers:
      - $.usage.input_tokens
      - $.usage.prompt_tokens
      - $.inputTextTokenCount
  completionTokens:
    location: payload
    identifier: $.usage.outputTokens
    fallbackIdentifiers:
      - $.usage.output_tokens
      - $.usage.completion_tokens
      - $.results[0].tokenCount
  cachedTokens:
    location: payload
    identifier: $.usage.cacheReadInputTokens
    fallbackIdentifiers:
      - $.usage.cache_read_input_tokens
      - $.usage.prompt_tokens_details.cached_tokens
  cacheWriteTokens:
    location: payload
    identifier: $.usage.cacheWriteInputTokens
    fallbackIdentifiers:
      - $.usage.cache_creation_input_tokens
      - $.usage.prompt_tokens_details.cache_write_tokens
```

`$.results[0].tokenCount` was verified to resolve — the JSONPath helper supports fixed array
indices, including negative ones.

- [ ] **Step 2: Retire the library's hardcoded model fallback**

`resolveModel` in `decode.go` currently hardcodes the response-then-request order, and the library
carries no provider knowledge beyond that. With fallbacks available, the chain
`$.model` → `$.modelVersion` → `$.message.model` belongs in the templates instead.

Add to `anthropic-template.yaml`'s `responseModel` the streaming envelope location:

```yaml
  responseModel:
    location: payload
    identifier: $.model
    fallbackIdentifiers:
      - $.message.model
```

Gemini already declares `$.modelVersion` as its `responseModel`, so it needs nothing.

Then simplify `resolveModel` to rely on the declared candidates rather than any built-in list. Keep
the response-model-before-request-model precedence — that ordering is policy, not provider
knowledge — and keep returning `ModelCandidates` so a consumer can still see what was tried.

- [ ] **Step 3: Verify streaming model resolution still works**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/sdk/ai/llmusage
GOWORK=off go clean -testcache && GOWORK=off go test ./... -run 'SSE|Model' -v 2>&1 | grep -E "^(--- |ok|FAIL)"
```

The existing SSE tests cover the `$.message.model` case. If any now fails, the template fallback is
not being consulted where the hardcoded chain used to apply — fix that rather than restoring the
hardcoded list.

- [ ] **Step 4: Confirm Bedrock resolves and prices**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/gateway/dev-policies/llm-cost-v2
GOWORK=off go clean -testcache && GOWORK=off go test ./... -v 2>&1 | grep -iE "bedrock" | head
```

- [ ] **Step 5: Full regression**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform
(cd sdk/ai/llmusage && GOWORK=off go test ./... 2>&1 | grep -E "^(ok|FAIL)")
(cd gateway/gateway-controller && GOWORK=off go test ./pkg/config/ ./pkg/utils/ 2>&1 | grep -E "^(ok|FAIL)")
(cd gateway/dev-policies/llm-cost-v2 && GOWORK=off go test ./... 2>&1 | grep -E "^(ok|FAIL)")
(cd ../gateway-controllers/policies/llm-cost && GOWORK=off go test ./... 2>&1 | grep -E "^(ok|FAIL)")
```

Expected: all `ok`, every previously-passing parity cost unchanged.

- [ ] **Step 6: Leave uncommitted and report**

Do not commit or stage.

---

### Task 5: Give Bedrock and Gemini their missing fee logic

This restores the two fields that vanished. Do it **before** Task 6 deletes `Normalize`, because
the logic being moved currently lives inside it.

**Files:**
- Modify: `gateway/dev-policies/llm-cost-v2/calculator_bedrock.go`
- Modify: `gateway/dev-policies/llm-cost-v2/calculator_gemini.go`
- Modify: `gateway/dev-policies/llm-cost-v2/fees_test.go`

**Interfaces:**
- Consumes: `bedrockCacheWritesByTTL` (already present, currently unreachable).
- Produces: `(*BedrockCalculator).fees`, `(*BedrockCalculator).Adjust`, and a Gemini `fees` that
  sets `ServiceTier`.

- [ ] **Step 1: Establish what the original does for each**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/gateway-controllers/policies/llm-cost
grep -n "bedrockCacheWritesByTTL" -B 12 -A 6 calculator_bedrock.go
grep -n "TrafficType" -A 4 calculator_gemini.go
sed -n '150,170p' calculator_gemini.go
```

Record both verbatim in your report. Bedrock buckets `cacheDetails[]` by `ttl`; Gemini maps
`trafficType` to a tier string. Reproduce the same mapping values exactly — a different mapping is
a different bill.

- [ ] **Step 2: Write the failing tests**

Append to `fees_test.go`:

```go
func TestApplyFees_BedrockSplitsCacheWritesByTTL(t *testing.T) {
	// Bedrock reports the per-TTL split inside cacheDetails, which a JSONPath
	// cannot select from, so the split is computed here.
	body := []byte(`{"usage":{
		"inputTokens":1000,"outputTokens":200,
		"cacheWriteInputTokens":500,
		"cacheDetails":[{"ttl":"5m","inputTokens":300},{"ttl":"1h","inputTokens":200}]}}`)

	got := applyFees(&BedrockCalculator{}, Usage{PromptTokens: 1000}, body, nil)

	if got.CacheWriteTokens != 300 {
		t.Errorf("CacheWriteTokens = %d, want 300 (the 5m bucket)", got.CacheWriteTokens)
	}
	if got.CacheWrite1hrTokens != 200 {
		t.Errorf("CacheWrite1hrTokens = %d, want 200 (the 1h bucket)", got.CacheWrite1hrTokens)
	}
}

func TestApplyFees_BedrockDoesNotInheritOpenAIWebSearch(t *testing.T) {
	// BedrockCalculator embeds OpenAICalculator; without its own fees method the
	// type assertion would match OpenAI's web-search detection.
	body := []byte(`{"choices":[{"message":{"annotations":[{"type":"url_citation"}]}}],
		"usage":{"inputTokens":10,"outputTokens":5}}`)

	got := applyFees(&BedrockCalculator{}, Usage{PromptTokens: 10}, body, nil)

	if got.WebSearchRequests != 0 {
		t.Errorf("WebSearchRequests = %d, want 0; Bedrock has no web-search fee", got.WebSearchRequests)
	}
}

func TestApplyFees_GeminiMapsTrafficTypeToTier(t *testing.T) {
	// The Vertex AI enum is TRAFFIC_TYPE_UNSPECIFIED, ON_DEMAND,
	// ON_DEMAND_PRIORITY, ON_DEMAND_FLEX, PROVISIONED_THROUGHPUT. Only the two
	// on-demand variants carry their own rates.
	tests := []struct {
		trafficType string
		want        string
	}{
		{"ON_DEMAND_PRIORITY", "priority"},
		{"ON_DEMAND_FLEX", "flex"},
		{"ON_DEMAND", ""},
		{"PROVISIONED_THROUGHPUT", ""},
		{"TRAFFIC_TYPE_UNSPECIFIED", ""},
		{"", ""},
	}

	for _, tt := range tests {
		body := []byte(`{"usageMetadata":{"promptTokenCount":10,"trafficType":"` + tt.trafficType + `"}}`)
		got := applyFees(&GeminiCalculator{}, Usage{PromptTokens: 10}, body, nil)
		if got.ServiceTier != tt.want {
			t.Errorf("trafficType %q -> ServiceTier %q, want %q", tt.trafficType, got.ServiceTier, tt.want)
		}
	}
}
```

**Use the mapping above, not `llm-cost`'s.** Verified against the Vertex AI reference: the enum is
`TRAFFIC_TYPE_UNSPECIFIED`, `ON_DEMAND`, `ON_DEMAND_PRIORITY`, `ON_DEMAND_FLEX`,
`PROVISIONED_THROUGHPUT`. `llm-cost` matches `"FLEX"` and `"BATCH"`, **neither of which exists**, so
it never recognises the real flex value. Reproducing that bug would be wrong.

This is spec-correct and costs nothing in parity terms: no gemini, `vertex_ai` or
`vertex_ai-language-models` entry in the shipped pricing file has a flex rate (0 of 143), so no
cost can move. Priority rates do exist and `ON_DEMAND_PRIORITY` behaves identically to `llm-cost`.

Note this in your report as a deliberate, documented divergence from the original.

- [ ] **Step 3: Run and confirm they fail for the right reason**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/gateway/dev-policies/llm-cost-v2
GOWORK=off go test ./... -run 'Bedrock|GeminiMapsTraffic' 2>&1 | head -12
```

Expected: the Bedrock TTL and Gemini tier tests fail on values, not on compile errors. The
`DoesNotInheritOpenAIWebSearch` test may already fail, which is the bug it documents.

- [ ] **Step 4: Change `fees` into a refiner, then add Bedrock's**

`fees` currently returns a fresh `Usage` carrying only fee fields, and `applyFees` copies those
across. That works while template-supplied fields and fee fields are disjoint sets — but Bedrock's
cache-write split breaks that: the **template** reads the aggregate and **Go** computes the per-TTL
split of the same field. Two owners for one value.

So change the signature across all five calculators to take what the template already produced:

```go
fees(body, requestBody []byte, current Usage) Usage
```

`applyFees` passes the template-derived usage in. Each `fees` returns only the fields it owns, as
before — the parameter exists so a calculator can *refine* a template value rather than compete
with it.

This mirrors `bedrockCacheWritesByTTL`'s existing signature, which already takes the aggregate:

```go
func bedrockCacheWritesByTTL(details []bedrockCacheDetail, aggregate int64) (int64, int64)
```

It falls back sensibly on its own — no `cacheDetails[]` returns `(aggregate, 0)`, and any aggregate
not covered by a recognised TTL is added to the 5-minute bucket. So Bedrock's `fees` becomes:

```go
cacheWrite5m, cacheWrite1h := bedrockCacheWritesByTTL(details, current.CacheWriteTokens)
```

Add that `fees` to `calculator_bedrock.go`, returning a `Usage` carrying **only**
`CacheWriteTokens` and `CacheWrite1hrTokens`. `applyFees` must then copy those two fields as well —
they were not in its list before.

Add an explicit `Adjust` so Bedrock stops inheriting OpenAI's:

```go
// Adjust applies no post-calculation correction; Bedrock prices purely per token.
func (c *BedrockCalculator) Adjust(baseCost float64, _ Usage, _ ModelPricing) float64 {
	return baseCost
}
```

Confirm against the original that OpenAI's `Adjust` is also a pass-through, so this is not a
behaviour change. If it is not, reproduce whatever the original does for Bedrock.

- [ ] **Step 5: Add Gemini's tier mapping to its `fees`**

Extend `(*GeminiCalculator).fees` to read `$.usageMetadata.trafficType` and set `ServiceTier` using
the exact mapping from Step 1.

- [ ] **Step 6: Run the suite**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/gateway/dev-policies/llm-cost-v2
GOWORK=off go clean -testcache && GOWORK=off go test ./... -v 2>&1 | tail -25
```

Expected: all pass, including the four providers' existing parity tests.

- [ ] **Step 7: Leave uncommitted and report**

Do not commit or stage. Report the exact `trafficType` mapping you used and where you got it.

---

### Task 6: Narrow the interface and delete `Normalize`

**Files:**
- Modify: `gateway/dev-policies/llm-cost-v2/pricing.go` (excise lines ~240-272)
- Create: `gateway/dev-policies/llm-cost-v2/calculators.go`
- Modify: `gateway/dev-policies/llm-cost-v2/fees.go` (`applyFees` takes the narrow type)
- Modify: all five `calculator_*.go` (delete `Normalize`)

**Interfaces:**
- Produces: `type feeCalculator interface { fees(...); Adjust(...) }` and
  `selectCalculator(provider string) feeCalculator`, both in `calculators.go`.

**Why `pricing.go` must change:** it declares `providerCalculator` (requiring `Normalize`) and
`selectCalculator` returning the five concrete types. While those lines exist, every calculator
must keep a `Normalize` method. Moving them out is what allows the deletion.

This ends `pricing.go`'s status as a byte-for-byte copy of the original. Record the excision
explicitly in your report — future auditors need to know it is "verbatim minus lines 240-272".

- [ ] **Step 1: Move the interface and selector into their own file**

Create `calculators.go` with a narrow interface — only what the cost path calls:

```go
package llmcostv2

// feeCalculator supplies the per-provider charges that cannot be expressed as a
// template field path, plus any post-calculation correction.
type feeCalculator interface {
	fees(responseBody, requestBody []byte, current Usage) Usage
	Adjust(baseCost float64, usage Usage, pricing ModelPricing) float64
}

// selectCalculator returns the calculator for a pricing entry's provider value,
// or nil when the provider has no provider-specific charges.
func selectCalculator(provider string) feeCalculator {
	switch provider {
	...
	}
}
```

Copy the `switch` body verbatim from `pricing.go` lines 252-272, including every `vertex_ai*`
alias — a missed alias silently drops that provider's fee logic.

- [ ] **Step 2: Excise the old declarations from `pricing.go`**

Delete `providerCalculator` (from its doc comment through the closing brace) and the whole
`selectCalculator` function. Change nothing else in that file.

- [ ] **Step 3: Point `applyFees` at the narrow type**

In `fees.go`, change the parameter from `providerCalculator` to `feeCalculator`. The interface now
declares `fees` directly, so the type assertion is no longer needed:

```go
func applyFees(calc feeCalculator, usage Usage, body, requestBody []byte) Usage {
	extra := calc.fees(body, requestBody, usage)
	...
}
```

The interface's `fees` signature is the refiner form from Task 5, so `usage` is passed in.

Keep the nil-calculator guard in `llm_cost_v2.go` — `selectCalculator` still returns nil for an
unknown provider.

- [ ] **Step 4: Delete every `Normalize`**

Remove the `Normalize` method from all five calculator files, along with any type or helper that
becomes unused as a result. Two cautions:

- Bedrock's `Normalize` calls `AnthropicCalculator.Normalize` and `OpenAICalculator.Normalize` for
  shape dispatch. All of it goes; the shape sniffing was only ever used to read token counts, which
  the template now supplies. Bedrock's `fees` from Task 5 must not depend on it.
- `bedrockCacheWritesByTTL` must **survive** — Task 5's `fees` calls it. Deleting it would silently
  restore the lost TTL split bug.

- [ ] **Step 5: Verify nothing was silently lost**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/gateway/dev-policies/llm-cost-v2
GOWORK=off go build ./... && GOWORK=off go vet ./...
grep -c "func .*) Normalize(" calculator_*.go | grep -v ":0" || echo "  no Normalize methods remain"
grep -c "bedrockCacheWritesByTTL" calculator_bedrock.go
wc -l calculator_*.go
```

Expected: build and vet clean; no `Normalize` methods; `bedrockCacheWritesByTTL` still present; the
calculator files substantially shorter than the 996 lines recorded above.

- [ ] **Step 6: Run everything**

```bash
GOWORK=off go clean -testcache && GOWORK=off go test ./... -v 2>&1 | tail -25
```

Expected: every test passes, including all parity tests. **No cost may change.** If any parity
value moves, a field that `Normalize` was silently supplying has been lost — stop and report.

- [ ] **Step 7: Leave uncommitted and report**

Do not commit or stage. Report the before/after line counts and the exact `pricing.go` excision.

---

### Task 7: Assert coverage so a field cannot vanish again

The two lost fields were not caught by any test, because nothing checked that every field the
original reads is covered. This adds that check.

**Files:**
- Create: `gateway/dev-policies/llm-cost-v2/completeness_test.go`

- [ ] **Step 1: Enumerate what the original sets, per provider**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/gateway-controllers/policies/llm-cost
for f in openai anthropic gemini mistral bedrock; do
  echo "[$f]"
  awk '/^func \(c \*[A-Za-z]*Calculator\) Normalize/{g=1} g{print} g&&/^}$/{exit}' calculator_$f.go \
    | grep -oE "^\s+[A-Z][A-Za-z0-9]+: " | tr -d ' :' | sort -u | tr '\n' ' '
  echo
done
```

That list is the contract. Record it in your report.

- [ ] **Step 2: Write the coverage test**

Create `completeness_test.go`. For each provider, declare the fields the original sets, then assert
each is supplied by **either** the template **or** `fees()` — driving a real response through the
policy and checking the field is non-zero where the response populates it.

```go
package llmcostv2

import "testing"

// fieldSource records where a Usage field is expected to come from now.
type fieldSource int

const (
	fromTemplate fieldSource = iota
	fromFees
)

// coverage lists every Usage field each provider's original calculator set, and
// which layer now supplies it. A field added to the original without being
// placed here will fail this test.
var coverage = map[string]map[string]fieldSource{
	"openai": {
		"PromptTokens": fromTemplate, "CompletionTokens": fromTemplate,
		"TotalTokens": fromTemplate, "CachedReadTokens": fromTemplate,
		"ReasoningTokens": fromTemplate, "AudioInputTokens": fromTemplate,
		"AudioOutputTokens": fromTemplate, "ServiceTier": fromTemplate,
		"WebSearchRequests": fromFees, "SearchContextSize": fromFees,
	},
	// ... one entry per provider, from Step 1's output
}

func TestCoverage_EveryOriginalFieldHasASource(t *testing.T) {
	for provider, fields := range coverage {
		if len(fields) == 0 {
			t.Errorf("%s: no coverage declared", provider)
		}
		for field, src := range fields {
			if src != fromTemplate && src != fromFees {
				t.Errorf("%s.%s: no source", provider, field)
			}
		}
	}
}
```

Then add, per provider, a test that drives a response populating every listed field and asserts
each arrives. That is what actually catches a regression — the map alone only documents intent.

Gemini's `ServiceTier` and Bedrock's `CacheWrite1hrTokens` must both be present and marked
`fromFees`. Those are the two that were lost; this test is what would have caught them.

- [ ] **Step 3: Run it**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/gateway/dev-policies/llm-cost-v2
GOWORK=off go clean -testcache && GOWORK=off go test ./... -run Coverage -v 2>&1 | tail -20
```

Expected: pass. A failure means a field is still uncovered — report which.

- [ ] **Step 4: Leave uncommitted and report**

Do not commit or stage.

---

### Task 8: Re-verify parity and rebuild

- [ ] **Step 1: Full parity run**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/gateway/dev-policies/llm-cost-v2
GOWORK=off go clean -testcache && GOWORK=off go test ./... -v 2>&1 | grep -E "^(--- |ok|FAIL)" | tail -30
```

Every parity test must still produce the **same cost strings** as before this plan. Record them
alongside the values in Plan 3's task-5 report and confirm they match.

- [ ] **Step 2: Confirm Bedrock now prices**

Bedrock previously returned `not_calculated` because its model never resolved. With Tasks 1-4 it
should price. Compare against `llm-cost` for the same body: Plan 3's report records
`0.0000837000` for a Converse response with a payload-substituted model. It should now reach that
without the substitution.

If it does not match, report the two values — do not adjust the test.

- [ ] **Step 3: Everything else still green**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform
(cd sdk/ai/llmusage && GOWORK=off go clean -testcache && GOWORK=off go test ./... 2>&1 | grep -E "^(ok|FAIL)")
(cd gateway/gateway-controller && GOWORK=off go clean -testcache && GOWORK=off go test ./pkg/config/ ./pkg/utils/ 2>&1 | grep -E "^(ok|FAIL)")
(cd ../gateway-controllers/policies/llm-cost && GOWORK=off go clean -testcache && GOWORK=off go test ./... 2>&1 | grep -E "^(ok|FAIL)")
```

- [ ] **Step 4: Rebuild the gateway**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/gateway
make build 2>&1 | tail -20
```

Expected: exit 0, both images built, `llm-cost-v2` discovered. If it fails, capture the error and
which stage — do not improvise.

Afterwards, remove any binaries the build leaves at the repo root (`mock-platform-api`,
`webhook-listener`) — a previous run left 29 MB of them behind.

- [ ] **Step 5: Final state**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform
git status --short
git -C ../gateway-controllers status --short
```

Report both in full. The `POLICIES` repo must be clean; nothing may be committed or staged.

---

## Definition of done

- [ ] The library reads `location: pathParam`, percent-decoding what it extracts.
- [ ] Bedrock's model resolves from the URL, including percent-encoded ARNs.
- [ ] Bedrock has its own `fees()` carrying the per-TTL cache split, and its own `Adjust()`.
- [ ] Gemini's `trafficType` maps to `ServiceTier`.
- [ ] No `Normalize` method remains in any calculator; `bedrockCacheWritesByTTL` survives.
- [ ] `pricing.go`'s excision is documented.
- [ ] A coverage test asserts every field the original sets has a source.
- [ ] Every previously-passing parity cost is **unchanged**.
- [ ] `make build` exits 0.
- [ ] Nothing committed, nothing staged, in either repository.

## Still open after this plan

- **Header and query-parameter locations are unread.** The schema allows `location: header` and
  `queryParam`; the library ignores both and returns empty — a silent zero. Either implement them or
  make the library log when it meets a location it cannot handle. `remainingTokens` is declared as a
  header in every template and is the one field this affects today.
- **A template field can declare only one location.** Bedrock's four response shapes share the
  `/invoke` path, so shape selection cannot be declared. A fallback list per field would close this
  and would also retire the model-resolution chain hardcoded in the library.
- **The Anthropic aggregate-cache divergence** from Plan 3 is unresolved: `0.0072750000` versus
  `0.0061500000` when `cache_creation_input_tokens` is present without the per-TTL breakdown.
- **Bedrock binary event-stream responses** are still not decoded.
- **The closed field vocabulary** — a genuinely new billable dimension still needs the three-layer
  schema change.
