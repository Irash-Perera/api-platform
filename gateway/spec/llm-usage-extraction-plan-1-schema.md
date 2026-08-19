# LLM Usage Extraction — Plan 1 of 3: Template Schema Extension

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extend `LlmProviderTemplate` so a template can declare the token-usage fields the
`llm-cost` policy needs, without changing any runtime behaviour yet.

**Architecture:** Seven new optional `ExtractionIdentifier` fields plus one string enum
(`cacheAccounting`) are added to the template schema. The schema exists at three independent
layers that each prune unknown fields — the management OpenAPI spec (code-generated into Go),
the operator CRD types (v1 and v1alpha1), and the deploy-time validator. All three must change
together or a field added to a YAML is silently dropped. This plan changes only the schema and
its validation; no extraction logic and no template YAML content changes here.

**Tech Stack:** Go 1.26.2, oapi-codegen v2.5.1, controller-gen (kubebuilder), Kubernetes CRDs
(apiextensions v1, structural schema).

**Reference spec:** `gateway/spec/llm-usage-extraction-design.md`, sections 5 and 7.

## Global Constraints

- Every new field is **optional**. Existing templates must remain valid with zero edits.
- `cacheAccounting` valid values are exactly `inclusive` and `additive`. Absent means `inclusive`.
- Field names, casing and JSON tags must match **exactly** between the OpenAPI spec, `v1`, and
  `v1alpha1`, because `convertViaJSON` converts by JSON tag.
- Do **not** hand-edit generated files: `pkg/api/management/generated.go`,
  `api/*/zz_generated.deepcopy.go`, `config/crd/bases/*.yaml`. Regenerate them.
- Do **not** modify `platform-api` — its artifact `spec` is a generic object and needs no change.
- After `make manifests`, the helm CRD copies must be refreshed; they are byte-identical copies.
- Comments: keep them short and explanatory. Never write comments describing a fix, a change,
  or history (no "fixed this", "added for X", "previously this did Y").
- Nothing in this plan may alter cost output. `llm-cost` is untouched.
- **Never commit and never stage.** Leave all changes in the working tree; the repository owner
  commits. Do not run `git commit`, `git add`, `git stash`, or `git checkout` on any path.
- The working tree carries unrelated uncommitted changes in `gateway/build.yaml`,
  `gateway/build-manifest.yaml`, `gateway/configs/config.toml`,
  `gateway/configs/config-template.toml` and `gateway/configs/llm-pricing/model_prices.json`.
  Do not touch, revert, or stage them.

---

### Task 1: Add the new fields to the management OpenAPI spec

**Files:**
- Modify: `gateway/gateway-controller/api/management-openapi.yaml:4104-4118` (add to `LLMProviderTemplateData`)
- Modify: `gateway/gateway-controller/api/management-openapi.yaml:4137-4148` (add to `LLMProviderTemplateResourceMapping`)
- Regenerate: `gateway/gateway-controller/pkg/api/management/generated.go`

**Interfaces:**
- Consumes: nothing (first task).
- Produces: `api.LLMProviderTemplateData` and `api.LLMProviderTemplateResourceMapping` each gain
  seven `*ExtractionIdentifier` fields — `CachedTokens`, `CacheWriteTokens`, `CacheWrite1hTokens`,
  `ReasoningTokens`, `AudioInputTokens`, `AudioOutputTokens`, `ServiceTier` — plus
  `CacheAccounting *string`. Tasks 2 and 4 depend on these exact names.

- [ ] **Step 1: Add the seven identifier fields plus the flag to `LLMProviderTemplateData`**

In `api/management-openapi.yaml`, immediately after the existing `responseModel` line (4114-4115)
and **before** the `resourceMappings` key, insert:

```yaml
        cachedTokens:
          $ref: '#/components/schemas/ExtractionIdentifier'
        cacheWriteTokens:
          $ref: '#/components/schemas/ExtractionIdentifier'
        cacheWrite1hTokens:
          $ref: '#/components/schemas/ExtractionIdentifier'
        reasoningTokens:
          $ref: '#/components/schemas/ExtractionIdentifier'
        audioInputTokens:
          $ref: '#/components/schemas/ExtractionIdentifier'
        audioOutputTokens:
          $ref: '#/components/schemas/ExtractionIdentifier'
        serviceTier:
          $ref: '#/components/schemas/ExtractionIdentifier'
        cacheAccounting:
          type: string
          description: |
            Whether the cached token count reported by the provider is already
            part of the input token total, or additional to it. One of
            'inclusive' or 'additive'. Defaults to inclusive when omitted.
          example: inclusive
```

Deliberately **no `enum:` here.** oapi-codegen turns an inline enum into a named type — the
existing `location` enum produced `ExtractionIdentifierLocation string` — and it would mint a
distinct type per parent struct.

To be precise about the trade: that is an inconvenience, not a blocker. A shared
`validateCacheAccounting(fieldPrefix string, value *string)` helper would still work with a
one-token cast at each call site, `(*string)(spec.CacheAccounting)`. What we are really choosing
is a looser published OpenAPI contract in exchange for not minting two named types for one flag.

It is safe because both enforcement points exist either way: the CRD carries
`+kubebuilder:validation:Enum=inclusive;additive` at all four sites, and `validateCacheAccounting`
runs on every gateway ingress path — `parseAndValidateLLMTemplate` is reached by the management
API, by `InitializeOOBTemplates`, and by the immutable loader. Accepted values are byte-identical
on both sides (case-sensitive, untrimmed), so no value is accepted by one layer and rejected by
the other.

- [ ] **Step 2: Add the same fields to `LLMProviderTemplateResourceMapping`**

In the same file, after the mapping's existing `responseModel` line (4147-4148), insert the
identical block shown in Step 1. Both levels must carry the fields so a per-resource mapping can
override them.

- [ ] **Step 3: Regenerate the Go models**

```bash
cd gateway/gateway-controller
make generate-server-code
```

- [ ] **Step 4: Verify the generated struct fields exist with the expected names**

```bash
cd gateway/gateway-controller
grep -nE "CachedTokens|CacheWriteTokens|CacheWrite1hTokens|ReasoningTokens|AudioInputTokens|AudioOutputTokens|ServiceTier|CacheAccounting" \
  pkg/api/management/generated.go
```

Expected: each name appears at least twice — once in `LLMProviderTemplateData`, once in
`LLMProviderTemplateResourceMapping`. `CacheAccounting` should be typed `*string`, the rest
`*ExtractionIdentifier`.

- [ ] **Step 5: Verify the package still builds**

```bash
cd gateway/gateway-controller && go build ./...
```

Expected: no output, exit 0.

- [ ] **Step 6: Commit**

```bash
git add gateway/gateway-controller/api/management-openapi.yaml \
        gateway/gateway-controller/pkg/api/management/generated.go
git commit -m "feat(gateway): add usage detail fields to LlmProviderTemplate schema"
```

---

### Task 2: Validate the new fields at deploy time

**Files:**
- Modify: `gateway/gateway-controller/pkg/config/llm_validator.go:160-180` (template spec validation)
- Modify: `gateway/gateway-controller/pkg/config/llm_validator.go` (`validateTemplateResourceMapping`)
- Test: `gateway/gateway-controller/pkg/config/llm_validator_additional_test.go`

**Interfaces:**
- Consumes: the struct fields produced by Task 1.
- Produces: `validateCacheAccounting(fieldPrefix string, value *string) []ValidationError`. No later
  task calls it directly; it is wired into the two existing validate functions.

- [ ] **Step 1: Write the failing tests**

Append to `pkg/config/llm_validator_additional_test.go`:

```go
func TestLLMValidator_ValidateTemplateSpec_NewUsageFields_Valid(t *testing.T) {
	v := NewLLMValidator()
	accounting := "additive"
	spec := &api.LLMProviderTemplateData{
		DisplayName: "anthropic",
		CachedTokens: &api.ExtractionIdentifier{
			Location:   api.Payload,
			Identifier: "$.usage.cache_read_input_tokens",
		},
		CacheWriteTokens: &api.ExtractionIdentifier{
			Location:   api.Payload,
			Identifier: "$.usage.cache_creation.ephemeral_5m_input_tokens",
		},
		ServiceTier: &api.ExtractionIdentifier{
			Location:   api.Payload,
			Identifier: "$.usage.service_tier",
		},
		CacheAccounting: &accounting,
	}

	errs := v.validateTemplateSpec(spec)
	if len(errs) != 0 {
		t.Fatalf("expected no validation errors, got %v", errs)
	}
}

func TestLLMValidator_ValidateTemplateSpec_NewUsageFields_MissingIdentifier(t *testing.T) {
	v := NewLLMValidator()
	spec := &api.LLMProviderTemplateData{
		DisplayName: "openai",
		CachedTokens: &api.ExtractionIdentifier{
			Location:   api.Payload,
			Identifier: "",
		},
	}

	errs := v.validateTemplateSpec(spec)
	if len(errs) != 1 {
		t.Fatalf("expected 1 validation error, got %d: %v", len(errs), errs)
	}
	if errs[0].Field != "spec.cachedTokens.identifier" {
		t.Errorf("expected field spec.cachedTokens.identifier, got %s", errs[0].Field)
	}
}

func TestLLMValidator_ValidateTemplateSpec_CacheAccounting_Invalid(t *testing.T) {
	v := NewLLMValidator()
	accounting := "sometimes"
	spec := &api.LLMProviderTemplateData{
		DisplayName:     "openai",
		CacheAccounting: &accounting,
	}

	errs := v.validateTemplateSpec(spec)
	if len(errs) != 1 {
		t.Fatalf("expected 1 validation error, got %d: %v", len(errs), errs)
	}
	if errs[0].Field != "spec.cacheAccounting" {
		t.Errorf("expected field spec.cacheAccounting, got %s", errs[0].Field)
	}
}

func TestLLMValidator_ValidateTemplateSpec_CacheAccounting_OmittedIsValid(t *testing.T) {
	v := NewLLMValidator()
	spec := &api.LLMProviderTemplateData{DisplayName: "openai"}

	errs := v.validateTemplateSpec(spec)
	if len(errs) != 0 {
		t.Fatalf("expected no validation errors, got %v", errs)
	}
}

func TestLLMValidator_ValidateTemplateResourceMapping_NewUsageFields(t *testing.T) {
	v := NewLLMValidator()
	accounting := "inclusive"
	mapping := &api.LLMProviderTemplateResourceMapping{
		Resource: "/responses",
		CachedTokens: &api.ExtractionIdentifier{
			Location:   api.Payload,
			Identifier: "$.usage.input_tokens_details.cached_tokens",
		},
		CacheAccounting: &accounting,
	}

	errs := v.validateTemplateResourceMapping("spec.resourceMappings.resources[0]", mapping)
	if len(errs) != 0 {
		t.Fatalf("expected no validation errors, got %v", errs)
	}
}
```

- [ ] **Step 2: Run the tests to verify they fail**

```bash
cd gateway/gateway-controller
go test ./pkg/config/ -run 'NewUsageFields|CacheAccounting' -v
```

Expected: FAIL. The `_Valid`, `_OmittedIsValid` and `_NewUsageFields` mapping tests pass
vacuously, but `_MissingIdentifier` and `_Invalid` fail with "expected 1 validation error, got 0"
because nothing validates the new fields yet.

- [ ] **Step 3: Add the cacheAccounting validator**

Add to `pkg/config/llm_validator.go`, immediately after `validateExtractionIdentifier` (ends line 273):

```go
// validateCacheAccounting checks the cache accounting mode. An absent value is
// valid and means inclusive.
func (v *LLMValidator) validateCacheAccounting(fieldPrefix string, value *string) []ValidationError {
	if value == nil {
		return nil
	}

	if *value != "inclusive" && *value != "additive" {
		return []ValidationError{{
			Field:   fieldPrefix,
			Message: "cacheAccounting must be 'inclusive' or 'additive'",
		}}
	}

	return nil
}
```

- [ ] **Step 4: Wire the new fields into template spec validation**

In `validateTemplateSpec`, immediately before the existing `if spec.ResourceMappings != nil {`
block (line 176), insert:

```go
	if spec.CachedTokens != nil {
		errors = append(errors, v.validateExtractionIdentifier("spec.cachedTokens", spec.CachedTokens)...)
	}

	if spec.CacheWriteTokens != nil {
		errors = append(errors, v.validateExtractionIdentifier("spec.cacheWriteTokens", spec.CacheWriteTokens)...)
	}

	if spec.CacheWrite1hTokens != nil {
		errors = append(errors, v.validateExtractionIdentifier("spec.cacheWrite1hTokens", spec.CacheWrite1hTokens)...)
	}

	if spec.ReasoningTokens != nil {
		errors = append(errors, v.validateExtractionIdentifier("spec.reasoningTokens", spec.ReasoningTokens)...)
	}

	if spec.AudioInputTokens != nil {
		errors = append(errors, v.validateExtractionIdentifier("spec.audioInputTokens", spec.AudioInputTokens)...)
	}

	if spec.AudioOutputTokens != nil {
		errors = append(errors, v.validateExtractionIdentifier("spec.audioOutputTokens", spec.AudioOutputTokens)...)
	}

	if spec.ServiceTier != nil {
		errors = append(errors, v.validateExtractionIdentifier("spec.serviceTier", spec.ServiceTier)...)
	}

	errors = append(errors, v.validateCacheAccounting("spec.cacheAccounting", spec.CacheAccounting)...)
```

- [ ] **Step 5: Wire the same fields into resource-mapping validation**

In `validateTemplateResourceMapping`, after the last existing `validateExtractionIdentifier`
call for that function, insert the same seven blocks and the `validateCacheAccounting` call,
replacing the `"spec."` prefix with `fieldPrefix`:

```go
	if mapping.CachedTokens != nil {
		errors = append(errors, v.validateExtractionIdentifier(
			fmt.Sprintf("%s.cachedTokens", fieldPrefix), mapping.CachedTokens)...)
	}

	if mapping.CacheWriteTokens != nil {
		errors = append(errors, v.validateExtractionIdentifier(
			fmt.Sprintf("%s.cacheWriteTokens", fieldPrefix), mapping.CacheWriteTokens)...)
	}

	if mapping.CacheWrite1hTokens != nil {
		errors = append(errors, v.validateExtractionIdentifier(
			fmt.Sprintf("%s.cacheWrite1hTokens", fieldPrefix), mapping.CacheWrite1hTokens)...)
	}

	if mapping.ReasoningTokens != nil {
		errors = append(errors, v.validateExtractionIdentifier(
			fmt.Sprintf("%s.reasoningTokens", fieldPrefix), mapping.ReasoningTokens)...)
	}

	if mapping.AudioInputTokens != nil {
		errors = append(errors, v.validateExtractionIdentifier(
			fmt.Sprintf("%s.audioInputTokens", fieldPrefix), mapping.AudioInputTokens)...)
	}

	if mapping.AudioOutputTokens != nil {
		errors = append(errors, v.validateExtractionIdentifier(
			fmt.Sprintf("%s.audioOutputTokens", fieldPrefix), mapping.AudioOutputTokens)...)
	}

	if mapping.ServiceTier != nil {
		errors = append(errors, v.validateExtractionIdentifier(
			fmt.Sprintf("%s.serviceTier", fieldPrefix), mapping.ServiceTier)...)
	}

	errors = append(errors, v.validateCacheAccounting(
		fmt.Sprintf("%s.cacheAccounting", fieldPrefix), mapping.CacheAccounting)...)
```

- [ ] **Step 6: Run the new tests to verify they pass**

```bash
cd gateway/gateway-controller
go test ./pkg/config/ -run 'NewUsageFields|CacheAccounting' -v
```

Expected: PASS, 5 tests.

- [ ] **Step 7: Run the whole config package to confirm nothing regressed**

```bash
cd gateway/gateway-controller && go test ./pkg/config/
```

Expected: `ok`.

- [ ] **Step 8: Leave the changes uncommitted and report**

Do not commit or stage. Confirm the only files you modified are the two this task names:

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform
git status --short gateway/gateway-controller/pkg/config/
```

Expected: exactly two ` M` entries — `llm_validator.go` and `llm_validator_additional_test.go`.
Report the paths and the test output; the repository owner commits.

---

### Task 3: Add the fields to the operator CRD types

**Files:**
- Modify: `kubernetes/gateway-operator/api/v1/llmprovidertemplate_types.go:36-85`
- Modify: `kubernetes/gateway-operator/api/v1alpha1/llmprovidertemplate_types.go`
- Regenerate: `api/v1/zz_generated.deepcopy.go`, `api/v1alpha1/zz_generated.deepcopy.go`
- Regenerate: `config/crd/bases/gateway.api-platform.wso2.com_llmprovidertemplates.yaml`
- Copy: `kubernetes/helm/operator-helm-chart/crds/gateway.api-platform.wso2.com_llmprovidertemplates.yaml`

**Interfaces:**
- Consumes: the field names and JSON tags established in Task 1. They must match exactly, because
  `v1alpha1.LlmProviderTemplate.ConvertTo/ConvertFrom` convert via JSON.
- Produces: a CRD that accepts the new fields. Plan 3 relies on this to deploy updated templates.

- [ ] **Step 1: Add the fields to the v1 mapping struct**

In `api/v1/llmprovidertemplate_types.go`, inside `LLMProviderTemplateResourceMapping` (after the
existing `TotalTokens` field, line 54), insert:

```go
	// +optional
	AudioInputTokens *ExtractionIdentifier `json:"audioInputTokens,omitempty"`
	// +optional
	AudioOutputTokens *ExtractionIdentifier `json:"audioOutputTokens,omitempty"`
	// +optional
	CachedTokens *ExtractionIdentifier `json:"cachedTokens,omitempty"`
	// +optional
	CacheWrite1hTokens *ExtractionIdentifier `json:"cacheWrite1hTokens,omitempty"`
	// +optional
	CacheWriteTokens *ExtractionIdentifier `json:"cacheWriteTokens,omitempty"`
	// +optional
	ReasoningTokens *ExtractionIdentifier `json:"reasoningTokens,omitempty"`
	// +optional
	ServiceTier *ExtractionIdentifier `json:"serviceTier,omitempty"`

	// CacheAccounting states whether the provider's cached token count is part
	// of the input total or additional to it. Defaults to inclusive.
	// +optional
	// +kubebuilder:validation:Enum=inclusive;additive
	CacheAccounting string `json:"cacheAccounting,omitempty"`
```

- [ ] **Step 2: Add the same fields to the v1 spec struct**

In the same file, inside `LLMProviderTemplateData` (after the existing `TotalTokens` field,
line 84), insert the identical block from Step 1.

- [ ] **Step 3: Add the same fields to both v1alpha1 structs**

In `api/v1alpha1/llmprovidertemplate_types.go`, insert the identical block from Step 1 into both
the resource-mapping struct (near line 46) and the template-data struct (near line 74).

Both versions need the fields: conversion goes through JSON, so a field missing from `v1alpha1`
is dropped whenever an object round-trips through that version.

- [ ] **Step 4: Regenerate deepcopy and CRD manifests**

```bash
cd kubernetes/gateway-operator
make generate
make manifests
```

- [ ] **Step 5: Verify the CRD now accepts the new fields**

```bash
cd kubernetes/gateway-operator
grep -cE "cachedTokens|cacheWriteTokens|cacheWrite1hTokens|reasoningTokens|audioInputTokens|audioOutputTokens|serviceTier|cacheAccounting" \
  config/crd/bases/gateway.api-platform.wso2.com_llmprovidertemplates.yaml
```

Expected: a count of at least 16 — eight names across the two struct levels.

- [ ] **Step 6: Verify the enum constraint reached the CRD**

```bash
cd kubernetes/gateway-operator
grep -n -A 4 "cacheAccounting:" config/crd/bases/gateway.api-platform.wso2.com_llmprovidertemplates.yaml | head -20
```

Expected: each occurrence shows an `enum:` list containing `inclusive` and `additive`.

- [ ] **Step 7: Refresh the helm CRD copy**

```bash
cd kubernetes
cp gateway-operator/config/crd/bases/gateway.api-platform.wso2.com_llmprovidertemplates.yaml \
   helm/operator-helm-chart/crds/gateway.api-platform.wso2.com_llmprovidertemplates.yaml
diff -q gateway-operator/config/crd/bases/gateway.api-platform.wso2.com_llmprovidertemplates.yaml \
        helm/operator-helm-chart/crds/gateway.api-platform.wso2.com_llmprovidertemplates.yaml
```

Expected: `diff -q` prints nothing. The chart ships this file, so a stale copy would prune the
new fields at admission.

- [ ] **Step 8: Verify the operator builds and its tests pass**

```bash
cd kubernetes/gateway-operator
go build ./... && go test ./api/... ./internal/controller/...
```

Expected: `ok` for each package.

- [ ] **Step 9: Leave the changes uncommitted and report**

Do not commit or stage. Confirm the change set:

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform
git status --short kubernetes/
```

Expected: modifications limited to the two `llmprovidertemplate_types.go` files, the two
`zz_generated.deepcopy.go` files, and the two CRD YAMLs (bases + helm). If `make manifests`
or `make generate` touched any other CRD or file, list it in your report — regeneration can
pick up drift unrelated to this task, and the owner needs to know before committing.

---

### Task 4: Prove a field survives the full round trip

This is the task that actually demonstrates the schema works. Each earlier task verified its own
layer; this one proves nothing in between silently prunes a field, which is the failure mode the
whole plan exists to prevent.

**Files:**
- Test: `gateway/gateway-controller/pkg/config/llm_parser_test.go`

**Interfaces:**
- Consumes: the generated structs from Task 1 and the validator from Task 2.
- Produces: nothing consumed by later tasks.

Parsing goes through `NewParser()` from `pkg/config/parser.go` and its
`Parse(data []byte, contentType string, target any) error` method, which is how
`llm_parser_test.go` already does it. There is no `ParseLLMProviderTemplate` function.

- [ ] **Step 1: Write the failing round-trip test**

Append to `pkg/config/llm_parser_test.go`:

```go
// A template carrying the usage detail fields must survive YAML parsing with
// every field intact, including inside a resource mapping.
func TestParseLLMProviderTemplate_UsageDetailFieldsSurviveYAML(t *testing.T) {
	yamlSpec := []byte(`
apiVersion: gateway.api-platform.wso2.com/v1
kind: LlmProviderTemplate
metadata:
  name: roundtrip-check
spec:
  displayName: Roundtrip Check
  cacheAccounting: additive
  promptTokens:
    location: payload
    identifier: $.usage.input_tokens
  cachedTokens:
    location: payload
    identifier: $.usage.cache_read_input_tokens
  cacheWriteTokens:
    location: payload
    identifier: $.usage.cache_creation.ephemeral_5m_input_tokens
  cacheWrite1hTokens:
    location: payload
    identifier: $.usage.cache_creation.ephemeral_1h_input_tokens
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
    identifier: $.usage.service_tier
  resourceMappings:
    resources:
      - resource: /responses
        cacheAccounting: inclusive
        cachedTokens:
          location: payload
          identifier: $.usage.input_tokens_details.cached_tokens
`)

	parser := NewParser()

	var tmpl api.LLMProviderTemplate
	if err := parser.Parse(yamlSpec, "application/yaml", &tmpl); err != nil {
		t.Fatalf("parse failed: %v", err)
	}

	spec := tmpl.Spec

	// The parsed template must also pass validation, so a field that parses but
	// fails deploy-time checks is caught here too.
	if errs := NewLLMValidator().validateTemplateSpec(&spec); len(errs) != 0 {
		t.Fatalf("expected valid template, got %v", errs)
	}

	if spec.CacheAccounting == nil || *spec.CacheAccounting != "additive" {
		t.Errorf("cacheAccounting not preserved: %v", spec.CacheAccounting)
	}

	checks := map[string]*api.ExtractionIdentifier{
		"$.usage.cache_read_input_tokens":                        spec.CachedTokens,
		"$.usage.cache_creation.ephemeral_5m_input_tokens":       spec.CacheWriteTokens,
		"$.usage.cache_creation.ephemeral_1h_input_tokens":       spec.CacheWrite1hTokens,
		"$.usage.completion_tokens_details.reasoning_tokens":     spec.ReasoningTokens,
		"$.usage.prompt_tokens_details.audio_tokens":             spec.AudioInputTokens,
		"$.usage.completion_tokens_details.audio_tokens":         spec.AudioOutputTokens,
		"$.usage.service_tier":                                   spec.ServiceTier,
	}
	for want, got := range checks {
		if got == nil {
			t.Errorf("field for %q was dropped", want)
			continue
		}
		if got.Identifier != want {
			t.Errorf("identifier mismatch: got %q want %q", got.Identifier, want)
		}
	}

	if spec.ResourceMappings == nil || spec.ResourceMappings.Resources == nil {
		t.Fatal("resourceMappings was dropped")
	}
	resources := *spec.ResourceMappings.Resources
	if len(resources) != 1 {
		t.Fatalf("expected 1 resource mapping, got %d", len(resources))
	}
	m := resources[0]
	if m.CachedTokens == nil || m.CachedTokens.Identifier != "$.usage.input_tokens_details.cached_tokens" {
		t.Errorf("mapping cachedTokens not preserved: %v", m.CachedTokens)
	}
	if m.CacheAccounting == nil || *m.CacheAccounting != "inclusive" {
		t.Errorf("mapping cacheAccounting not preserved: %v", m.CacheAccounting)
	}
}
```

- [ ] **Step 2: Run the test to verify it passes**

```bash
cd gateway/gateway-controller
go test ./pkg/config/ -run UsageDetailFieldsSurviveYAML -v
```

Expected: PASS. It exercises Task 1's generated structs; a failure here means a JSON/YAML tag is
misspelled in the OpenAPI spec.

- [ ] **Step 3: Confirm every existing shipped template still validates**

```bash
cd gateway/gateway-controller && go test ./pkg/config/ ./pkg/utils/
```

Expected: `ok` for both. This is the backward-compatibility check — all seven shipped templates
are unedited and must still parse and validate.

- [ ] **Step 4: Leave the change uncommitted and report**

Do not commit or stage. Confirm only the test file changed:

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform
git status --short gateway/gateway-controller/pkg/config/
```

Report the test output; the repository owner commits.

---

## Definition of done

- [ ] `make generate-server-code` and `make manifests` produce no uncommitted diff when re-run.
- [ ] `config/crd/bases/...llmprovidertemplates.yaml` and the helm copy are byte-identical.
- [ ] `go test ./pkg/config/ ./pkg/utils/` passes in `gateway-controller`.
- [ ] `go build ./... && go test ./api/...` passes in `gateway-operator`.
- [ ] All seven shipped templates in `default-llm-provider-templates/` are **unchanged**.
- [ ] `llm-cost` is **unchanged**; no cost behaviour differs.

## What this plan deliberately excludes

- No extraction logic. That is Plan 2 (`sdk/ai/llmusage`).
- No template YAML content changes. Those land per provider in Plan 3.
- No `azure-llm-cost` changes.
- No `platform-api` changes — verified unnecessary.
- No pricing-file changes. The `cache_write_tokens` billing question is tracked as open
  question 4 in the design spec and is out of scope here.
