# Service-Tier Rate Backfill — Implementation Plan (Plan 8)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Supply the priority and flex rates the pricing file is missing, so a request served on those tiers is not silently billed at the standard rate.

**Architecture:** Candidate values come from the reference catalogue, are checked against the provider's own published pricing before being written, and are inserted textually so the diff shows only the additions. Nothing already in the file is altered.

**Scope:** 9 OpenAI models, up to 31 values, plus 2 Anthropic cache values. No code change is required — the fields are already read.

## Global Constraints

- **Never run `git commit`, `git add`, or any committing command.** No task ends in a commit.
- **Never run `git checkout <file>` or `git restore <file>`.** Back up with `cp` first.
- **Do not name the source catalogue anywhere** — not in code, comments, the pricing file, or output. Call it the reference catalogue.
- **`gateway/configs/llm-pricing/model_prices.json` is edited under explicit authorisation for this plan only.**
- **Additive only.** A field already present is never overwritten. Assert an overwrite count of zero after every edit.
- **Insert textually, never re-serialise.** A `json.dump` round-trip rewrites number notation across the whole file (`0.0000275` → `2.75e-05`), turning a 50-line change into a 3,000-line diff. Reuse the textual inserter from the previous backfill.
- **Both files or neither:** `configs/llm-pricing/model_prices.json` and `dev-policies/llm-cost-v2-1/testdata/model_prices.json` must stay in step, or tests stop reflecting production.
- **`gateway-controllers/policies/llm-cost` (v1) must not be modified.** It reads the same pricing file, so its costs change too; report that, do not prevent it.
- **All Go verification runs with `GOWORK=off`,** and `go clean -testcache` before asserting a pass.
- **Comments must never describe a change, fix, or history.**

## Decisions already taken

**1. The four `gpt-4.1*` search rates are NOT changed.** Our file says `0.01`; the reference catalogue says `0.025`. OpenAI's published pricing lists standard web search at **$10 / 1k calls** (`0.01`) and a preview tier at **$25 / 1k** (`0.025`) for non-reasoning models — and `gpt-4.1` is not a preview model. Our value matches the provider; the catalogue does not. Adopting the catalogue here would introduce a 2.5× overcharge on four mainstream models.

**2. The three genuinely tiered OpenAI entries are left alone.** `gpt-4o-mini-2024-07-18`, `gpt-4o-mini-search-preview`, `gpt-4o-search-preview` carry non-uniform maps and are all preview-family models, where per-context tiering is plausible. No evidence to change them.

**3. `max_input_tokens` / `max_tokens` for two Gemini embedding models are skipped.** Neither affects a charge; they feed context-window tier thresholds, and embeddings do not tier.

**4. The governing rule for this and future backfills:** trust the reference catalogue where we have no better source; prefer the provider's own published pricing wherever the two conflict. The `gpt-4.1*` case is why — treating the catalogue as ground truth would have introduced an overcharge while appearing to fix a gap.

## The gap this plan closes

| field | models | effect today |
|---|---|---|
| `input_cost_per_token_priority` | 9 | priority input billed at the standard rate |
| `output_cost_per_token_priority` | 8 | priority output billed at the standard rate |
| `cache_read_input_token_cost_priority` | 8 | priority cache reads billed at the standard rate |
| `input_cost_per_token_flex` | 2 | flex input billed at the standard rate |
| `output_cost_per_token_flex` | 2 | flex output billed at the standard rate |
| `cache_read_input_token_cost_flex` | 2 | flex cache reads billed at the standard rate |
| `cache_creation_input_token_cost_above_1hr_above_200k_tokens` | 2 | long-context 1-hour cache writes billed at the shorter-TTL rate |

The errors run both ways and some are large. On `gpt-4o-2024-08-06` a priority response bills output at `2.5e-06` where the catalogue says `1.7e-05` — **6.8× under**. On `gpt-5-nano-2025-08-07` priority input is `5e-08` against `2.5e-06` — **50× under**. Conversely every `cache_read_..._priority` is *lower* than the standard rate, so priority cache reads are currently **over**charged.

Affected models: `gpt-4.1-2025-04-14`, `gpt-4.1-mini-2025-04-14`, `gpt-4.1-nano-2025-04-14`, `gpt-4o-2024-08-06`, `gpt-4o-2024-11-20`, `gpt-4o-mini-2024-07-18`, `gpt-5-nano-2025-08-07`, `o3-2025-04-16`, `o4-mini-2025-04-16`; plus `claude-sonnet-4-5` and `claude-sonnet-4-5-20250929` for the cache field.

---

### Task 1: Check the candidate rates against the provider

Decision 4 exists because the catalogue was wrong once already. Do not skip this.

**Files:** none modified.

- [ ] **Step 1: List the candidates**

```bash
cd api-platform/gateway/configs/llm-pricing
python3 - <<'PY'
import json
ours=json.load(open('model_prices.json')); ref=json.load(open('<reference.json>'))
for f in ['input_cost_per_token_priority','output_cost_per_token_priority',
          'cache_read_input_token_cost_priority','input_cost_per_token_flex',
          'output_cost_per_token_flex','cache_read_input_token_cost_flex',
          'cache_creation_input_token_cost_above_1hr_above_200k_tokens']:
    for k in ours:
        if isinstance(ours[k],dict) and isinstance(ref.get(k),dict) \
           and f in ref[k] and f not in ours[k]:
            print(f"{k}\t{f}\t{ref[k][f]}\tstandard={ours[k].get('input_cost_per_token')}")
PY
```

- [ ] **Step 2: Check OpenAI's published priority pricing**

Look up the provider's own pricing page for the priority processing tier. Confirm at least the input and output rate for **`gpt-4o-2024-08-06`** and **`gpt-4.1-2025-04-14`** — two mainstream models with large deltas. Convert per-million figures to per-token before comparing (`$/1M ÷ 1e6`).

Record what the provider states beside what the catalogue states.

- [ ] **Step 3: Check the flex rates**

Same for `o3-2025-04-16` and `o4-mini-2025-04-16`. The provider documents a flex processing tier; confirm the input and output rates.

- [ ] **Step 4: Decide per field group**

- Provider and catalogue **agree** → backfill that group.
- They **disagree** → do not backfill it; record both values and stop for a decision. Prefer the provider.
- Provider **publishes nothing** for it → backfill from the catalogue and note it as catalogue-only.

Write the outcome into this plan's Decisions section before proceeding. A group that fails this check is out of scope, not something to fix later in the same pass.

- [ ] **Step 5: Confirm the Anthropic cache field**

`cache_creation_input_token_cost_above_1hr_above_200k_tokens` is a 1-hour cache write above the 200k context threshold. Confirm Anthropic publishes a distinct rate for that combination for `claude-sonnet-4-5`. If not, treat as catalogue-only.

---

### Task 2: Backfill the verified values

**Files:**
- Modify: `gateway/configs/llm-pricing/model_prices.json`
- Modify: `dev-policies/llm-cost-v2-1/testdata/model_prices.json`

**Interfaces:**
- Consumes: the field list approved in Task 1.
- Produces: both files carrying those values, with nothing else altered.

- [ ] **Step 1: Back up and record the cost baseline**

```bash
cd api-platform/gateway
mkdir -p /tmp/p8bak
cp configs/llm-pricing/model_prices.json /tmp/p8bak/real.json
cp dev-policies/llm-cost-v2-1/testdata/model_prices.json /tmp/p8bak/fixture.json
cd dev-policies/llm-cost-v2-1
GOWORK=off go clean -testcache
GOWORK=off go test -v ./... 2>&1 | grep -oE "0\.[0-9]{10}" | sort -u > /tmp/p8-cost-baseline.txt
cat /tmp/p8-cost-baseline.txt
```

- [ ] **Step 2: Write the inserter**

A scalar insert, simpler than the nested-map one used previously. For each target model, append the approved fields before the entry's closing brace, adding a comma to the preceding line. Do **not** load-and-dump the file.

```python
import json, re, sys, collections
target, refpath = sys.argv[1], sys.argv[2]
APPROVED = [...]                       # exactly the fields Task 1 approved
ref = json.load(open(refpath))
src = open(target).read()
data = json.loads(src, object_pairs_hook=collections.OrderedDict)

todo = {}
for m, e in data.items():
    if not isinstance(e, dict) or not isinstance(ref.get(m), dict):
        continue
    add = {f: ref[m][f] for f in APPROVED if f in ref[m] and f not in e}
    if add:
        todo[m] = add

out, current, added = [], None, 0
for line in src.split('\n'):
    m = re.match(r'^  "(.+)": \{$', line)
    if m and m.group(1) in todo:
        current = m.group(1)
    if current and re.match(r'^  \},?$', line):
        if not out[-1].rstrip().endswith(','):
            out[-1] = out[-1].rstrip() + ','
        items = list(todo[current].items())
        for i, (f, v) in enumerate(items):
            out.append('    "%s": %s%s' % (f, repr(v), '' if i == len(items)-1 else ','))
        added += len(items)
        current = None
    out.append(line)
open(target, 'w').write('\n'.join(out))
print("inserted", added, "values")
```

- [ ] **Step 3: Run against the fixture, then the deployed file**

```bash
cd api-platform/gateway
python3 /tmp/p8-insert.py dev-policies/llm-cost-v2-1/testdata/model_prices.json <reference.json>
python3 /tmp/p8-insert.py configs/llm-pricing/model_prices.json <reference.json>
```

- [ ] **Step 4: Assert the edit was additive and valid**

```bash
python3 - <<'PY'
import json
for name, old, new in [("fixture","/tmp/p8bak/fixture.json","dev-policies/llm-cost-v2-1/testdata/model_prices.json"),
                       ("deployed","/tmp/p8bak/real.json","configs/llm-pricing/model_prices.json")]:
    o=json.load(open(old)); n=json.load(open(new))
    over=[(m,f) for m,e in n.items() if isinstance(e,dict) for f,v in e.items()
          if isinstance(o.get(m),dict) and f in o[m] and o[m][f]!=v]
    added=[(m,f) for m,e in n.items() if isinstance(e,dict) and isinstance(o.get(m),dict)
           for f in e if f not in o[m]]
    print(f"{name}: valid={len(n)} models  overwritten={len(over)}  lost={len(set(o)-set(n))}  added={len(added)}")
    print("  fields:", {f for _,f in added})
    print("  models:", sorted({m for m,_ in added}))
PY
```

Expected: overwritten 0, lost 0, and the model list exactly the 9 (+2) named above.

- [ ] **Step 5: Check the diff size**

```bash
diff /tmp/p8bak/real.json configs/llm-pricing/model_prices.json | wc -l
```

Expected: roughly two lines per inserted value plus separators — tens, not thousands. A four-figure count means the file was re-serialised; restore from backup and fix the inserter.

---

### Task 3: Verify, and prove the tier rate is now used

A backfill nobody can observe is indistinguishable from no backfill.

- [ ] **Step 1: Diff the costs**

```bash
cd api-platform/gateway/dev-policies/llm-cost-v2-1
GOWORK=off go clean -testcache
GOWORK=off go test ./... 2>&1 | tail -3
GOWORK=off go test -v ./... 2>&1 | grep -oE "0\.[0-9]{10}" | sort -u | diff /tmp/p8-cost-baseline.txt -
```

Any value that moves must be traced to a specific model and field. An unexplained change is a stop condition.

- [ ] **Step 2: Write a test that fails without the backfill**

In `fees_test.go`, price the same token counts twice for `gpt-4o-2024-08-06` — once with `ServiceTier: ""`, once with `"priority"` — and assert the priority cost is strictly higher. Read the expected rates from the pricing entry rather than restating them, so the test tracks the data.

Confirm it would have failed before: temporarily restore `/tmp/p8bak/fixture.json`, run the test, see it fail because both tiers price identically, then restore the backfilled fixture.

- [ ] **Step 3: Re-verify all five providers independently**

Re-run the per-provider check: drive one response per provider through the shipped templates, recompute every cost component in Python straight from the pricing file, and compare to twelve decimal places. All five must match.

- [ ] **Step 4: Confirm the protected state**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/gateway-controllers && git status --porcelain policies/llm-cost/
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform && git status --porcelain | head
for r in . ../gateway-controllers; do (cd $r && git log --oneline -1); done
```

Expected: `llm-cost` v1 source unchanged, nothing committed, `model_prices.json` modified as the authorised exception.

---

### Task 4: Report

- [ ] **Step 1: State plainly**

Which field groups passed Task 1 and which were rejected, with the provider figure beside the catalogue figure for any rejection; how many values were written across how many models; which costs moved and why; that `llm-cost` v1 reads the same file and its priority-tier charges change identically; and anything skipped.

---

## Known limitations after this plan

- **The import still drops these fields.** This is a snapshot fix; the next regeneration loses them again unless whatever produces the pricing file is changed.
- **Only tiers with a rate are billed correctly.** A model whose priority rate is still absent silently bills the standard rate — the same failure this plan closes for nine models, left open for any model the catalogue does not cover.
- **`scale` and `fast` service tiers are not modelled.** Both appear in provider documentation; neither has rate columns in the pricing file, so both resolve to standard.
- **Free-tier allowances are not modelled** anywhere in the policy: they span requests, and a policy sees one response.
