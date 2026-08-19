# Declarative Service-Tier Value Mapping — Implementation Plan (Plan 5)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let a provider template declare how its own service-tier strings map onto the gateway's billing tiers, so a new LLM provider with unfamiliar tier names needs a YAML edit rather than a Go change.

**Architecture:** `ExtractionIdentifier` gains an optional `valueMap` (`map[string]string`). The library applies it to a value it has just read: if the raw value is a key in the map, the mapped value replaces it; otherwise the raw value passes through untouched. `normalizeServiceTier` remains the final gate that reduces anything unrecognised to the standard tier, so the pricing layer still only ever sees `priority`, `flex`, `batch` or `""`. Templates without a `valueMap` behave exactly as they do today.

**Tech Stack:** Go 1.x, oapi-codegen v2.5.1, controller-gen, `sdk/ai/llmusage` nested module, `dev-policies/llm-cost-v2`.

## Global Constraints

- **Never run `git commit`, `git add`, or any committing command.** No task in this plan ends in a commit. This applies to subagents as strictly as to the main session. If you believe a commit is needed, stop and ask.
- **Never run `git checkout <file>` or `git restore <file>`.** A previous session destroyed a needed `go.work` entry this way. Back up with `cp` before editing anything you might need to revert.
- **`gateway-controllers/policies/llm-cost` (v1) must not be modified.** It is the parity reference.
- **Never modify these owner-held uncommitted files:** `gateway/build-manifest.yaml`, `gateway/configs/config.toml`, `gateway/configs/config-template.toml`, `gateway/configs/llm-pricing/model_prices.json`. Read them freely.
- **Comments must never describe a change, fix, or history.** Write comments that explain what the code does for a reader who has never seen the old version. No "now handles…", no "fixed…", no ticket references.
- **All Go verification runs with `GOWORK=off`.** `go.work` masks the module resolution the Docker build uses and has hidden a real defect before.
- **Run `go clean -testcache` before asserting that tests pass.** A cached `ok` has been misreported as a pass before.
- **Provider facts come only from official published specs.** Do not infer a provider's response values from existing gateway code, the pricing file, or another policy — that reasoning is circular. The verified enums for this plan are listed in Task 3 and were confirmed against vendor documentation.
- **Do not name or allude to any third-party pricing-data source** anywhere, including comments.
- **Parity baselines that must be unchanged at the end:** `0.0002700000`, `0.0061500000`, `0.0072750000`. `0.0000000000` must not appear for a priced model.

## Ordering rationale (read before starting)

The shipped default templates are **not** passed through to the runtime as raw YAML. `gateway-controller/pkg/utils/llm_provider_template_loader.go:138` unmarshals each file into the typed `api.LLMProviderTemplate` generated from `api/management-openapi.yaml`, then re-marshals it (line 149) for delivery over xDS. Plain `json.Unmarshal` is lenient, so an unknown key does **not** error — it is **silently dropped**.

Consequence: **the management OpenAPI type must gain `valueMap` before any template declares it**, or the key vanishes with no diagnostic. Task 1 therefore comes first. This is also why `fallbackIdentifiers` works today — `management-openapi.yaml:4212` already declares it.

`platform-api` is a separate, additional pruning layer that affects only templates authored through the management API, not the shipped defaults. It already drops `fallbackIdentifiers`, `cachedTokens`, `cacheWriteTokens`, `serviceTier` and `cacheAccounting` — a pre-existing gap from Plan 4, not something this plan introduces. Task 6 closes it.

## File Structure

| File | Responsibility in this plan |
|---|---|
| `gateway/gateway-controller/api/management-openapi.yaml` | Source of truth for the runtime template type; add `valueMap` |
| `gateway/gateway-controller/pkg/api/management/generated.go` | Regenerated, never hand-edited |
| `gateway/gateway-controller/pkg/config/llm_validator.go` | Reject malformed `valueMap` entries |
| `sdk/ai/llmusage/template.go` | `fieldSpec.ValueMap`, read it in `readFieldSpec` |
| `sdk/ai/llmusage/decode.go` | Apply the map to a value just read |
| `sdk/ai/llmusage/usage.go` | `normalizeServiceTier` stays as the closing gate |
| `gateway-controller/default-llm-provider-templates/*.yaml` | Declare each provider's tier vocabulary |
| `dev-policies/llm-cost-v2/calculator_gemini.go` | Delete the hardcoded trafficType switch |
| `kubernetes/gateway-operator/api/v1{,alpha1}/llmprovidertemplate_types.go` | CRD types for k8s-authored templates |
| `platform-api/…` | Close the management-API pruning layer |

---

### Task 1: Management OpenAPI gains `valueMap`

This is the runtime bottleneck. Nothing else in the plan has any effect until this lands.

**Files:**
- Modify: `api-platform/gateway/gateway-controller/api/management-openapi.yaml:4193-4219`
- Regenerate: `api-platform/gateway/gateway-controller/pkg/api/management/generated.go`

**Interfaces:**
- Produces: `api.ExtractionIdentifier.ValueMap *map[string]string` with JSON tag `valueMap`, consumed by Task 2 (validator) and relied on by Task 4 (templates surviving the loader).

- [ ] **Step 1: Confirm the current generated type has no ValueMap**

```bash
cd api-platform/gateway/gateway-controller
grep -n -A8 "type ExtractionIdentifier struct" pkg/api/management/generated.go
```

Expected: fields `FallbackIdentifiers`, `Identifier`, `Location` only.

- [ ] **Step 2: Add the property to the OpenAPI schema**

In `api/management-openapi.yaml`, directly after the `fallbackIdentifiers` block that ends at line 4219, add at the same indentation:

```yaml
        valueMap:
          type: object
          additionalProperties:
            type: string
          description: |
            Maps values this provider reports onto the vocabulary the gateway
            bills on. A reported value that is not a key is used unchanged.
            Used where providers name the same billing tier differently.
          example:
            ON_DEMAND_PRIORITY: priority
            ON_DEMAND_FLEX: flex
```

- [ ] **Step 3: Regenerate the server code**

```bash
cd api-platform/gateway/gateway-controller && make generate-server-code
```

- [ ] **Step 4: Verify the field appears and the module builds**

```bash
cd api-platform/gateway/gateway-controller
grep -n -A10 "type ExtractionIdentifier struct" pkg/api/management/generated.go
GOWORK=off go build ./...
```

Expected: `ValueMap *map[string]string \`json:"valueMap,omitempty"\`` present; build silent.

- [ ] **Step 5: Prove the loader now preserves the key**

Create `api-platform/gateway/gateway-controller/pkg/utils/valuemap_roundtrip_test.go`:

```go
package utils

import (
	"encoding/json"
	"testing"

	"gopkg.in/yaml.v3"

	api "github.com/wso2/api-platform/gateway/gateway-controller/pkg/api/management"
)

// The loader converts template YAML into the typed model and re-marshals it for
// delivery, so a key absent from the type is dropped without an error. This
// pins valueMap to that path.
func TestTemplateYAMLRoundTripKeepsValueMap(t *testing.T) {
	source := []byte(`
serviceTier:
  location: payload
  identifier: $.usageMetadata.trafficType
  valueMap:
    ON_DEMAND_PRIORITY: priority
    ON_DEMAND_FLEX: flex
`)

	var generic map[string]interface{}
	if err := yaml.Unmarshal(source, &generic); err != nil {
		t.Fatalf("yaml: %v", err)
	}
	asJSON, err := json.Marshal(generic)
	if err != nil {
		t.Fatalf("marshal: %v", err)
	}

	var spec struct {
		ServiceTier api.ExtractionIdentifier `json:"serviceTier"`
	}
	if err := json.Unmarshal(asJSON, &spec); err != nil {
		t.Fatalf("unmarshal: %v", err)
	}

	if spec.ServiceTier.ValueMap == nil {
		t.Fatal("valueMap was dropped by the typed model")
	}
	if got := (*spec.ServiceTier.ValueMap)["ON_DEMAND_PRIORITY"]; got != "priority" {
		t.Errorf("ON_DEMAND_PRIORITY = %q, want priority", got)
	}
}
```

The import alias and path are the ones already used by the loader itself (`pkg/utils/llm_provider_template_loader.go:29`), so they are correct as written.

- [ ] **Step 6: Run it**

```bash
cd api-platform/gateway/gateway-controller
GOWORK=off go clean -testcache
GOWORK=off go test ./pkg/utils/ -run TestTemplateYAMLRoundTripKeepsValueMap -v
```

Expected: PASS.

- [ ] **Step 7: Confirm nothing else broke**

```bash
cd api-platform/gateway/gateway-controller
GOWORK=off go test ./... 2>&1 | grep -v "^ok\|no test files" | head
```

Expected: no failures.

---

### Task 2: Validator rejects a malformed `valueMap`

**Files:**
- Modify: `api-platform/gateway/gateway-controller/pkg/config/llm_validator.go` — inside `validateExtractionIdentifier`, after the `FallbackIdentifiers` block ending at line 349
- Test: `api-platform/gateway/gateway-controller/pkg/config/llm_validator_test.go`

**Interfaces:**
- Consumes: `api.ExtractionIdentifier.ValueMap` from Task 1.
- Produces: validation errors at field path `<prefix>.valueMap[<key>]`.

- [ ] **Step 1: Write the failing test**

Append to `pkg/config/llm_validator_test.go`:

```go
func TestValidateExtractionIdentifier_ValueMapRejectsEmptyKey(t *testing.T) {
	v := &LLMValidator{}
	empty := ""
	id := api.ExtractionIdentifier{
		Location:   "payload",
		Identifier: "$.usage.service_tier",
		ValueMap:   &map[string]string{empty: "priority"},
	}

	errs := v.validateExtractionIdentifier("serviceTier", &id)

	if len(errs) == 0 {
		t.Fatal("expected an error for an empty valueMap key")
	}
}

func TestValidateExtractionIdentifier_ValueMapAcceptsEmptyTarget(t *testing.T) {
	v := &LLMValidator{}
	id := api.ExtractionIdentifier{
		Location:   "payload",
		Identifier: "$.usage.service_tier",
		ValueMap:   &map[string]string{"scale": ""},
	}

	errs := v.validateExtractionIdentifier("serviceTier", &id)

	if len(errs) != 0 {
		t.Fatalf("mapping to the standard tier is valid, got %v", errs)
	}
}
```

`validateExtractionIdentifier` is declared at `pkg/config/llm_validator.go:319` on `*LLMValidator`, and that file already imports the generated package as `api` (line 28), so no new import is needed. Confirm the parameter order before running, since the declaration spans several lines:

```bash
cd api-platform/gateway/gateway-controller
sed -n '319,325p' pkg/config/llm_validator.go
```

- [ ] **Step 2: Run to verify it fails**

```bash
cd api-platform/gateway/gateway-controller
GOWORK=off go clean -testcache
GOWORK=off go test ./pkg/config/ -run TestValidateExtractionIdentifier_ValueMap -v
```

Expected: `TestValidateExtractionIdentifier_ValueMapRejectsEmptyKey` FAILS ("expected an error"). The second test passes already.

- [ ] **Step 3: Implement the check**

In `validateExtractionIdentifier`, after the `FallbackIdentifiers` loop:

```go
	if identifier.ValueMap != nil {
		for key := range *identifier.ValueMap {
			if key == "" {
				errors = append(errors, ValidationError{
					Field:   fmt.Sprintf("%s.valueMap", fieldPrefix),
					Message: "valueMap key cannot be empty",
				})
			}
		}
	}
```

An empty target is deliberately allowed: it is how a template states that a reported tier carries no distinct rates.

- [ ] **Step 4: Run to verify both pass**

```bash
cd api-platform/gateway/gateway-controller
GOWORK=off go test ./pkg/config/ -run TestValidateExtractionIdentifier_ValueMap -v
```

Expected: both PASS.

- [ ] **Step 5: Full package check**

```bash
cd api-platform/gateway/gateway-controller
GOWORK=off go test ./pkg/config/ ./pkg/utils/ 2>&1 | tail -5
```

Expected: `ok`.

---

### Task 3: Library applies the map

**Files:**
- Modify: `api-platform/sdk/ai/llmusage/template.go:5-9` (`fieldSpec`), `:65-88` (`readFieldSpec`)
- Modify: `api-platform/sdk/ai/llmusage/decode.go` (`readString`)
- Test: `api-platform/sdk/ai/llmusage/template_test.go`, `api-platform/sdk/ai/llmusage/decode_test.go`

**Interfaces:**
- Consumes: template maps shaped as in `buildTemplate()` in `template_test.go`.
- Produces: `fieldSpec.ValueMap map[string]string`; `readString` returns the mapped value when the raw value is a key, the raw value otherwise. `normalizeServiceTier` is unchanged and still runs last.

**Verified provider vocabularies** (official docs; do not re-derive from code):

| Provider | Field | Documented values |
|---|---|---|
| OpenAI | `$.service_tier` | `auto`, `default`, `flex`, `scale`, `priority`, `fast` |
| Anthropic | `$.usage.service_tier` | `standard`, `priority`, `batch` |
| AWS Bedrock | `$.serviceTier.type` | `priority`, `default`, `flex`, `reserved` |
| Vertex / Gemini | `$.usageMetadata.trafficType` | `TRAFFIC_TYPE_UNSPECIFIED`, `ON_DEMAND`, `ON_DEMAND_PRIORITY`, `ON_DEMAND_FLEX`, `PROVISIONED_THROUGHPUT` |

- [ ] **Step 1: Write the failing tests**

Append to `template_test.go`:

```go
func TestReadFieldSpec_ReadsValueMap(t *testing.T) {
	source := map[string]interface{}{
		"serviceTier": map[string]interface{}{
			"location":   "payload",
			"identifier": "$.usageMetadata.trafficType",
			"valueMap": map[string]interface{}{
				"ON_DEMAND_PRIORITY": "priority",
				"ON_DEMAND_FLEX":     "flex",
			},
		},
	}

	spec, ok := readFieldSpec(source, "serviceTier")
	if !ok {
		t.Fatal("readFieldSpec reported no spec")
	}
	if spec.ValueMap["ON_DEMAND_PRIORITY"] != "priority" {
		t.Errorf("ON_DEMAND_PRIORITY = %q, want priority", spec.ValueMap["ON_DEMAND_PRIORITY"])
	}
	if spec.ValueMap["ON_DEMAND_FLEX"] != "flex" {
		t.Errorf("ON_DEMAND_FLEX = %q, want flex", spec.ValueMap["ON_DEMAND_FLEX"])
	}
}

func TestReadFieldSpec_AbsentValueMapIsNil(t *testing.T) {
	spec, ok := readFieldSpec(buildTemplate(), "serviceTier")
	if !ok {
		t.Fatal("readFieldSpec reported no spec")
	}
	if spec.ValueMap != nil {
		t.Errorf("ValueMap = %v, want nil", spec.ValueMap)
	}
}

func TestReadFieldSpec_MalformedValueMapEntriesIgnored(t *testing.T) {
	source := map[string]interface{}{
		"serviceTier": map[string]interface{}{
			"location":   "payload",
			"identifier": "$.service_tier",
			"valueMap": map[string]interface{}{
				"fast":  "priority",
				"bogus": 42,
			},
		},
	}

	spec, _ := readFieldSpec(source, "serviceTier")
	if spec.ValueMap["fast"] != "priority" {
		t.Errorf("fast = %q, want priority", spec.ValueMap["fast"])
	}
	if _, present := spec.ValueMap["bogus"]; present {
		t.Error("non-string value should be skipped")
	}
}
```

Append to `decode_test.go`:

```go
func TestExtractUsage_ValueMapTranslatesServiceTier(t *testing.T) {
	template := map[string]interface{}{
		"promptTokens": map[string]interface{}{
			"location": "payload", "identifier": "$.usageMetadata.promptTokenCount",
		},
		"completionTokens": map[string]interface{}{
			"location": "payload", "identifier": "$.usageMetadata.candidatesTokenCount",
		},
		"responseModel": map[string]interface{}{
			"location": "payload", "identifier": "$.modelVersion",
		},
		"serviceTier": map[string]interface{}{
			"location":   "payload",
			"identifier": "$.usageMetadata.trafficType",
			"valueMap": map[string]interface{}{
				"ON_DEMAND_PRIORITY": "priority",
				"ON_DEMAND_FLEX":     "flex",
			},
		},
	}

	body := []byte(`{
		"modelVersion": "gemini-2.0-flash",
		"usageMetadata": {
			"promptTokenCount": 100,
			"candidatesTokenCount": 20,
			"trafficType": "ON_DEMAND_PRIORITY"
		}
	}`)

	usage, err := extractUsage(template, body, nil, "/v1/models/gemini-2.0-flash:generateContent")
	if err != nil {
		t.Fatalf("extractUsage: %v", err)
	}
	if usage.ServiceTier != "priority" {
		t.Errorf("ServiceTier = %q, want priority", usage.ServiceTier)
	}
	if !usage.IsPriority {
		t.Error("IsPriority = false, want true")
	}
}

func TestExtractUsage_UnmappedTierFallsThroughToStandard(t *testing.T) {
	template := map[string]interface{}{
		"promptTokens": map[string]interface{}{
			"location": "payload", "identifier": "$.usageMetadata.promptTokenCount",
		},
		"serviceTier": map[string]interface{}{
			"location":   "payload",
			"identifier": "$.usageMetadata.trafficType",
			"valueMap": map[string]interface{}{
				"ON_DEMAND_PRIORITY": "priority",
			},
		},
	}

	body := []byte(`{"usageMetadata":{"promptTokenCount":100,"trafficType":"PROVISIONED_THROUGHPUT"}}`)

	usage, err := extractUsage(template, body, nil, "/v1/models/x:generateContent")
	if err != nil {
		t.Fatalf("extractUsage: %v", err)
	}
	if usage.ServiceTier != "" {
		t.Errorf("ServiceTier = %q, want empty", usage.ServiceTier)
	}
}

func TestExtractUsage_NoValueMapKeepsExistingBehaviour(t *testing.T) {
	template := map[string]interface{}{
		"promptTokens": map[string]interface{}{
			"location": "payload", "identifier": "$.usage.prompt_tokens",
		},
		"serviceTier": map[string]interface{}{
			"location": "payload", "identifier": "$.service_tier",
		},
	}

	body := []byte(`{"service_tier":"flex","usage":{"prompt_tokens":100}}`)

	usage, err := extractUsage(template, body, nil, "/v1/chat/completions")
	if err != nil {
		t.Fatalf("extractUsage: %v", err)
	}
	if usage.ServiceTier != "flex" {
		t.Errorf("ServiceTier = %q, want flex", usage.ServiceTier)
	}
}
```

- [ ] **Step 2: Run to verify they fail**

```bash
cd api-platform/sdk/ai/llmusage
GOWORK=off go clean -testcache
GOWORK=off go test -run "ValueMap|UnmappedTier|NoValueMap" -v ./... 2>&1 | head -30
```

Expected: compile failure on `spec.ValueMap` (field does not exist). That counts as the failing state.

- [ ] **Step 3: Add the field**

`template.go`, replace the `fieldSpec` struct:

```go
type fieldSpec struct {
	Location    string
	Identifier  string
	Identifiers []string

	// ValueMap translates values the provider reports into the vocabulary the
	// gateway works in. A value that is not a key is used as reported.
	ValueMap map[string]string
}
```

- [ ] **Step 4: Read it in `readFieldSpec`**

In `readFieldSpec`, after the `fallbackIdentifiers` loop and before the return:

```go
	var valueMap map[string]string
	if raw, ok := raw["valueMap"].(map[string]interface{}); ok {
		for key, value := range raw {
			mapped, ok := value.(string)
			if !ok || key == "" {
				continue
			}
			if valueMap == nil {
				valueMap = make(map[string]string, len(raw))
			}
			valueMap[key] = mapped
		}
	}

	return fieldSpec{
		Location:    location,
		Identifier:  identifier,
		Identifiers: identifiers,
		ValueMap:    valueMap,
	}, true
```

Delete the old single-line `return fieldSpec{...}, true`.

- [ ] **Step 5: Apply the map in `readString`**

Add this helper to `decode.go`:

```go
// mapValue translates a value the provider reported into the vocabulary the
// library reports. A value absent from the map is returned unchanged, so a
// field with no valueMap behaves as though the map were not there.
func mapValue(spec fieldSpec, raw string) string {
	if spec.ValueMap == nil || raw == "" {
		return raw
	}
	if mapped, ok := spec.ValueMap[raw]; ok {
		return mapped
	}
	return raw
}
```

Then change exactly one line in `readString` (`decode.go:162-164`). It currently reads:

```go
		if value != "" {
			return value
		}
```

Make it:

```go
		if value != "" {
			return mapValue(spec, value)
		}
```

Leave the two `return ""` paths and the `continue` untouched. Mapping the value at this point means a mapped value is returned even when it came from a fallback identifier, and an unmapped value still satisfies the `value != ""` check that drives fallback selection.

- [ ] **Step 6: Run the new tests**

```bash
cd api-platform/sdk/ai/llmusage
GOWORK=off go clean -testcache
GOWORK=off go test -run "ValueMap|UnmappedTier|NoValueMap" -v ./... 2>&1 | head -30
```

Expected: all PASS.

- [ ] **Step 7: Run the whole library suite**

```bash
cd api-platform/sdk/ai/llmusage
GOWORK=off go test ./... 2>&1 | tail -3
GOWORK=off go vet ./...
```

Expected: `ok`, vet silent. All pre-existing tests must still pass — in particular `TestNormalize_ServiceTierKeepsRateBearingTiers` and `TestExtractUsage_FallbackIdentifiersTryInOrder`.

---

### Task 4: Templates declare their tier vocabularies

**Files:**
- Modify: `api-platform/gateway/gateway-controller/default-llm-provider-templates/openai-template.yaml`
- Modify: `api-platform/gateway/gateway-controller/default-llm-provider-templates/anthropic-template.yaml`
- Modify: `api-platform/gateway/gateway-controller/default-llm-provider-templates/awsbedrock-template.yaml`
- Modify: `api-platform/gateway/gateway-controller/default-llm-provider-templates/gemini-template.yaml`

**Interfaces:**
- Consumes: `valueMap` support from Tasks 1 and 3.
- Produces: `gemini-template.yaml` gains a `serviceTier` field for the first time, which is what allows Task 5 to delete Go code.

Only values that select a different rate need mapping. Everything else is deliberately left unlisted so it falls through to the standard tier.

- [ ] **Step 1: Back up all four files**

```bash
cd api-platform/gateway/gateway-controller/default-llm-provider-templates
mkdir -p /tmp/p5-tpl-bak && cp openai-template.yaml anthropic-template.yaml awsbedrock-template.yaml gemini-template.yaml /tmp/p5-tpl-bak/
```

- [ ] **Step 2: OpenAI — map `fast` onto priority**

`openai-template.yaml` already has:

```yaml
  serviceTier:
    location: payload
    identifier: $.service_tier
```

Add beneath `identifier`, at the same indentation as `identifier`:

```yaml
    valueMap:
      priority: priority
      flex: flex
      fast: priority
```

`auto`, `default` and `scale` are intentionally unlisted — they have no distinct rate variants in the pricing data.

- [ ] **Step 3: Anthropic**

Under the existing `serviceTier` block in `anthropic-template.yaml`:

```yaml
    valueMap:
      priority: priority
      batch: batch
```

`standard` is unlisted.

- [ ] **Step 4: AWS Bedrock**

Under the existing `serviceTier` block in `awsbedrock-template.yaml`:

```yaml
    valueMap:
      priority: priority
      flex: flex
```

`default` and `reserved` are unlisted.

- [ ] **Step 5: Gemini — add the field**

`gemini-template.yaml` currently declares no `serviceTier`. Add it as a sibling of the other extraction fields, matching their indentation:

```yaml
  serviceTier:
    location: payload
    identifier: $.usageMetadata.trafficType
    valueMap:
      ON_DEMAND_PRIORITY: priority
      ON_DEMAND_FLEX: flex
```

`ON_DEMAND`, `PROVISIONED_THROUGHPUT` and `TRAFFIC_TYPE_UNSPECIFIED` are unlisted.

- [ ] **Step 6: Confirm all four parse and carry the map**

```bash
cd api-platform/gateway/gateway-controller/default-llm-provider-templates
for f in openai anthropic awsbedrock gemini; do
  printf "%-12s " "$f"
  python3 -c "
import yaml,sys
d=yaml.safe_load(open('$f-template.yaml'))
st=d.get('spec',d).get('serviceTier')
print('serviceTier:', 'MISSING' if st is None else st.get('identifier'), '| valueMap keys:', sorted((st or {}).get('valueMap',{}).keys()))
"
done
```

Expected: every provider prints an identifier and a non-empty key list.

- [ ] **Step 7: Confirm the shipped templates still load**

```bash
cd api-platform/gateway/gateway-controller
GOWORK=off go test ./pkg/utils/ ./pkg/config/ 2>&1 | tail -4
```

Expected: `ok`.

---

### Task 5: Delete the hardcoded Gemini tier switch

**Files:**
- Modify: `api-platform/gateway/dev-policies/llm-cost-v2/calculator_gemini.go` — the `switch m.TrafficType` at approximately lines 94-100 and the `current.ServiceTier = serviceTier` assignment at approximately line 150
- Test: `api-platform/gateway/dev-policies/llm-cost-v2/llm_cost_v2_test.go`

**Interfaces:**
- Consumes: `serviceTier` now supplied by `gemini-template.yaml` (Task 4) and carried into the policy's own `Usage` by `toPricingUsage` (`fees.go:20`, already maps `ServiceTier: u.ServiceTier`).
- Produces: no Go file in `llm-cost-v2` references `TrafficType`.

- [ ] **Step 1: Confirm the value now arrives from the template**

Append to `llm_cost_v2_test.go`, following the existing `loadShippedTemplate` / `newTestPolicy` / `runResponse` pattern used at line 71:

```go
// The Gemini tier is declared in the shipped template's valueMap, so a priority
// response must price above an otherwise identical standard-tier one.
func TestGeminiPriorityTierComesFromTemplate(t *testing.T) {
	loadShippedTemplate(t, "gemini", "gemini-template.yaml")
	p := newTestPolicy(t)

	standard := []byte(`{"modelVersion":"gemini-3-flash-preview","usageMetadata":{"promptTokenCount":100000,"candidatesTokenCount":1000,"trafficType":"ON_DEMAND"}}`)
	priority := []byte(`{"modelVersion":"gemini-3-flash-preview","usageMetadata":{"promptTokenCount":100000,"candidatesTokenCount":1000,"trafficType":"ON_DEMAND_PRIORITY"}}`)

	path := "/v1/models/gemini-3-flash-preview:generateContent"

	standardCost, standardStatus := runResponse(t, p, "gemini", standard, nil, path)
	priorityCost, priorityStatus := runResponse(t, p, "gemini", priority, nil, path)

	if standardStatus != costStatusCalculated || priorityStatus != costStatusCalculated {
		t.Fatalf("statuses = %q / %q, want both %q", standardStatus, priorityStatus, costStatusCalculated)
	}
	t.Logf("gemini standard = %s  priority = %s", standardCost, priorityCost)

	if standardCost == priorityCost {
		t.Errorf("priority cost %s equals standard cost %s; the tier is not reaching pricing",
			priorityCost, standardCost)
	}
}
```

The model choice matters and has been verified. Only 12 gemini/vertex entries carry a priority rate, and most common ones do not:

- `gemini-3-flash-preview` — input `5e-07`, priority `9e-07` — **use this one**
- `gemini-2.0-flash` — **no priority rate**; the two costs would match and the test would fail spuriously
- `gemini/gemini-2.5-pro` — priority equals standard (`1.25e-06`); also unusable as a discriminator

If the pricing data has moved on, re-check before changing the model:

```bash
cd api-platform/gateway/configs/llm-pricing
python3 -c "
import json
d=json.load(open('model_prices.json'))
for k,v in d.items():
    if isinstance(v,dict) and 'gemini' in k.lower() and 'input_cost_per_token_priority' in v:
        if v['input_cost_per_token'] != v['input_cost_per_token_priority']:
            print(k, v['input_cost_per_token'], '->', v['input_cost_per_token_priority'])
"
```

- [ ] **Step 2: Run it — it should pass before the deletion**

```bash
cd api-platform/gateway/dev-policies/llm-cost-v2
GOWORK=off go clean -testcache
GOWORK=off go test -run TestGeminiPriorityTierComesFromTemplate -v ./...
```

Expected: PASS. This proves the template path works **before** the Go path is removed, so a failure after deletion is unambiguous.

- [ ] **Step 3: Record the Gemini cost baseline**

```bash
cd api-platform/gateway/dev-policies/llm-cost-v2
GOWORK=off go test -run "Gemini" -v ./... 2>&1 | grep -oE "0\.[0-9]{10}" | sort -u
```

Write the values down. They must be identical in Step 6.

- [ ] **Step 4: Delete the switch**

In `calculator_gemini.go`, remove:

```go
	var serviceTier string
	switch m.TrafficType {
	case "ON_DEMAND_PRIORITY":
		serviceTier = "priority"
	case "ON_DEMAND_FLEX":
		serviceTier = "flex"
	}
```

and the assignment:

```go
	current.ServiceTier = serviceTier
```

Also remove the `TrafficType string \`json:"trafficType"\`` struct member and its comment block, since nothing reads it any more. Leave every other field of the anonymous struct alone.

- [ ] **Step 5: Confirm no references remain**

```bash
cd api-platform/gateway/dev-policies/llm-cost-v2
grep -rn "TrafficType\|trafficType" *.go || echo "  clean"
GOWORK=off go vet ./...
```

Expected: `clean`, vet silent.

- [ ] **Step 6: Confirm costs are unchanged**

```bash
cd api-platform/gateway/dev-policies/llm-cost-v2
GOWORK=off go clean -testcache
GOWORK=off go test ./... 2>&1 | tail -3
GOWORK=off go test -run "Gemini" -v ./... 2>&1 | grep -oE "0\.[0-9]{10}" | sort -u
```

Expected: `ok`, and the same values recorded in Step 3.

- [ ] **Step 7: Confirm the global parity baselines**

```bash
cd api-platform/gateway/dev-policies/llm-cost-v2
GOWORK=off go test -v ./... 2>&1 | grep -oE "0\.[0-9]{10}" | sort -u
```

Expected: includes `0.0002700000`, `0.0061500000`, `0.0072750000`. `0.0000000000` must not appear for a priced model.

---

### Task 6: CRD types and the platform-api layer

Templates authored through Kubernetes or the management API travel through mirrors of the schema that this plan has not yet touched. Both currently drop `valueMap`, and `platform-api` additionally drops `fallbackIdentifiers` and four other Plan 4 fields.

**Files:**
- Modify: `api-platform/kubernetes/gateway-operator/api/v1/llmprovidertemplate_types.go:25-39`
- Modify: `api-platform/kubernetes/gateway-operator/api/v1alpha1/llmprovidertemplate_types.go:25-39`
- Regenerate: `config/crd/bases/gateway.api-platform.wso2.com_llmprovidertemplates.yaml`, both `zz_generated.deepcopy.go`, and the helm CRD copy
- Modify: `api-platform/platform-api/internal/model/llm.go:22-25`
- Modify: `api-platform/platform-api/internal/utils/llm_provider_template_loader.go:170-177`
- Modify: `api-platform/platform-api/resources/openapi.yaml:6818-6836`
- Regenerate: `api-platform/platform-api/api/generated.go`

**Interfaces:**
- Consumes: the schema shape settled in Task 1. Field name `valueMap`, Go field `ValueMap`, type `map[string]string`.
- Produces: `valueMap` and `fallbackIdentifiers` survive both authoring paths.

Both CRD versions must be edited. `convertViaJSON` converts between them field-by-field through JSON, so a field present in only one version is lost on conversion.

- [ ] **Step 1: Add the field to CRD v1**

In `kubernetes/gateway-operator/api/v1/llmprovidertemplate_types.go`, inside `ExtractionIdentifier` after `FallbackIdentifiers`:

```go
	// ValueMap translates values the provider reports into the vocabulary the
	// gateway works in. A reported value that is not a key is used unchanged.
	// +optional
	ValueMap map[string]string `json:"valueMap,omitempty"`
```

- [ ] **Step 2: Add the identical field to CRD v1alpha1**

Same block, same comment, in `kubernetes/gateway-operator/api/v1alpha1/llmprovidertemplate_types.go`.

- [ ] **Step 3: Regenerate CRDs and deepcopy**

```bash
cd api-platform/kubernetes/gateway-operator
make generate
make manifests
```

- [ ] **Step 4: Verify both versions carry it and it reached the CRD YAML**

```bash
cd api-platform/kubernetes/gateway-operator
grep -c "ValueMap" api/v1/zz_generated.deepcopy.go api/v1alpha1/zz_generated.deepcopy.go
grep -c "valueMap" config/crd/bases/gateway.api-platform.wso2.com_llmprovidertemplates.yaml
GOWORK=off go build ./...
```

Expected: non-zero counts for all three; build silent.

- [ ] **Step 5: Sync the helm CRD copy**

```bash
cd api-platform/kubernetes
diff config/crd/bases/gateway.api-platform.wso2.com_llmprovidertemplates.yaml \
     ../kubernetes/helm/operator-helm-chart/crds/gateway.api-platform.wso2.com_llmprovidertemplates.yaml | head
```

If the operator Makefile does not already copy it, copy the regenerated file over the helm one. Confirm the path first with `ls helm/operator-helm-chart/crds/`.

- [ ] **Step 6: Extend the platform-api model**

`platform-api/internal/model/llm.go`, replace `ExtractionIdentifier`:

```go
type ExtractionIdentifier struct {
	Location            string            `json:"location" db:"-"`
	Identifier          string            `json:"identifier" db:"-"`
	FallbackIdentifiers []string          `json:"fallbackIdentifiers,omitempty" db:"-"`
	ValueMap            map[string]string `json:"valueMap,omitempty" db:"-"`
}
```

- [ ] **Step 7: Carry both fields through the mapper**

`platform-api/internal/utils/llm_provider_template_loader.go`, in `mapExtractionIdentifier` (line ~170), replace the return:

```go
	return &model.ExtractionIdentifier{
		Location:            in.Location,
		Identifier:          in.Identifier,
		FallbackIdentifiers: in.FallbackIdentifiers,
		ValueMap:            in.ValueMap,
	}
```

`extractionIdentifierYAML` at `internal/utils/llm_provider_template_loader.go:31` currently declares only `Location` and `Identifier`, so **both** members must be added — without them the mapper above has nothing to copy:

```go
type extractionIdentifierYAML struct {
	Location            string            `yaml:"location"`
	Identifier          string            `yaml:"identifier"`
	FallbackIdentifiers []string          `yaml:"fallbackIdentifiers"`
	ValueMap            map[string]string `yaml:"valueMap"`
}
```

- [ ] **Step 8: Extend the platform-api OpenAPI and regenerate**

In `platform-api/resources/openapi.yaml`, after `identifier` at line 6836, add — matching the Task 1 wording:

```yaml
        fallbackIdentifiers:
          type: array
          items:
            type: string
          description: |
            Additional locations to try when the primary identifier yields no
            value, in order.
        valueMap:
          type: object
          additionalProperties:
            type: string
          description: |
            Maps values this provider reports onto the vocabulary the gateway
            bills on. A reported value that is not a key is used unchanged.
```

Regenerate with the `generate` target (`platform-api/Makefile:104`). It requires `yq` on PATH — it preprocesses the spec into `resources/openapi_with_binding.yaml` before running oapi-codegen, so editing `resources/openapi.yaml` is correct and the intermediate file is generated:

```bash
cd api-platform/platform-api
command -v yq >/dev/null || echo "INSTALL yq FIRST — the generate target needs it"
make generate
```

- [ ] **Step 9: Verify platform-api**

```bash
cd api-platform/platform-api
grep -c "ValueMap" api/generated.go internal/model/llm.go
GOWORK=off go build ./...
GOWORK=off go clean -testcache
GOWORK=off go test ./... 2>&1 | grep -v "^ok\|no test files" | head
```

Expected: non-zero counts, build silent, no test failures.

---

### Task 7: End-to-end verification

**Files:** none modified. This task only runs checks.

- [ ] **Step 1: Every touched Go module**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform
for m in sdk/ai/llmusage gateway/dev-policies/llm-cost-v2 gateway/gateway-controller platform-api kubernetes/gateway-operator; do
  printf "=== %s\n" "$m"
  (cd "$m" && GOWORK=off go clean -testcache && GOWORK=off go vet ./... && GOWORK=off go test ./... 2>&1 | grep -v "no test files" | tail -3)
done
```

Expected: vet silent and `ok` everywhere.

- [ ] **Step 2: Parity baselines**

```bash
cd api-platform/gateway/dev-policies/llm-cost-v2
GOWORK=off go test -v ./... 2>&1 | grep -oE "0\.[0-9]{10}" | sort -u
```

Expected: `0.0002700000`, `0.0061500000`, `0.0072750000` all present; no `0.0000000000` for a priced model.

- [ ] **Step 3: Confirm `llm-cost` v1 was not touched**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/gateway-controllers
git status --porcelain policies/llm-cost/
```

Expected: no output.

- [ ] **Step 4: Confirm no owner-held file was modified**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform
git status --porcelain gateway/build-manifest.yaml gateway/configs/config.toml \
  gateway/configs/config-template.toml gateway/configs/llm-pricing/model_prices.json
```

Expected: unchanged from the state at the start of the plan. If any line appears that was not there before, restore it from a `cp` backup — never with `git checkout`.

- [ ] **Step 5: Confirm nothing was committed**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform && git log --oneline -1 && git diff --cached --stat | tail -1
cd /Users/irashperera/Documents/APIM/api-platform/dev/gateway-controllers && git log --oneline -1 && git diff --cached --stat | tail -1
```

Expected: the same HEAD as before the plan, and an empty staged diff in both repos.

- [ ] **Step 6: Rebuild the gateway runtime**

The Python policies clone `github.com/wso2/gateway-controllers` at build time and that clone intermittently stalls for 75 seconds from inside the container VM. It is environmental. Retry, and only treat it as a real failure if a Go compile error appears:

```bash
cd api-platform/gateway
for i in 1 2 3 4 5; do
  echo "=== attempt $i ==="
  if make build-gateway-runtime > /tmp/p5_build_$i.log 2>&1; then echo "SUCCESS"; break; fi
  if grep -qE "\.go:[0-9]+:[0-9]+:|undefined:|cannot use" /tmp/p5_build_$i.log; then
    echo "REAL COMPILE ERROR"; grep -nE "\.go:[0-9]+:[0-9]+:" /tmp/p5_build_$i.log | head; break
  fi
  echo "  network stall, retrying"
done
```

- [ ] **Step 7: Confirm the policy loads with no chain-build warnings**

After redeploying:

```bash
docker logs $(docker ps --format '{{.Names}}' | grep gateway-runtime) 2>&1 | grep -i "llm-cost-v2" | grep -iE "registered|pricing map loaded|does not implement"
```

Expected: `Registered policy llm-cost-v2`, `pricing map loaded`, and **no** `does not implement` lines.

- [ ] **Step 8: Report**

State plainly: which providers now declare a `valueMap`, that the Gemini switch is gone, that the parity baselines held, and which schema layers were updated. If any step was skipped or failed, say so explicitly rather than summarising around it.

---

## Execution notes

Recorded during execution; both were discovered rather than anticipated.

**`platform-api/api/generated.go` was deliberately left at its committed state.** Running `make generate` there is currently lossy: `resources/openapi.yaml` is an OpenAPI **3.1** document and oapi-codegen v2.5.1 warns it does not support 3.1. Regenerating changed 171 lines, only 14 of them related to this plan, and silently downgraded required fields to optional pointers — for example in `SecretCreateRequest` and `SecretUpdateRequest`:

```go
before:  Value  string  `binding:"required" json:"value"`
after:   Value *string  `binding:"required" json:"value,omitempty"`
```

The `binding:"required"` tag survives because the yq step adds it from the `required` list, but oapi-codegen itself no longer treats the field as required, so the Go type changes. That is a pre-existing mismatch between the committed generated file and the spec, unrelated to this plan. The spec addition in `resources/openapi.yaml` was kept, since the spec is the source of truth; the generated file was restored from a `cp` backup verified byte-identical to HEAD.

**Task 6's platform-api half was dropped entirely — it was over-scoped and could not have worked.** All four platform-api files were reverted to their committed state (`api/generated.go`, `resources/openapi.yaml`, `internal/model/llm.go`, `internal/utils/llm_provider_template_loader.go`).

The plan included platform-api on the reasoning that it is a fourth pruning layer for `ExtractionIdentifier`. That reasoning checked *what* `mapExtractionIdentifier` copies (only `Location` and `Identifier`) but never checked *which fields it is called on*. It is called on exactly six: `promptTokens`, `completionTokens`, `totalTokens`, `remainingTokens`, `requestModel`, `responseModel`. Its own `templateRuntimeArtifact` type — commented as "the subset of an LLM provider template that the gateway actually consumes" — lists the same six plus `resourceMappings`.

`serviceTier` is not among them, nor are `cachedTokens`, `cacheWriteTokens`, `cacheWrite1hTokens`, `reasoningTokens`, `audioInputTokens`, `audioOutputTokens` or `cacheAccounting`. platform-api discards `serviceTier` wholesale, so a `valueMap` hanging off it could never survive that layer no matter what the `ExtractionIdentifier` struct declares. Extending that struct was inert.

Anyone later wanting provider templates to round-trip through platform-api's HTTP API must first teach it the eight missing fields, and separately resolve the OpenAPI 3.1 codegen mismatch above. That is its own piece of work and is not a prerequisite for anything in this plan.

**Both images must be rebuilt, not just the runtime.** The provider templates ship only in the gateway-controller image at `/app/default-llm-provider-templates/`; the gateway-runtime image does not contain them. A `make build-gateway-runtime` alone leaves the old templates in place and the new `valueMap` keys never reach the policy. Run `make build-controller` as well, and redeploy both.

## Known limitations after this plan

- **The right-hand side of `valueMap` stays closed.** Targets must be `priority`, `flex`, `batch` or empty, because those select rate columns (`_priority`, `_flex`, `_batches`) in the pricing data. A provider with a genuinely new *billing* tier still needs new rate columns and Go to read them. `normalizeServiceTier` enforces this, so a template typo degrades to standard pricing rather than mispricing.
- **`valueMap` is generic but only `serviceTier` uses it.** `readString` applies it to any string field, so it would also work for model-name normalisation. No template does that yet.
- **OpenAI batch remains undetected.** OpenAI signals batch by endpoint, not by `service_tier`, so the `_batches` rate columns stay unreachable for that provider. Anthropic batch works.
- **The analytics system policy still does not read `valueMap`.** It has its own extractor that reads only `identifier`, so Moesif's tier-dependent fields are unaffected by this plan. Left untouched deliberately.
- **`Accumulate` still truncates for a second streaming policy.** Unrelated to this plan; see the Plan 5 discussion notes in `llm-usage-extraction-design.md`.
