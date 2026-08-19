# LLM Usage Extraction — Plan 2 of 3: The `llmusage` Extraction Library

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `sdk/ai/llmusage`, a shared library that reads an `LlmProviderTemplate` and a
provider's HTTP response and returns normalized token usage — so field paths live in template
YAML instead of hardcoded Go structs.

**Architecture:** A new nested Go module under `sdk/ai/llmusage` depending only on `sdk/core`.
It resolves the route's template from the policy SDK's lazy-resource store, selects the matching
`resourceMappings` entry by request path, reads each declared JSONPath out of the response, and
normalizes the result according to the template's `cacheAccounting` flag. Results are memoized in
`SharedContext` so several policies on one route extract once. No policy consumes it in this plan;
`llm-cost` migrates in Plan 3.

**Tech Stack:** Go 1.26.2, `sdk/core` v0.3.4 (policy SDK + `utils.ExtractStringValueFromJsonpath`),
standard library only otherwise.

**Reference spec:** `gateway/spec/llm-usage-extraction-design.md`, sections 3 and 6.
**Depends on:** Plan 1 (schema fields exist). Plan 1's changes are uncommitted in the working tree.

## Global Constraints

- **Never commit and never stage.** Leave all changes in the working tree; the repository owner
  commits. Do not run `git commit`, `git add`, `git stash`, `git checkout`, `git restore`, or
  `git reset`.
- The working tree carries unrelated uncommitted owner changes (`gateway/build.yaml`,
  `gateway/build-manifest.yaml`, `gateway/configs/config.toml`,
  `gateway/configs/config-template.toml`, `gateway/configs/llm-pricing/model_prices.json`,
  untracked `gateway/spec/*.md`) **plus all of Plan 1's uncommitted work**. Do not touch, revert,
  or stage any of it.
- The library reads the template as an **untyped `map[string]interface{}`**, because that is what
  `policy.LazyResource.Resource` holds. It must not import gateway-controller packages — those are
  not in the `sdk` module tree.
- `cacheAccounting` semantics: `"additive"` means the provider's cached/cache-write counts are
  **on top of** the input total; anything else (including absent) means `"inclusive"` — already
  part of it. At the `resourceMappings` level, absent means **inherit the template-level value**,
  never "default to inclusive".
- Per-field fallback: a `resourceMappings` entry overrides only the fields it sets. Every unset
  field falls back to the template root **individually**, not as a whole object.
- Comments: short and explanatory. NEVER write a comment describing a fix, a change, or history
  (no "fixed this", "added for X", "previously this did Y").
- No behaviour change to any existing policy, and no edit to any module outside
  `sdk/ai/llmusage`. It must not alter what `llm-cost` or `token-based-ratelimit` currently do.
- Every new exported symbol needs a doc comment. Every task ends with tests passing under
  `go clean -testcache`.

## File Structure

```
sdk/ai/llmusage/                     NEW nested module, depends only on sdk/core
  go.mod
  usage.go            Usage type + cacheAccounting normalization   (pure, no I/O)
  template.go         read field paths out of the template map     (pure)
  decode.go           SSE merge + Bedrock event-stream decoding    (pure)
  extract.go          Get / Accumulate — the public API, SharedContext-aware
  usage_test.go
  template_test.go
  decode_test.go
  extract_test.go

  pathmatch.go        NEW: pathsMatch + preferMoreSpecificPath (unexported)
  pathmatch_test.go   NEW
```

Nothing outside `sdk/ai/llmusage` is touched. In particular the gateway-controller keeps its own
copy of the path-matching rule — see Task 3 for why.

Why the split: `usage.go`, `template.go` and `decode.go` are pure functions testable with no SDK
context at all, which keeps the tricky logic cheap to test. Only `extract.go` touches
`SharedContext` and the lazy-resource store.

---

### Task 1: Module skeleton, `Usage` type, and cache normalization

**Files:**
- Create: `sdk/ai/llmusage/go.mod`
- Create: `sdk/ai/llmusage/usage.go`
- Create: `sdk/ai/llmusage/usage_test.go`
- Modify: `go.work` (register the new module)

**Interfaces:**
- Consumes: nothing.
- Produces: `type Usage struct` with the exact field names below, and
  `func normalize(raw rawCounts, accounting string) Usage`. Tasks 2 and 4 depend on both.

- [ ] **Step 1: Create the module and register it in the workspace**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform
mkdir -p sdk/ai/llmusage
cat > sdk/ai/llmusage/go.mod <<'EOF'
module github.com/wso2/api-platform/sdk/ai/llmusage

go 1.26.2

require github.com/wso2/api-platform/sdk/core v0.3.4
EOF
```

Then add `./sdk/ai/llmusage` to the `use (...)` block in `go.work`, keeping the list
alphabetically ordered (it belongs immediately after `./sdk/ai`).

- [ ] **Step 2: Write the failing test**

Create `sdk/ai/llmusage/usage_test.go`:

```go
package llmusage

import "testing"

func TestNormalize_InclusiveKeepsInputTotal(t *testing.T) {
	// OpenAI style: cached_tokens is already inside prompt_tokens.
	raw := rawCounts{InputTokens: 1000, OutputTokens: 200, CachedTokens: 800}

	got := normalize(raw, "inclusive")

	if got.TotalInputTokens != 1000 {
		t.Errorf("TotalInputTokens = %d, want 1000", got.TotalInputTokens)
	}
	if got.CachedReadTokens != 800 {
		t.Errorf("CachedReadTokens = %d, want 800", got.CachedReadTokens)
	}
	if got.UncachedInputTokens != 200 {
		t.Errorf("UncachedInputTokens = %d, want 200", got.UncachedInputTokens)
	}
}

func TestNormalize_AdditiveSumsIntoInputTotal(t *testing.T) {
	// Anthropic style: total input is input + cache_creation + cache_read.
	raw := rawCounts{
		InputTokens:        1000,
		OutputTokens:       200,
		CachedTokens:       500,
		CacheWriteTokens:   300,
		CacheWrite1hTokens: 100,
	}

	got := normalize(raw, "additive")

	if got.TotalInputTokens != 1900 {
		t.Errorf("TotalInputTokens = %d, want 1900", got.TotalInputTokens)
	}
	if got.UncachedInputTokens != 1000 {
		t.Errorf("UncachedInputTokens = %d, want 1000", got.UncachedInputTokens)
	}
}

func TestNormalize_AbsentAccountingIsInclusive(t *testing.T) {
	raw := rawCounts{InputTokens: 500, CachedTokens: 200}

	got := normalize(raw, "")

	if got.TotalInputTokens != 500 {
		t.Errorf("TotalInputTokens = %d, want 500", got.TotalInputTokens)
	}
}

func TestNormalize_UncachedNeverNegative(t *testing.T) {
	// A provider reporting more cached tokens than input must not produce a
	// negative billable count.
	raw := rawCounts{InputTokens: 100, CachedTokens: 500}

	got := normalize(raw, "inclusive")

	if got.UncachedInputTokens != 0 {
		t.Errorf("UncachedInputTokens = %d, want 0", got.UncachedInputTokens)
	}
}

func TestNormalize_TotalTokensPreferredWhenReported(t *testing.T) {
	raw := rawCounts{InputTokens: 100, OutputTokens: 50, TotalTokens: 175}

	got := normalize(raw, "inclusive")

	if got.TotalTokens != 175 {
		t.Errorf("TotalTokens = %d, want 175 (reported value, not the sum)", got.TotalTokens)
	}
}

func TestNormalize_TotalTokensDerivedWhenAbsent(t *testing.T) {
	raw := rawCounts{InputTokens: 100, OutputTokens: 50}

	got := normalize(raw, "inclusive")

	if got.TotalTokens != 150 {
		t.Errorf("TotalTokens = %d, want 150", got.TotalTokens)
	}
}

func TestNormalize_PriorityTierDetected(t *testing.T) {
	raw := rawCounts{InputTokens: 10, ServiceTier: "priority"}

	if got := normalize(raw, ""); !got.IsPriority {
		t.Error("IsPriority = false, want true for service_tier=priority")
	}
	if got := normalize(rawCounts{ServiceTier: "standard"}, ""); got.IsPriority {
		t.Error("IsPriority = true, want false for service_tier=standard")
	}
}
```

- [ ] **Step 3: Run the test to verify it fails**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/sdk/ai/llmusage
go test ./... 2>&1 | head -20
```

Expected: FAIL — `undefined: rawCounts`, `undefined: normalize`.

- [ ] **Step 4: Write the implementation**

Create `sdk/ai/llmusage/usage.go`:

```go
package llmusage

// Usage holds token counts normalized from a provider response, with the
// provider's cache accounting already resolved.
type Usage struct {
	// TotalInputTokens is every input token the request consumed, including
	// cached and cache-write tokens.
	TotalInputTokens int64

	// UncachedInputTokens is the subset of TotalInputTokens billed at the
	// standard input rate.
	UncachedInputTokens int64

	// CachedReadTokens is the subset billed at the cache-read rate.
	CachedReadTokens int64

	// CacheWriteTokens and CacheWrite1hTokens are the subsets billed at the
	// cache-creation rates. CacheWrite1hTokens covers the extended TTL.
	CacheWriteTokens   int64
	CacheWrite1hTokens int64

	OutputTokens    int64
	ReasoningTokens int64

	AudioInputTokens  int64
	AudioOutputTokens int64

	// TotalTokens is the provider's reported total when it gives one, and
	// input + output otherwise. Providers do not always agree with the sum.
	TotalTokens int64

	// IsPriority reports a priority service tier, which selects higher rates.
	IsPriority bool

	// Model is the model name resolved from the response or request.
	Model string

	// ModelCandidates lists the names considered, in the order they were
	// tried, so a consumer needing the fallback chain can see it.
	ModelCandidates []string
}

// rawCounts holds the values read straight out of a response, before cache
// accounting is applied.
type rawCounts struct {
	InputTokens        int64
	OutputTokens       int64
	TotalTokens        int64
	CachedTokens       int64
	CacheWriteTokens   int64
	CacheWrite1hTokens int64
	ReasoningTokens    int64
	AudioInputTokens   int64
	AudioOutputTokens  int64
	ServiceTier        string
	Model              string
	ModelCandidates    []string
}

// accountingAdditive is the cacheAccounting value meaning the cached and
// cache-write counts sit outside the reported input total.
const accountingAdditive = "additive"

// priorityServiceTier selects priority rates. Providers spell the standard
// tier variously ("default", "standard"); only this value is special.
const priorityServiceTier = "priority"

// normalize resolves cache accounting into a Usage. With "additive" the cache
// counts are added to the reported input total; otherwise they are already
// inside it and only the uncached remainder is derived.
func normalize(raw rawCounts, accounting string) Usage {
	u := Usage{
		OutputTokens:      raw.OutputTokens,
		CachedReadTokens:  raw.CachedTokens,
		ReasoningTokens:   raw.ReasoningTokens,
		AudioInputTokens:  raw.AudioInputTokens,
		AudioOutputTokens: raw.AudioOutputTokens,
		IsPriority:        raw.ServiceTier == priorityServiceTier,
		Model:             raw.Model,
		ModelCandidates:   raw.ModelCandidates,
	}

	u.CacheWriteTokens = raw.CacheWriteTokens
	u.CacheWrite1hTokens = raw.CacheWrite1hTokens

	if accounting == accountingAdditive {
		u.TotalInputTokens = raw.InputTokens + raw.CachedTokens +
			raw.CacheWriteTokens + raw.CacheWrite1hTokens
		u.UncachedInputTokens = raw.InputTokens
	} else {
		u.TotalInputTokens = raw.InputTokens
		u.UncachedInputTokens = raw.InputTokens - raw.CachedTokens -
			raw.CacheWriteTokens - raw.CacheWrite1hTokens
		if u.UncachedInputTokens < 0 {
			u.UncachedInputTokens = 0
		}
	}

	u.TotalTokens = raw.TotalTokens
	if u.TotalTokens == 0 {
		u.TotalTokens = u.TotalInputTokens + u.OutputTokens
	}

	return u
}
```

- [ ] **Step 5: Run the tests to verify they pass**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/sdk/ai/llmusage
go clean -testcache && go test ./... -v 2>&1 | tail -25
```

Expected: PASS, 7 tests.

- [ ] **Step 6: Confirm the new module is visible to the workspace**

The repo root has **no `go.mod`** — only `go.work` — so `go build ./...` from the root is invalid
and reports `directory prefix . does not contain modules listed in go.work`. Build the affected
modules explicitly instead:

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform
(cd sdk/ai/llmusage && go build ./...)
(cd sdk/core && go build ./...)
(cd sdk/ai && go build ./...)
```

Expected: no output from any of the three. A missing or misplaced `go.work` entry surfaces as a
resolution error in the first one.

Note: `event-gateway/gateway-controller` does not build at this commit — `cmd/controller/main.go`
calls `adminserver.NewServer` with three arguments while the committed signature takes four. Both
files are unmodified committed code, so this breakage pre-dates this work and is not ours to fix.
Do not attempt to repair it; it is only listed here so it is not mistaken for a regression.

- [ ] **Step 7: Leave uncommitted and report**

Do not commit or stage. Run `git status --short sdk/ ../go.work` and report the file list.

---

### Task 2: Read field paths out of the template map

**Files:**
- Create: `sdk/ai/llmusage/template.go`
- Create: `sdk/ai/llmusage/template_test.go`

**Interfaces:**
- Consumes: nothing from Task 1 directly.
- Produces:
  - `type fieldSpec struct { Location, Identifier string }`
  - `func resolveFields(template map[string]interface{}, requestPath string) (map[string]fieldSpec, string)` —
    returns the effective field specs keyed by template field name
    (`"promptTokens"`, `"cachedTokens"`, …) and the effective `cacheAccounting`.
  Task 4 calls `resolveFields`.

- [ ] **Step 1: Write the failing test**

Create `sdk/ai/llmusage/template_test.go`:

```go
package llmusage

import "testing"

// buildTemplate mirrors the shape the lazy-resource store holds: the template
// spec decoded into a generic map.
func buildTemplate() map[string]interface{} {
	return map[string]interface{}{
		"cacheAccounting": "additive",
		"promptTokens": map[string]interface{}{
			"location": "payload", "identifier": "$.usage.input_tokens",
		},
		"completionTokens": map[string]interface{}{
			"location": "payload", "identifier": "$.usage.output_tokens",
		},
		"serviceTier": map[string]interface{}{
			"location": "payload", "identifier": "$.usage.service_tier",
		},
		"resourceMappings": map[string]interface{}{
			"resources": []interface{}{
				map[string]interface{}{
					"resource": "/responses",
					"promptTokens": map[string]interface{}{
						"location": "payload", "identifier": "$.usage.prompt_tokens",
					},
					"cacheAccounting": "inclusive",
				},
				map[string]interface{}{
					"resource": "/v1/*",
					"completionTokens": map[string]interface{}{
						"location": "payload", "identifier": "$.usage.wildcard_tokens",
					},
				},
			},
		},
	}
}

func TestResolveFields_NoMappingUsesTemplateRoot(t *testing.T) {
	fields, accounting := resolveFields(buildTemplate(), "/chat/completions")

	if got := fields["promptTokens"].Identifier; got != "$.usage.input_tokens" {
		t.Errorf("promptTokens = %q, want the template root value", got)
	}
	if accounting != "additive" {
		t.Errorf("cacheAccounting = %q, want additive", accounting)
	}
}

func TestResolveFields_MappingOverridesOnlyItsOwnFields(t *testing.T) {
	fields, accounting := resolveFields(buildTemplate(), "/responses")

	if got := fields["promptTokens"].Identifier; got != "$.usage.prompt_tokens" {
		t.Errorf("promptTokens = %q, want the mapping override", got)
	}
	// completionTokens is not set on this mapping, so it must fall back
	// individually rather than being lost with the rest of the object.
	if got := fields["completionTokens"].Identifier; got != "$.usage.output_tokens" {
		t.Errorf("completionTokens = %q, want the inherited root value", got)
	}
	if got := fields["serviceTier"].Identifier; got != "$.usage.service_tier" {
		t.Errorf("serviceTier = %q, want the inherited root value", got)
	}
	if accounting != "inclusive" {
		t.Errorf("cacheAccounting = %q, want the mapping override", accounting)
	}
}

func TestResolveFields_MappingWithoutAccountingInheritsIt(t *testing.T) {
	// The /v1/* mapping sets no cacheAccounting, so the template-level value
	// must be inherited rather than defaulting to inclusive.
	_, accounting := resolveFields(buildTemplate(), "/v1/anything")

	if accounting != "additive" {
		t.Errorf("cacheAccounting = %q, want the inherited additive", accounting)
	}
}

func TestResolveFields_ExactMappingBeatsWildcard(t *testing.T) {
	tmpl := map[string]interface{}{
		"resourceMappings": map[string]interface{}{
			"resources": []interface{}{
				map[string]interface{}{
					"resource": "/v1/*",
					"promptTokens": map[string]interface{}{
						"location": "payload", "identifier": "$.wildcard",
					},
				},
				map[string]interface{}{
					"resource": "/v1/exact",
					"promptTokens": map[string]interface{}{
						"location": "payload", "identifier": "$.exact",
					},
				},
			},
		},
	}

	fields, _ := resolveFields(tmpl, "/v1/exact")

	if got := fields["promptTokens"].Identifier; got != "$.exact" {
		t.Errorf("promptTokens = %q, want the exact mapping to win over the wildcard", got)
	}
}

func TestResolveFields_MalformedEntriesIgnored(t *testing.T) {
	tmpl := map[string]interface{}{
		"promptTokens":     "not-an-object",
		"completionTokens": map[string]interface{}{"location": "payload"},
		"resourceMappings": "not-an-object",
	}

	fields, accounting := resolveFields(tmpl, "/chat/completions")

	if _, ok := fields["promptTokens"]; ok {
		t.Error("promptTokens should be absent when it is not an object")
	}
	if _, ok := fields["completionTokens"]; ok {
		t.Error("completionTokens should be absent when it has no identifier")
	}
	if accounting != "" {
		t.Errorf("cacheAccounting = %q, want empty", accounting)
	}
}

func TestResolveFields_NilTemplate(t *testing.T) {
	fields, accounting := resolveFields(nil, "/chat/completions")

	if len(fields) != 0 {
		t.Errorf("fields = %v, want empty", fields)
	}
	if accounting != "" {
		t.Errorf("cacheAccounting = %q, want empty", accounting)
	}
}
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/sdk/ai/llmusage
go test ./... -run ResolveFields 2>&1 | head -10
```

Expected: FAIL — `undefined: resolveFields`.

- [ ] **Step 3: Write the implementation**

Create `sdk/ai/llmusage/template.go`:

```go
package llmusage

import "github.com/wso2/api-platform/sdk/core/utils"

// fieldSpec is one extraction identifier from a template.
type fieldSpec struct {
	Location   string
	Identifier string
}

// extractionFields are the template keys this library reads. Names match the
// LlmProviderTemplate schema exactly.
var extractionFields = []string{
	"promptTokens",
	"completionTokens",
	"totalTokens",
	"cachedTokens",
	"cacheWriteTokens",
	"cacheWrite1hTokens",
	"reasoningTokens",
	"audioInputTokens",
	"audioOutputTokens",
	"serviceTier",
	"requestModel",
	"responseModel",
}

// resolveFields returns the effective extraction fields for a request path and
// the effective cache accounting mode. A resourceMappings entry matching the
// path overrides individual fields; every field it does not set falls back to
// the template root on its own.
func resolveFields(template map[string]interface{}, requestPath string) (map[string]fieldSpec, string) {
	fields := make(map[string]fieldSpec, len(extractionFields))
	if template == nil {
		return fields, ""
	}

	for _, name := range extractionFields {
		if spec, ok := readFieldSpec(template, name); ok {
			fields[name] = spec
		}
	}
	accounting, _ := template["cacheAccounting"].(string)

	mapping := selectMapping(template, requestPath)
	if mapping == nil {
		return fields, accounting
	}

	for _, name := range extractionFields {
		if spec, ok := readFieldSpec(mapping, name); ok {
			fields[name] = spec
		}
	}
	if override, ok := mapping["cacheAccounting"].(string); ok && override != "" {
		accounting = override
	}

	return fields, accounting
}

// readFieldSpec reads one extraction identifier. A field that is absent,
// malformed, or missing its identifier is treated as not set.
func readFieldSpec(source map[string]interface{}, name string) (fieldSpec, bool) {
	raw, ok := source[name].(map[string]interface{})
	if !ok {
		return fieldSpec{}, false
	}
	identifier, ok := raw["identifier"].(string)
	if !ok || identifier == "" {
		return fieldSpec{}, false
	}
	location, _ := raw["location"].(string)
	return fieldSpec{Location: location, Identifier: identifier}, true
}

// selectMapping returns the resourceMappings entry that best matches the
// request path, preferring an exact match over a wildcard and a longer pattern
// over a shorter one.
func selectMapping(template map[string]interface{}, requestPath string) map[string]interface{} {
	mappings, ok := template["resourceMappings"].(map[string]interface{})
	if !ok {
		return nil
	}
	resources, ok := mappings["resources"].([]interface{})
	if !ok {
		return nil
	}

	var selected map[string]interface{}
	var selectedPath string
	for _, entry := range resources {
		candidate, ok := entry.(map[string]interface{})
		if !ok {
			continue
		}
		candidatePath, ok := candidate["resource"].(string)
		if !ok || !pathsMatch(requestPath, candidatePath) {
			continue
		}
		if selected == nil || preferMoreSpecificPath(candidatePath, selectedPath) {
			selected, selectedPath = candidate, candidatePath
		}
	}

	return selected
}
```

`pathsMatch` and `preferMoreSpecificPath` do not exist yet — Task 3 creates them, in the same
package.
This task's tests will not compile until Task 3 lands, so implement Task 3 next and run both
task's tests together at the end of Task 3.

- [ ] **Step 4: Confirm the expected compile failure names the missing helpers**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/sdk/ai/llmusage
go test ./... 2>&1 | head -5
```

Expected: a compile error naming `pathsMatch` / `preferMoreSpecificPath` as undefined. Anything else means `template.go` has an unrelated mistake — fix that before moving on.

- [ ] **Step 5: Leave uncommitted and report**

Do not commit or stage. Report that Task 2's tests are blocked on Task 3 by design.

---

### Task 3: The path-matching primitives

The library needs the same rule the gateway-controller uses for matching a
`resourceMappings.resource` pattern against a request path. It gets its **own unexported copy**
inside `llmusage`, and the controller is left completely untouched.

An earlier revision of this plan instead moved the rule into `sdk/core/utils` and had the
controller delegate to it, so there would be one implementation. That was reverted, because it
breaks the controller's image build: the controller is compiled inside Docker where `sdk/core`
resolves to its pinned published tag (v0.2.18), which has no such file, and its `go.mod` carries no
replace. Fixing that would mean either publishing `sdk/core` first or forcing the controller from
v0.2.18 to v0.3.4 content — far too large a change for a local experiment.

The accepted cost is two small copies of a stable rule. Unify them when the library is published,
at which point it costs nothing.

**Files:**
- Create: `sdk/ai/llmusage/pathmatch.go`
- Create: `sdk/ai/llmusage/pathmatch_test.go`

**Interfaces:**
- Consumes: nothing.
- Produces: `func pathsMatch(requestPath, pattern string) bool` and
  `func preferMoreSpecificPath(candidate, current string) bool`, both unexported in package
  `llmusage`. Task 2 calls both.

- [ ] **Step 1: Write the failing test**

Create `sdk/ai/llmusage/pathmatch_test.go`:

```go
package llmusage

import "testing"

func TestPathsMatch(t *testing.T) {
	tests := []struct {
		name        string
		requestPath string
		pattern     string
		want        bool
	}{
		{"root wildcard matches anything", "/chat/completions", "/*", true},
		{"exact match", "/responses", "/responses", true},
		{"prefix wildcard matches", "/chat/completions", "/chat/*", true},
		{"prefix wildcard matches deeper", "/chat/completions/stream", "/chat/*", true},
		{"different path does not match", "/responses", "/chat/completions", false},
		{"specific pattern does not match wildcard request", "/chat/*", "/chat/completions", false},
		{"empty pattern does not match", "/responses", "", false},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			if got := pathsMatch(tt.requestPath, tt.pattern); got != tt.want {
				t.Errorf("pathsMatch(%q, %q) = %v, want %v", tt.requestPath, tt.pattern, got, tt.want)
			}
		})
	}
}

func TestPreferMoreSpecificPath(t *testing.T) {
	tests := []struct {
		name      string
		candidate string
		current   string
		want      bool
	}{
		{"exact beats wildcard", "/v1/exact", "/v1/*", true},
		{"wildcard loses to exact", "/v1/*", "/v1/exact", false},
		{"longer exact wins", "/v1/chat/completions", "/v1/chat", true},
		{"shorter exact loses", "/v1/chat", "/v1/chat/completions", false},
		{"longer wildcard wins over shorter wildcard", "/v1/chat/*", "/v1/*", true},
		{"equal length does not displace", "/v1/aaa", "/v1/bbb", false},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			if got := preferMoreSpecificPath(tt.candidate, tt.current); got != tt.want {
				t.Errorf("preferMoreSpecificPath(%q, %q) = %v, want %v",
					tt.candidate, tt.current, got, tt.want)
			}
		})
	}
}
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/sdk/ai/llmusage
go test ./... -run 'pathsMatch|preferMoreSpecificPath' 2>&1 | head -5
```

Expected: FAIL — `undefined: pathsMatch`.

- [ ] **Step 3: Write the implementation**

Create `sdk/ai/llmusage/pathmatch.go`:

```go
package llmusage

import "strings"

// pathsMatch reports whether a request path is covered by a route or resource
// pattern. A pattern may end in a wildcard to cover everything beneath a
// prefix. A specific pattern never matches a wildcard request path, so
// catch-all routes do not pick up resource-specific configuration.
func pathsMatch(requestPath, pattern string) bool {
	if pattern == "/*" {
		return true
	}
	if requestPath == pattern {
		return true
	}
	if strings.Contains(pattern, "*") {
		prefix := pattern[:strings.LastIndex(pattern, "*")]
		return strings.HasPrefix(requestPath, prefix)
	}
	return false
}

// preferMoreSpecificPath reports whether candidate is a better match than
// current: a pattern without a wildcard beats one with a wildcard, and a
// longer pattern beats a shorter one.
func preferMoreSpecificPath(candidate, current string) bool {
	candidateHasWildcard := strings.Contains(candidate, "*")
	currentHasWildcard := strings.Contains(current, "*")

	if !candidateHasWildcard && currentHasWildcard {
		return true
	}
	if candidateHasWildcard && !currentHasWildcard {
		return false
	}

	return len(candidate) > len(current)
}
```

- [ ] **Step 4: Run the tests to verify they pass**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/sdk/ai/llmusage
go clean -testcache && go test ./... -run 'pathsMatch|preferMoreSpecificPath' -v 2>&1 | tail -25
```

Expected: PASS, 13 subtests.

- [ ] **Step 5: Verify Task 2's tests now pass**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/sdk/ai/llmusage
go clean -testcache && go test ./... -v 2>&1 | tail -30
```

Expected: PASS — Task 1's 7 tests and Task 2's 6 tests.

- [ ] **Step 6: Leave uncommitted and report**

Do not commit or stage. Report the file list and both suites' results.

---

### Task 4: Read the response and produce a `Usage`

**Files:**
- Create: `sdk/ai/llmusage/decode.go`
- Create: `sdk/ai/llmusage/decode_test.go`

**Interfaces:**
- Consumes: `rawCounts` and `normalize` (Task 1), `resolveFields` and `fieldSpec` (Task 2).
- Produces:
  - `func decodeBody(body []byte, requestPath string) []byte`
  - `func extractUsage(template map[string]interface{}, body, requestBody []byte, requestPath string) (Usage, error)`
  Task 5 calls `extractUsage`.

- [ ] **Step 1: Write the failing test**

Create `sdk/ai/llmusage/decode_test.go`:

```go
package llmusage

import "testing"

func openAITemplate() map[string]interface{} {
	return map[string]interface{}{
		"promptTokens": map[string]interface{}{
			"location": "payload", "identifier": "$.usage.prompt_tokens",
		},
		"completionTokens": map[string]interface{}{
			"location": "payload", "identifier": "$.usage.completion_tokens",
		},
		"totalTokens": map[string]interface{}{
			"location": "payload", "identifier": "$.usage.total_tokens",
		},
		"cachedTokens": map[string]interface{}{
			"location": "payload", "identifier": "$.usage.prompt_tokens_details.cached_tokens",
		},
		"serviceTier": map[string]interface{}{
			"location": "payload", "identifier": "$.service_tier",
		},
		"responseModel": map[string]interface{}{
			"location": "payload", "identifier": "$.model",
		},
	}
}

func TestExtractUsage_BufferedResponse(t *testing.T) {
	body := []byte(`{
		"model": "gpt-4o-mini-2024-07-18",
		"service_tier": "default",
		"usage": {
			"prompt_tokens": 1000,
			"completion_tokens": 200,
			"total_tokens": 1200,
			"prompt_tokens_details": {"cached_tokens": 800}
		}
	}`)

	got, err := extractUsage(openAITemplate(), body, nil, "/chat/completions")
	if err != nil {
		t.Fatalf("extractUsage returned error: %v", err)
	}

	if got.TotalInputTokens != 1000 {
		t.Errorf("TotalInputTokens = %d, want 1000", got.TotalInputTokens)
	}
	if got.CachedReadTokens != 800 {
		t.Errorf("CachedReadTokens = %d, want 800", got.CachedReadTokens)
	}
	if got.UncachedInputTokens != 200 {
		t.Errorf("UncachedInputTokens = %d, want 200", got.UncachedInputTokens)
	}
	if got.OutputTokens != 200 {
		t.Errorf("OutputTokens = %d, want 200", got.OutputTokens)
	}
	if got.TotalTokens != 1200 {
		t.Errorf("TotalTokens = %d, want 1200", got.TotalTokens)
	}
	if got.Model != "gpt-4o-mini-2024-07-18" {
		t.Errorf("Model = %q, want the response model", got.Model)
	}
	if got.IsPriority {
		t.Error("IsPriority = true, want false for service_tier=default")
	}
}

func TestExtractUsage_ModelFallsBackToRequestBody(t *testing.T) {
	tmpl := openAITemplate()
	tmpl["requestModel"] = map[string]interface{}{
		"location": "payload", "identifier": "$.model",
	}

	// The responses API echoes no model, so the request body supplies it.
	body := []byte(`{"usage":{"prompt_tokens":10,"completion_tokens":5}}`)
	requestBody := []byte(`{"model":"my-deployment"}`)

	got, err := extractUsage(tmpl, body, requestBody, "/responses")
	if err != nil {
		t.Fatalf("extractUsage returned error: %v", err)
	}

	if got.Model != "my-deployment" {
		t.Errorf("Model = %q, want my-deployment from the request body", got.Model)
	}
	if len(got.ModelCandidates) == 0 {
		t.Error("ModelCandidates is empty, want the tried names recorded")
	}
}

func TestExtractUsage_SSEStreamMergesEvents(t *testing.T) {
	// The model arrives in the first event and usage only in the last, so the
	// merged view must carry both.
	body := []byte(`data: {"model":"gpt-4o-mini","choices":[{"delta":{"content":"hi"}}]}

data: {"usage":{"prompt_tokens":50,"completion_tokens":10,"total_tokens":60}}

data: [DONE]
`)

	got, err := extractUsage(openAITemplate(), body, nil, "/chat/completions")
	if err != nil {
		t.Fatalf("extractUsage returned error: %v", err)
	}

	if got.TotalInputTokens != 50 {
		t.Errorf("TotalInputTokens = %d, want 50", got.TotalInputTokens)
	}
	if got.Model != "gpt-4o-mini" {
		t.Errorf("Model = %q, want the model from the first event", got.Model)
	}
}

func TestExtractUsage_SSEEmptyStringDoesNotOverwrite(t *testing.T) {
	// A later event carrying an empty model must not erase an earlier real one.
	body := []byte(`data: {"model":"gpt-4o-mini"}

data: {"model":"","usage":{"prompt_tokens":5,"completion_tokens":1}}
`)

	got, err := extractUsage(openAITemplate(), body, nil, "/chat/completions")
	if err != nil {
		t.Fatalf("extractUsage returned error: %v", err)
	}

	if got.Model != "gpt-4o-mini" {
		t.Errorf("Model = %q, want the non-empty earlier value preserved", got.Model)
	}
}

func TestExtractUsage_NoUsageObject(t *testing.T) {
	body := []byte(`{"model":"gpt-4o-mini","choices":[]}`)

	got, err := extractUsage(openAITemplate(), body, nil, "/chat/completions")
	if err != nil {
		t.Fatalf("extractUsage returned error: %v", err)
	}

	if got.TotalInputTokens != 0 || got.OutputTokens != 0 {
		t.Errorf("expected zero counts, got in=%d out=%d", got.TotalInputTokens, got.OutputTokens)
	}
}

func TestExtractUsage_UnparseableBody(t *testing.T) {
	if _, err := extractUsage(openAITemplate(), []byte(`not json at all`), nil, "/x"); err == nil {
		t.Error("expected an error for an unparseable body")
	}
}

func TestExtractUsage_AnthropicAdditiveAccounting(t *testing.T) {
	tmpl := map[string]interface{}{
		"cacheAccounting": "additive",
		"promptTokens": map[string]interface{}{
			"location": "payload", "identifier": "$.usage.input_tokens",
		},
		"completionTokens": map[string]interface{}{
			"location": "payload", "identifier": "$.usage.output_tokens",
		},
		"cachedTokens": map[string]interface{}{
			"location": "payload", "identifier": "$.usage.cache_read_input_tokens",
		},
		"cacheWriteTokens": map[string]interface{}{
			"location": "payload", "identifier": "$.usage.cache_creation.ephemeral_5m_input_tokens",
		},
		"serviceTier": map[string]interface{}{
			"location": "payload", "identifier": "$.usage.service_tier",
		},
	}
	body := []byte(`{
		"usage": {
			"input_tokens": 1000,
			"output_tokens": 200,
			"cache_read_input_tokens": 500,
			"cache_creation": {"ephemeral_5m_input_tokens": 300},
			"service_tier": "priority"
		}
	}`)

	got, err := extractUsage(tmpl, body, nil, "/v1/messages")
	if err != nil {
		t.Fatalf("extractUsage returned error: %v", err)
	}

	if got.TotalInputTokens != 1800 {
		t.Errorf("TotalInputTokens = %d, want 1800 (1000+500+300)", got.TotalInputTokens)
	}
	if got.UncachedInputTokens != 1000 {
		t.Errorf("UncachedInputTokens = %d, want 1000", got.UncachedInputTokens)
	}
	if !got.IsPriority {
		t.Error("IsPriority = false, want true")
	}
}
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/sdk/ai/llmusage
go test ./... -run ExtractUsage 2>&1 | head -5
```

Expected: FAIL — `undefined: extractUsage`.

- [ ] **Step 3: Write the implementation**

Create `sdk/ai/llmusage/decode.go`:

```go
package llmusage

import (
	"bytes"
	"encoding/json"
	"strconv"
	"strings"

	"github.com/wso2/api-platform/sdk/core/utils"
)

// decodeBody turns a response body into a single JSON object suitable for
// JSONPath extraction. Server-sent event streams are merged so values spread
// across events — the model in an early event, usage in the last — end up in
// one view. Anything else is returned unchanged.
func decodeBody(body []byte, requestPath string) []byte {
	if isSSE(body) {
		if merged, ok := mergeSSEEvents(body); ok {
			return merged
		}
	}
	return body
}

// isSSE reports whether the body looks like a server-sent event stream.
func isSSE(body []byte) bool {
	trimmed := bytes.TrimLeft(body, " \t\r\n")
	return bytes.HasPrefix(trimmed, []byte("data:")) ||
		bytes.HasPrefix(trimmed, []byte("event:"))
}

// mergeSSEEvents shallow-merges every JSON event in a stream, later events
// winning. Empty strings never displace a value already seen, so a trailing
// event with an empty model cannot erase a real one.
func mergeSSEEvents(body []byte) ([]byte, bool) {
	merged := make(map[string]interface{})
	found := false

	for _, line := range bytes.Split(body, []byte("\n")) {
		line = bytes.TrimSpace(line)
		if !bytes.HasPrefix(line, []byte("data:")) {
			continue
		}
		payload := bytes.TrimSpace(bytes.TrimPrefix(line, []byte("data:")))
		if len(payload) == 0 || bytes.Equal(payload, []byte("[DONE]")) {
			continue
		}

		var event map[string]interface{}
		if err := json.Unmarshal(payload, &event); err != nil {
			continue
		}
		found = true
		for key, value := range event {
			if str, ok := value.(string); ok && str == "" {
				continue
			}
			merged[key] = value
		}
	}

	if !found {
		return nil, false
	}
	out, err := json.Marshal(merged)
	if err != nil {
		return nil, false
	}
	return out, true
}

// extractUsage reads every field the template declares out of the response and
// normalizes the result. A field whose path is absent from the response
// contributes zero; that is not an error, since providers omit fields routinely.
func extractUsage(template map[string]interface{}, body, requestBody []byte, requestPath string) (Usage, error) {
	fields, accounting := resolveFields(template, requestPath)
	decoded := decodeBody(body, requestPath)

	// Confirm the body is usable before reading fields, so a malformed
	// response is reported rather than silently yielding zeros.
	var probe interface{}
	if err := json.Unmarshal(decoded, &probe); err != nil {
		return Usage{}, err
	}

	raw := rawCounts{
		InputTokens:        readInt(decoded, fields, "promptTokens"),
		OutputTokens:       readInt(decoded, fields, "completionTokens"),
		TotalTokens:        readInt(decoded, fields, "totalTokens"),
		CachedTokens:       readInt(decoded, fields, "cachedTokens"),
		CacheWriteTokens:   readInt(decoded, fields, "cacheWriteTokens"),
		CacheWrite1hTokens: readInt(decoded, fields, "cacheWrite1hTokens"),
		ReasoningTokens:    readInt(decoded, fields, "reasoningTokens"),
		AudioInputTokens:   readInt(decoded, fields, "audioInputTokens"),
		AudioOutputTokens:  readInt(decoded, fields, "audioOutputTokens"),
		ServiceTier:        readString(decoded, fields, "serviceTier"),
	}

	raw.Model, raw.ModelCandidates = resolveModel(decoded, requestBody, fields)

	return normalize(raw, accounting), nil
}

// resolveModel prefers the model the response reports, falling back to the one
// the request asked for. Both candidates are returned in the order tried.
func resolveModel(body, requestBody []byte, fields map[string]fieldSpec) (string, []string) {
	var candidates []string

	if name := readString(body, fields, "responseModel"); name != "" {
		candidates = append(candidates, name)
	}
	if len(requestBody) > 0 {
		if name := readString(requestBody, fields, "requestModel"); name != "" {
			candidates = append(candidates, name)
		}
	}

	if len(candidates) == 0 {
		return "", nil
	}
	return candidates[0], candidates
}

// readInt reads a declared field as an integer, returning zero when the field
// is not declared or the path is absent from the payload.
func readInt(payload []byte, fields map[string]fieldSpec, name string) int64 {
	value := readString(payload, fields, name)
	if value == "" {
		return 0
	}
	parsed, err := strconv.ParseFloat(value, 64)
	if err != nil {
		return 0
	}
	return int64(parsed)
}

// readString reads a declared field as a string. Only payload fields are read
// here; header and path locations are resolved by the caller that owns them.
func readString(payload []byte, fields map[string]fieldSpec, name string) string {
	spec, ok := fields[name]
	if !ok || spec.Location != locationPayload {
		return ""
	}
	value, err := utils.ExtractStringValueFromJsonpath(payload, spec.Identifier)
	if err != nil {
		return ""
	}
	return strings.TrimSpace(value)
}

// locationPayload is the ExtractionIdentifier location for a response body.
const locationPayload = "payload"
```

- [ ] **Step 4: Run the tests to verify they pass**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/sdk/ai/llmusage
go clean -testcache && go test ./... -v 2>&1 | tail -35
```

Expected: PASS — Task 1's 7, Task 2's 6, and Task 4's 7 tests.

- [ ] **Step 5: Leave uncommitted and report**

Do not commit or stage. Report the file list and the suite result.

---

### Task 5: The public API — `Get` and `Accumulate`

**Files:**
- Create: `sdk/ai/llmusage/extract.go`
- Create: `sdk/ai/llmusage/extract_test.go`

**Interfaces:**
- Consumes: `extractUsage` (Task 4).
- Produces the library's entire public surface:
  - `func Get(sc *policy.SharedContext, body, requestBody []byte, requestPath string) (Usage, error)`
  - `func Accumulate(sc *policy.SharedContext, chunk *policy.StreamBody) []byte`
  - `func Publish(sc *policy.SharedContext, u Usage)`
  - `const StatusKey`, `UsageKey`, and the status values.
  Plan 3's `llm-cost` migration calls `Get`, `Accumulate` and `Publish`.

- [ ] **Step 1: Write the failing test**

Create `sdk/ai/llmusage/extract_test.go`:

```go
package llmusage

import (
	"testing"

	policy "github.com/wso2/api-platform/sdk/core/policy/v1alpha2"
)

func newSharedContext() *policy.SharedContext {
	return &policy.SharedContext{Metadata: map[string]interface{}{}}
}

func TestAccumulate_DedupesByChunkIndex(t *testing.T) {
	sc := newSharedContext()

	// Two policies see the same chunk, so it must be appended once.
	first := &policy.StreamBody{Chunk: []byte("abc"), Index: 0}
	Accumulate(sc, first)
	Accumulate(sc, first)

	got := Accumulate(sc, &policy.StreamBody{Chunk: []byte("def"), Index: 1})

	if string(got) != "abcdef" {
		t.Errorf("accumulated = %q, want %q", got, "abcdef")
	}
}

func TestAccumulate_OutOfOrderIndexIgnored(t *testing.T) {
	sc := newSharedContext()

	Accumulate(sc, &policy.StreamBody{Chunk: []byte("abc"), Index: 0})
	Accumulate(sc, &policy.StreamBody{Chunk: []byte("def"), Index: 1})
	got := Accumulate(sc, &policy.StreamBody{Chunk: []byte("STALE"), Index: 0})

	if string(got) != "abcdef" {
		t.Errorf("accumulated = %q, want the stale chunk ignored", got)
	}
}

func TestAccumulate_NilContextDoesNotPanic(t *testing.T) {
	if got := Accumulate(nil, &policy.StreamBody{Chunk: []byte("abc")}); got != nil {
		t.Errorf("got %q, want nil for a nil context", got)
	}
}

func TestGet_MemoizesAcrossCalls(t *testing.T) {
	sc := newSharedContext()
	sc.Metadata[MetadataTemplateHandle] = "openai"
	storeTestTemplate(t, "openai", openAITemplate())

	body := []byte(`{"model":"gpt-4o-mini","usage":{"prompt_tokens":10,"completion_tokens":5}}`)

	first, err := Get(sc, body, nil, "/chat/completions")
	if err != nil {
		t.Fatalf("first Get returned error: %v", err)
	}

	// A second call with a different body must return the memoized value,
	// proving extraction happened once for the request.
	second, err := Get(sc, []byte(`{"usage":{"prompt_tokens":999}}`), nil, "/chat/completions")
	if err != nil {
		t.Fatalf("second Get returned error: %v", err)
	}

	if second.TotalInputTokens != first.TotalInputTokens {
		t.Errorf("second call re-extracted: got %d, want the memoized %d",
			second.TotalInputTokens, first.TotalInputTokens)
	}
	if first.TotalInputTokens != 10 {
		t.Errorf("TotalInputTokens = %d, want 10", first.TotalInputTokens)
	}
}

func TestGet_NoTemplateHandleReportsStatus(t *testing.T) {
	sc := newSharedContext()

	_, err := Get(sc, []byte(`{"usage":{"prompt_tokens":10}}`), nil, "/chat/completions")

	if err == nil {
		t.Fatal("expected an error when the route carries no template handle")
	}
	if sc.Metadata[StatusKey] != StatusTemplateMissing {
		t.Errorf("status = %v, want %q", sc.Metadata[StatusKey], StatusTemplateMissing)
	}
}

func TestPublish_WritesUsageAndStatus(t *testing.T) {
	sc := newSharedContext()

	Publish(sc, Usage{TotalInputTokens: 100, OutputTokens: 20, TotalTokens: 120, Model: "m"})

	if sc.Metadata[StatusKey] != StatusExtracted {
		t.Errorf("status = %v, want %q", sc.Metadata[StatusKey], StatusExtracted)
	}
	published, ok := sc.Metadata[UsageKey].(Usage)
	if !ok {
		t.Fatalf("usage under %q is %T, want Usage", UsageKey, sc.Metadata[UsageKey])
	}
	if published.TotalInputTokens != 100 {
		t.Errorf("published TotalInputTokens = %d, want 100", published.TotalInputTokens)
	}
}

func TestPublish_NilContextDoesNotPanic(t *testing.T) {
	Publish(nil, Usage{TotalInputTokens: 1})
}
```

Add this helper at the end of the same file — it seeds the lazy-resource store the library reads:

```go
// storeTestTemplate puts a template into the shared lazy-resource store and
// removes it when the test ends, so tests do not leak state into each other.
func storeTestTemplate(t *testing.T, handle string, spec map[string]interface{}) {
	t.Helper()

	store := policy.GetLazyResourceStoreInstance()
	if err := store.StoreResource(&policy.LazyResource{
		ID:           handle,
		ResourceType: ResourceTypeLLMProviderTemplate,
		Resource:     spec,
	}); err != nil {
		t.Fatalf("failed to store test template: %v", err)
	}
	t.Cleanup(func() {
		_ = store.RemoveResourceByIDAndType(handle, ResourceTypeLLMProviderTemplate)
	})
}
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/sdk/ai/llmusage
go test ./... -run 'Accumulate|TestGet|Publish' 2>&1 | head -8
```

Expected: FAIL — `undefined: Accumulate`, `undefined: Get`, `undefined: Publish`.

- [ ] **Step 3: Write the implementation**

Create `sdk/ai/llmusage/extract.go`:

```go
// Package llmusage extracts normalized LLM token usage from a provider
// response, using the field locations declared in the route's
// LlmProviderTemplate rather than per-provider Go code. Results are memoized
// in SharedContext so several policies on one route extract once.
package llmusage

import (
	"errors"

	policy "github.com/wso2/api-platform/sdk/core/policy/v1alpha2"
)

// Metadata keys the library reads from and writes to SharedContext.
const (
	// MetadataTemplateHandle is set by the kernel and names the route's template.
	MetadataTemplateHandle = "template_handle"

	// UsageKey holds the extracted Usage. The version suffix allows the shape
	// to change without breaking existing readers.
	UsageKey = "llm:usage:v1"

	// StatusKey reports whether extraction succeeded, so a consumer can tell
	// a genuinely free request from a failed extraction.
	StatusKey = "llm:usage:status"
)

// Status values written to StatusKey.
const (
	StatusExtracted       = "extracted"
	StatusTemplateMissing = "template_missing"
	StatusNoUsage         = "no_usage"
)

// ResourceTypeLLMProviderTemplate is the lazy-resource type holding templates.
const ResourceTypeLLMProviderTemplate = "LlmProviderTemplate"

// streamAccumKey and streamIndexKey hold per-request streaming state. Both are
// removed by Accumulate at end of stream.
const (
	streamAccumKey = "llm:usage:stream-accum"
	streamIndexKey = "llm:usage:stream-index"
)

// ErrNoTemplate reports that the route carries no resolvable template, so
// there are no field locations to extract from.
var ErrNoTemplate = errors.New("llmusage: no template for route")

// Get returns normalized usage for this request. The first call extracts and
// stores the result; later calls return the stored value, so several policies
// on one route pay for extraction once.
func Get(sc *policy.SharedContext, body, requestBody []byte, requestPath string) (Usage, error) {
	if sc == nil {
		return Usage{}, ErrNoTemplate
	}
	if sc.Metadata == nil {
		sc.Metadata = make(map[string]interface{})
	}

	if cached, ok := sc.Metadata[UsageKey].(Usage); ok {
		return cached, nil
	}

	template, err := templateForRoute(sc)
	if err != nil {
		sc.Metadata[StatusKey] = StatusTemplateMissing
		return Usage{}, err
	}

	usage, err := extractUsage(template, body, requestBody, requestPath)
	if err != nil {
		sc.Metadata[StatusKey] = StatusNoUsage
		return Usage{}, err
	}

	Publish(sc, usage)
	return usage, nil
}

// Publish stores usage and its status in SharedContext for other policies.
func Publish(sc *policy.SharedContext, u Usage) {
	if sc == nil {
		return
	}
	if sc.Metadata == nil {
		sc.Metadata = make(map[string]interface{})
	}

	sc.Metadata[UsageKey] = u
	if u.TotalInputTokens == 0 && u.OutputTokens == 0 {
		sc.Metadata[StatusKey] = StatusNoUsage
		return
	}
	sc.Metadata[StatusKey] = StatusExtracted
}

// Accumulate appends a stream chunk to the shared buffer and returns the buffer
// so far. Chunks are deduped by index, so several policies accumulating the
// same stream produce one buffer rather than one each. At end of stream the
// buffer is returned and the state removed.
func Accumulate(sc *policy.SharedContext, chunk *policy.StreamBody) []byte {
	if sc == nil || chunk == nil {
		return nil
	}
	if sc.Metadata == nil {
		sc.Metadata = make(map[string]interface{})
	}

	buffered, _ := sc.Metadata[streamAccumKey].([]byte)
	lastIndex, seen := sc.Metadata[streamIndexKey].(uint64)

	isNew := !seen || chunk.Index > lastIndex
	if isNew && len(chunk.Chunk) > 0 {
		buffered = append(buffered, chunk.Chunk...)
		sc.Metadata[streamAccumKey] = buffered
	}
	if isNew {
		sc.Metadata[streamIndexKey] = chunk.Index
	}

	if chunk.EndOfStream {
		delete(sc.Metadata, streamAccumKey)
		delete(sc.Metadata, streamIndexKey)
	}

	return buffered
}

// templateForRoute resolves the route's template from the lazy-resource store
// using the handle the kernel placed in SharedContext.
func templateForRoute(sc *policy.SharedContext) (map[string]interface{}, error) {
	handle, ok := sc.Metadata[MetadataTemplateHandle].(string)
	if !ok || handle == "" {
		return nil, ErrNoTemplate
	}

	resource, err := policy.GetLazyResourceStoreInstance().
		GetResourceByIDAndType(handle, ResourceTypeLLMProviderTemplate)
	if err != nil || resource == nil {
		return nil, ErrNoTemplate
	}

	spec, ok := resource.Resource["spec"].(map[string]interface{})
	if ok {
		return spec, nil
	}
	return resource.Resource, nil
}
```

Note the `spec` unwrap at the end: the lazy resource may hold either the whole template document
(with the fields under `spec`) or the spec alone, depending on the publisher. Handling both keeps
the library working either way. Confirm which shape arrives in practice during Plan 3's live
verification and simplify then if only one occurs.

- [ ] **Step 4: Run the tests to verify they pass**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/sdk/ai/llmusage
go clean -testcache && go test ./... -v 2>&1 | tail -40
```

Expected: PASS — all 27 tests across the four test files.

- [ ] **Step 5: Verify nothing else in the workspace broke**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform
(cd sdk/ai/llmusage && go build ./...)
(cd sdk/core && go build ./...)
(cd gateway/gateway-controller && go build ./...)
(cd gateway/gateway-controller && go clean -testcache && go test ./pkg/utils/ ./pkg/config/ 2>&1 | grep -E "^(ok|FAIL)")
(cd sdk/core && go clean -testcache && go test ./utils/ 2>&1 | grep -E "^(ok|FAIL)")
```

Expected: no build output; `ok` for all three suites. Do not run `go build ./...` from the repo
root — there is no `go.mod` there, only `go.work`, so it fails for reasons unrelated to this work.

- [ ] **Step 6: Leave uncommitted and report**

Do not commit or stage. Run `git status --short` and report the full file list, confirming nothing
outside `sdk/ai/llmusage/` changed.

---

## Definition of done

- [ ] `sdk/ai/llmusage` builds as its own module and depends only on `sdk/core`.
- [ ] `./sdk/ai/llmusage` is registered in `go.work`.
- [ ] `go clean -testcache && go test ./...` passes in `sdk/ai/llmusage` and `sdk/core`.
- [ ] `gateway-controller` is **unmodified**, and its `pkg/utils` / `pkg/config` suites still pass
      with `GOWORK=off` (the way the Docker build compiles).
- [ ] `sdk/ai/llmusage`, `sdk/core`, `sdk/ai` and `gateway/gateway-controller` each build from
      inside their own module directory. (`go build ./...` from the repo root is not a valid
      check — the root has no `go.mod`.)
- [ ] No policy behaviour changed: `llm-cost` and `token-based-ratelimit` are untouched.
- [ ] Nothing committed, nothing staged.

## What this plan deliberately excludes

- No `llm-cost` changes. Deleting its parsing and calling this library is Plan 3.
- No template YAML content changes. Those land per provider in Plan 3.
- No Bedrock binary event-stream decoding. Only SSE merging is implemented here, because Bedrock
  is the last provider to migrate in Plan 3 and its framing is better added alongside that
  migration, with its fixtures to test against. `decodeBody` takes `requestPath` precisely so
  that path-specific decoding can be added without changing any caller.
- No `azure-llm-cost` changes.
- No fee logic (web search, grounding, audio-minute conversion). Those stay in `llm-cost` by
  design — see the design spec's section 4.
