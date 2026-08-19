# Pricing Data Backfill — Implementation Plan (Plan 7)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Close the gap between the rates the gateway's pricing file carries and the rates the cost code is able to read, so charges come from data rather than from constants.

**Architecture:** The pricing file is compared field-by-field against the reference catalogue it is derived from, for every field `ModelPricing` reads. Missing values are backfilled by script — additively, never overwriting an existing value. Costs are computed before and after and every difference is accounted for.

**Tech Stack:** Python for the comparison and backfill, Go for the cost verification.

## Global Constraints

- **Never run `git commit`, `git add`, or any committing command.** No task ends in a commit.
- **Never run `git checkout <file>` or `git restore <file>`.** Back up with `cp` before editing.
- **Do not name the source catalogue anywhere** — not in code, comments, the pricing file, commit messages, or this plan's output. Refer to it as the reference catalogue. If a previous edit named it, remove that.
- **`gateway/configs/llm-pricing/model_prices.json` is edited under explicit authorisation for this plan only.** It is otherwise an owner-held file. Every edit must be scripted, additive, and reversible from a backup taken in Task 1.
- **Additive only.** Never delete or overwrite a value already in the pricing file. Where a stale value exists, add the field the code prefers instead — the code's precedence order makes the stale one unreachable without removing it.
- **`gateway-controllers/policies/llm-cost` (v1) must not be modified.** It reads the same pricing file, so backfilled rates reach it too; that is expected and must be reported, not prevented.
- **All Go verification runs with `GOWORK=off`,** and `go clean -testcache` before asserting a pass.
- **Comments must never describe a change, fix, or history.**

## What the analysis already established

Measured across the 823 models present in both files, for the 50 fields `ModelPricing` reads:

| field | ours | reference | missing |
|---|---|---|---|
| `search_context_cost_per_query` | 77 | 201 | **124** |
| `input_cost_per_token_priority` | 82 | 89 | 9 |
| `output_cost_per_token_priority` | 81 | 87 | 8 |
| `cache_read_input_token_cost_priority` | 79 | 85 | 8 |
| `input_cost_per_token_flex` | 12 | 14 | 2 |
| `output_cost_per_token_flex` | 12 | 14 | 2 |
| `cache_read_input_token_cost_flex` | 10 | 12 | 2 |
| `cache_creation_input_token_cost_above_1hr_above_200k_tokens` | 7 | 9 | 2 |
| `max_input_tokens` / `max_tokens` | 685 / 675 | 687 / 677 | 2 / 2 |
| **total** | | | **161** |

**124 of 161 are the search rate, and 52 of those are Gemini** — which is why a Gemini grounding fee ended up as a Go constant.

Three further facts, each verified:

1. **`web_search_cost_per_request` does not exist in the reference catalogue** (0 entries) yet 46 of our models rely on it alone. It is introduced somewhere in the import.
2. **Four of those 46 disagree with the reference** — `gpt-4.1`, `gpt-4.1-2025-04-14`, `gpt-4.1-mini` and one more carry a flat `0.01` where the reference says `0.025` across all tiers. They undercharge web search by 2.5×.
3. **`web_search_billing_unit` has exactly one value** across all 41 models that carry it: `per_query`. All 41 also carry the tiered map. It therefore states what the code already does and is **not** worth carrying or reading — recorded in Decisions below.

## Decisions taken

1. **Backfill `search_context_cost_per_query`, not `web_search_cost_per_request`.** The code prefers the tiered map (`pricing.go:582`), so adding the map both fills the 124 gaps *and* corrects the four stale flat rates without deleting anything. The stale flat values become unreachable rather than wrong.
2. **`web_search_billing_unit` is not added.** One distinct value, agreeing with existing behaviour. A field with no variance carries no information. Revisit if a second value ever appears.
3. **Backfill every other missing value the code reads.** The 37 non-search gaps are small but free to close, and a missing priority rate silently bills the standard rate — the same class of silent-undercharge the search gap caused.
4. **`max_input_tokens` / `max_tokens` are included** even though they do not affect a charge, because they are read by `ModelPricing` and the tier thresholds consult context size.
5. **Gemini grounding stays rate-driven.** Plan 6's change already removed the constant; this plan supplies the data that makes it charge correctly. If a Gemini model still lacks a rate after backfill, it charges nothing — visible in `aiCost`, unlike a constant.

## File Structure

| File | Responsibility |
|---|---|
| `gateway/configs/llm-pricing/model_prices.json` | the deployed rates; backfilled additively |
| `dev-policies/llm-cost-v2-1/testdata/model_prices.json` | the test fixture; backfilled identically so tests reflect production |
| `gateway/spec/pricing-gap-report.md` | **new** — the written gap analysis, reproducible |

---

### Task 1: Freeze a baseline and produce the gap report

Nothing may be edited before the current costs are recorded, or a later difference cannot be attributed.

**Files:**
- Create: `api-platform/gateway/spec/pricing-gap-report.md`
- Back up: both pricing files

- [ ] **Step 1: Back up both pricing files**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform/gateway
mkdir -p /tmp/p7-pricing-bak
cp configs/llm-pricing/model_prices.json /tmp/p7-pricing-bak/real.json
cp dev-policies/llm-cost-v2-1/testdata/model_prices.json /tmp/p7-pricing-bak/fixture.json
ls -la /tmp/p7-pricing-bak/
```

- [ ] **Step 2: Record the cost baseline**

```bash
cd api-platform/gateway/dev-policies/llm-cost-v2-1
GOWORK=off go clean -testcache
GOWORK=off go test -v ./... 2>&1 | grep -oE "0\.[0-9]{10}" | sort -u > /tmp/p7-cost-baseline.txt
cat /tmp/p7-cost-baseline.txt
```

Every value here must still be explainable after the backfill. A cost that changes must be traceable to a rate that was missing.

- [ ] **Step 3: Write the gap report**

Create `gateway/spec/pricing-gap-report.md` containing, for each of the 50 fields `ModelPricing` reads: how many models carry it in the pricing file, how many in the reference catalogue, and how many are missing. Include the three findings above (the introduced `web_search_cost_per_request`, the four stale rates by name, and the single-valued `web_search_billing_unit`). Do not name the reference catalogue.

Generate the table with the script from Step 4 rather than by hand.

- [ ] **Step 4: Make the comparison reproducible**

The report must be regenerable. Save the comparison as a script at `gateway/spec/pricing-gap-check.py`, taking both file paths as arguments and printing the table. It must read the field list out of `pricing.go` rather than hardcoding it, so a new rate field is picked up automatically:

```python
fields = re.findall(r'`json:"([a-z0-9_]+)"`',
                    re.search(r'type ModelPricing struct.*?\n\}', src, re.S).group(0))
```

- [ ] **Step 5: Verify the script reproduces the reported numbers**

```bash
cd api-platform/gateway
python3 spec/pricing-gap-check.py configs/llm-pricing/model_prices.json <reference.json> | tail -20
```

Expected: `search_context_cost_per_query` missing 124, total 161.

---

### Task 2: Backfill the pricing file

**Files:**
- Modify: `gateway/configs/llm-pricing/model_prices.json`
- Modify: `dev-policies/llm-cost-v2-1/testdata/model_prices.json`

**Interfaces:**
- Produces: both files carrying every value the reference has for the 50 fields the code reads, with no existing value altered.

- [ ] **Step 1: Write the backfill as a script, not by hand**

`/tmp/p7-backfill.py`, taking a target file and the reference:

```python
import json, sys, re

target_path, ref_path, gopath = sys.argv[1], sys.argv[2], sys.argv[3]
src = open(gopath).read()
struct = re.search(r'type ModelPricing struct.*?\n\}', src, re.S).group(0)
fields = re.findall(r'`json:"([a-z0-9_]+)"`', struct)

target = json.load(open(target_path))
ref    = json.load(open(ref_path))

added, touched = 0, set()
for model, entry in target.items():
    if not isinstance(entry, dict):
        continue
    source = ref.get(model)
    if not isinstance(source, dict):
        continue
    for field in fields:
        if field in source and field not in entry:   # additive only
            entry[field] = source[field]
            added += 1
            touched.add(model)

json.dump(target, open(target_path, 'w'), indent=4, sort_keys=True)
print(f"added {added} values across {len(touched)} models")
```

The `field not in entry` guard is what makes it additive. Do not remove it.

- [ ] **Step 2: Check the file's existing formatting first**

Rewriting 932 entries with the wrong indent or key order produces an unreadable diff.

```bash
cd api-platform/gateway/configs/llm-pricing
head -12 model_prices.json
python3 -c "
import json; d=json.load(open('model_prices.json'))
k=list(d)[1]; print('keys sorted?', list(d[k])==sorted(d[k]))
print('top-level sorted?', list(d)==sorted(d))"
```

Match `indent` and `sort_keys` in the script to what this reports before running it.

- [ ] **Step 3: Run it against the fixture first**

```bash
cd api-platform/gateway/dev-policies/llm-cost-v2-1
python3 /tmp/p7-backfill.py testdata/model_prices.json <reference.json> pricing.go
```

- [ ] **Step 4: Confirm the fixture edit was additive**

```bash
python3 - <<'PY'
import json
old=json.load(open('/tmp/p7-pricing-bak/fixture.json'))
new=json.load(open('testdata/model_prices.json'))
changed=[(m,f) for m,e in new.items() if isinstance(e,dict)
         for f,v in e.items()
         if isinstance(old.get(m),dict) and f in old[m] and old[m][f]!=v]
print("values OVERWRITTEN (must be 0):", len(changed), changed[:5])
print("models lost (must be 0):", len(set(old)-set(new)))
PY
```

- [ ] **Step 5: Run the same against the deployed file**

```bash
cd api-platform/gateway/configs/llm-pricing
python3 /tmp/p7-backfill.py model_prices.json <reference.json> \
  ../../dev-policies/llm-cost-v2-1/pricing.go
```

Then repeat the Step 4 check against `/tmp/p7-pricing-bak/real.json`.

- [ ] **Step 6: Confirm the gap is closed**

```bash
cd api-platform/gateway
python3 spec/pricing-gap-check.py configs/llm-pricing/model_prices.json <reference.json> | tail -6
```

Expected: total missing 0, or only fields the code does not read.

---

### Task 3: Verify every cost change is explained

A backfill that changes a cost is doing its job; a backfill that changes a cost nobody can explain is a defect.

- [ ] **Step 1: Recompute the costs**

```bash
cd api-platform/gateway/dev-policies/llm-cost-v2-1
GOWORK=off go clean -testcache
GOWORK=off go test ./... 2>&1 | tail -3
GOWORK=off go test -v ./... 2>&1 | grep -oE "0\.[0-9]{10}" | sort -u > /tmp/p7-cost-after.txt
diff /tmp/p7-cost-baseline.txt /tmp/p7-cost-after.txt
```

- [ ] **Step 2: Account for every difference**

For each value that appears or disappears, name the model, the field that was backfilled, and why it moves the cost. A test whose figure changes because a priority rate now exists is correct; a test whose figure changes for no identifiable reason is not — stop and investigate before proceeding.

- [ ] **Step 3: Update any test that asserted a now-stale figure**

Only after Step 2 explains it. Record the old and new value in the test's comment — as a statement of what the rate is, not of what changed.

- [ ] **Step 4: Re-verify per provider against an independent calculation**

Re-run the per-provider check from the Plan 6 verification: drive one response per provider through the shipped templates, then recompute every cost component in Python straight from the pricing file and compare to twelve decimal places. All five providers must match.

---

### Task 4: Confirm Gemini grounding now charges

The point of the exercise.

- [ ] **Step 1: Confirm Gemini models now carry a search rate**

```bash
cd api-platform/gateway/configs/llm-pricing
python3 -c "
import json; d=json.load(open('model_prices.json'))
g=[k for k,v in d.items() if isinstance(v,dict) and 'gemini' in k.lower()
   and v.get('search_context_cost_per_query')]
print(len(g),'gemini models now carry a search rate'); print(g[:5])"
```

Expected: ~52, where it was 0.

- [ ] **Step 2: Write the test**

In `dev-policies/llm-cost-v2-1/fees_test.go`, drive a Gemini response carrying grounding queries through the shipped template and assert the cost exceeds the token-only cost by `queries × rate`, reading the rate from the pricing entry rather than restating it.

- [ ] **Step 3: Run it**

```bash
cd api-platform/gateway/dev-policies/llm-cost-v2-1
GOWORK=off go clean -testcache
GOWORK=off go test -run Grounding -v ./... 2>&1 | grep -E "^(--- |ok|FAIL)"
```

- [ ] **Step 4: Confirm no Go constant remains**

```bash
grep -rnE "= *[0-9]+(\.[0-9]+)?e-[0-9]+|const .*Cost.*=" calculator_*.go || echo "no hardcoded prices"
```

---

### Task 5: Report

- [ ] **Step 1: State plainly**

How many values were backfilled and across how many models; which costs changed and why; that `llm-cost` v1 reads the same file and is affected identically; that `web_search_billing_unit` was deliberately not added, with the reason; and anything skipped.

- [ ] **Step 2: Confirm the protected state**

```bash
cd /Users/irashperera/Documents/APIM/api-platform/dev/gateway-controllers && git status --porcelain policies/llm-cost/
cd /Users/irashperera/Documents/APIM/api-platform/dev/api-platform && git status --porcelain | head -20
for r in . ../gateway-controllers; do (cd $r && git log --oneline -1); done
```

Expected: `llm-cost` v1 unchanged, nothing committed, and `model_prices.json` modified — the one authorised exception.

---

## Known limitations after this plan

- **The import itself is not fixed.** This backfills a snapshot. The next regeneration of the pricing file will drop the same 161 values again unless whatever produces it is changed. That is outside this repo and is the durable fix.
- **`web_search_cost_per_request` remains** on 46 models, unreachable wherever a tiered map was added. Harmless, but it is a field the reference does not have, and its origin is unexplained.
- **Free-tier allowances are not modelled.** Grounding carries a monthly or daily free quota; a policy sees one response at a time and cannot track it, so searches are charged from the first query.
- **Only fields `ModelPricing` reads are backfilled.** 44 other keys exist in the file and are ignored by the code; if one later becomes chargeable it must be added to the struct and re-backfilled.
