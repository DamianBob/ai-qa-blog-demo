# Edge Case Analysis — AI Blog QA Pipeline

> This document catalogues every identified failure mode in the two-agent QA pipeline, how the workflow currently handles each one, and what a production-grade hardening would look like. Cases are grouped by origin layer and ranked by severity.

---

## Severity Scale

| Level | Meaning |
|-------|---------|
| 🔴 **P0** | Silent data corruption or undiagnosable crash — must be fixed |
| 🟡 **P1** | Wrong output with no crash — degrades quality silently |
| 🟢 **P2** | Nice-to-have robustness improvement |

---

## Layer 1 — GitHub API

### G1 · File not found (404 on GET) 🔴 P0

**Scenario:** `posts/demo-post.md` was accidentally deleted from the repo, or the path in `Set: Config` is wrong.

**Current behaviour:** GitHub returns `{ "message": "Not Found" }` with no `content` field. Node 4 (`Decode & Prepare Lines`) calls `Buffer.from(undefined, 'base64')` — TypeError, cryptic crash.

**Desired behaviour:** Fail immediately with a human-readable error before attempting to decode anything.

**Fix (Node 4):**
```js
if (!ghResponse.content) {
  throw new Error(
    `GitHub file not found: "${ghResponse.message ?? 'no content returned'}". ` +
    `Check that ${filePath} exists in ${owner}/${repo}.`
  );
}
```

---

### G2 · Missing or invalid SHA 🟡 P1

**Scenario:** GitHub changes its response schema, or a branch-level permission strips the SHA field.

**Current behaviour:** `sha` is `undefined`. Downstream nodes thread it silently. The final PUT to GitHub either creates a duplicate file or returns 422 ("SHA does not match") depending on whether the file already exists.

**Desired behaviour:** Validate the SHA immediately at the GET step — fail loudly rather than silently corrupt the commit.

**Fix (Node 4):**
```js
if (!sha || typeof sha !== 'string') {
  throw new Error('GitHub GET did not return a valid SHA. Cannot safely commit without it.');
}
```

---

### G3 · SHA mismatch — 409 Conflict on PUT 🟡 P1

**Scenario:** A concurrent workflow run (two triggers fired in quick succession) modifies `demo-post.md` between the GET and the PUT. The SHA captured at GET is now stale.

**Current behaviour:** GitHub returns 409. n8n records a generic node error with no context about why.

**Desired behaviour:** Detect the 409 and surface a clear "file modified externally — re-run to fetch the latest SHA" message. For a demo workflow, halting with context is preferable to a silent retry that might commit the wrong content.

**Fix:** Set `GitHub Commit File`'s "On Error" to Continue, then add a guard node:
```js
if ($input.first().json.statusCode === 409) {
  throw new Error(
    'GitHub 409 Conflict: demo-post.md was modified between GET and PUT. ' +
    'Re-run the workflow to capture the current SHA.'
  );
}
```

---

### G4 · Rate limiting — 403 / 429 🟢 P2

**Scenario:** The GitHub PAT hits the authenticated rate limit (5,000 requests/hour). Unlikely in demo conditions (each pipeline run uses ~6 calls) but possible if the workflow is triggered in a loop.

**Current behaviour:** HTTP 429 arrives; n8n records a node error.

**Desired behaviour:** A dedicated branch that explains rate-limit context and instructs the operator to wait. No code change needed — a simple downstream status check and routing IF node is sufficient.

---

### G5 · File too large — 403 from GitHub Contents API 🟢 P2

**Scenario:** GitHub's Contents API returns 403 for files over 1MB. Not realistic for a blog post, but relevant if the workflow is repurposed for larger documents.

**Current behaviour:** Same as G1 — crash at `Buffer.from(undefined, ...)`.

**Desired behaviour:** Detect the "too large" message in the G1 guard:
```js
if (ghResponse.message?.includes('too large')) {
  throw new Error('File exceeds 1MB — use the Git Data API (trees/blobs) for large files.');
}
```

---

## Layer 2 — Agent Response Parsing

This is the highest-risk layer. LLMs are non-deterministic and will occasionally produce output that deviates from the specified contract — especially under token pressure.

### A1 · Response truncated mid-JSON at `max_tokens` 🔴 P0

**Scenario:** The agent has many issues to report. At token 2000 the response is cut off mid-array, e.g. `[{"line_range":[3,3],"issue_type":"grammar","desc`. `JSON.parse` throws. All partial data is silently discarded.

**Current behaviour:** `catch` block sets `issues = []`. The workflow continues as if there are no issues — producing a misleading "clean" QA report.

**Why this matters:** With 30 issues found in testing, the JSON output is ~3,000 characters. A longer post or more verbose model response will exceed 2,000 tokens and trigger truncation. The failure is silent and looks like success.

**Fix — truncation recovery:**
```js
} catch(e) {
  const lastClose = cleaned.lastIndexOf('}');
  if (lastClose > -1) {
    try {
      const partial = cleaned.slice(0, lastClose + 1).replace(/,\s*$/, '') + ']';
      const recovered = JSON.parse(partial);
      if (Array.isArray(recovered)) {
        issues = recovered;
        parseWarning = 'truncated_response_partial_recovery';
      }
    } catch { issues = []; }
  }
}
```

**Additionally:** Check `choices[0].finish_reason === 'length'` and surface it in the QA report so the operator knows the output may be incomplete.

---

### A2 · OpenRouter 402 / 5xx — crash on `choices` access 🔴 P0

**Scenario:** Credit is exhausted (402) or OpenRouter has a transient error (503). The response body has no `choices` array.

**Current behaviour:** `$input.first().json.choices[0].message.content` throws `TypeError: Cannot read properties of undefined (reading '0')`. The node fails with a cryptic error that gives no hint about the actual cause.

**Fix — guard before touching `choices`:**
```js
const apiResponse = $input.first().json;
if (!apiResponse.choices?.[0]?.message?.content) {
  const status = apiResponse.statusCode ?? apiResponse.status ?? 'unknown';
  throw new Error(
    `OpenRouter API error (HTTP ${status}): ${JSON.stringify(apiResponse).slice(0, 300)}`
  );
}
```

---

### A3 · Agent wraps JSON in prose 🟡 P1

**Scenario:** Agent ignores the "return JSON only" instruction and responds with: *"Here are the issues I found: [...] Let me know if you need anything else."*

**Current behaviour:** The fence-stripping regex does nothing useful. `JSON.parse` fails on the surrounding text. Issues discarded.

**Fix — `extractJSON` helper that finds `[`…`]` boundaries regardless of surrounding content:**
```js
function extractJSON(str) {
  const s = str.replace(/^```[a-z]*\n?/i, '').replace(/\n?```$/, '').trim();
  const start = s.indexOf('[');
  const end   = s.lastIndexOf(']');
  if (start !== -1 && end > start) {
    try { return { data: JSON.parse(s.slice(start, end + 1)), warning: null }; }
    catch { /* fall through to truncation recovery */ }
  }
  return { data: null, warning: 'parse_failed' };
}
```

This also handles trailing prose after a valid JSON array.

---

### A4 · Non-`json` fence specifier 🟡 P1

**Scenario:** Agent wraps output in ` ```javascript `, ` ```text `, or plain ` ``` ` instead of ` ```json `.

**Current behaviour:** The regex `/^```json\n?/` only matches the ` ```json ` variant. Other fences are not stripped. `JSON.parse` fails on the backtick characters.

**Fix — broaden the fence regex:**
```js
const cleaned = raw
  .replace(/^```[a-z]*\n?/i, '')
  .replace(/\n?```$/, '');
```

---

### A5 · Agent wraps array in an object 🟡 P1

**Scenario:** Agent returns `{ "issues": [...] }` or `{ "patches": [...] }` instead of a bare array.

**Current behaviour:** `!Array.isArray(issues)` is true → `issues = []`. All valid data discarded.

**Fix — unwrap common wrapper keys before falling back:**
```js
if (!Array.isArray(issues) && typeof issues === 'object' && issues !== null) {
  for (const k of ['issues', 'patches', 'items', 'results', 'data']) {
    if (Array.isArray(issues[k])) { issues = issues[k]; break; }
  }
  if (!Array.isArray(issues)) issues = [];
}
```

---

### A6 · Empty agent response 🟡 P1

**Scenario:** Agent returns an empty string (credit soft-limit, content filter, or model glitch).

**Current behaviour:** `JSON.parse('')` throws. `catch` → `issues = []`. Indistinguishable from "the file has no issues." The QA report looks clean.

**Fix — distinguish empty response from genuinely clean output:**
```js
if (!raw || raw.length === 0) {
  return [{ json: {
    issues: [],
    issueCount: 0,
    parseError: 'empty_agent_response',
    // ... thread remaining fields
  }}];
}
```

---

### A7 · Agent uses `line_number` (singular) instead of `line_range` 🟢 P2

**Scenario:** Agent returns `{ "line_number": 5 }` instead of `{ "line_range": [5, 5] }`.

**Current behaviour:** Already handled. Both `Parse QA JSON` and `Parse & Validate Patches` use `p.line_range?.[0] ?? p.line_number` as a fallback. No fix needed.

---

## Layer 3 — Line Number Validity

### L1 · Overlapping patch ranges 🔴 P0

**Scenario:** Agent returns two patches targeting `[5, 8]` and `[7, 10]`. Both pass individual OOB validation. Applied bottom-to-top (patch at 7 runs first on the original array, then patch at 5 runs on the *already modified* array), the second patch operates on content that has shifted — producing silently corrupted output.

**Current behaviour:** Both patches are accepted and applied. File content is corrupted with no error or warning.

**Why this matters:** Overlapping patches are the most likely structural failure mode. When the agent sees multiple issues in a dense paragraph, it will naturally produce overlapping ranges.

**Fix — overlap sweep after descending sort:**
```js
const validPatches = [];
let lastAcceptedStart = Infinity;
for (const p of candidatePatches) { // already sorted descending
  const pEnd   = p.line_range[1];
  const pStart = p.line_range[0];
  if (pEnd < lastAcceptedStart) {
    validPatches.push(p);
    lastAcceptedStart = pStart;
  } else {
    skippedCount++;
    skippedReasons.push(
      `patch [${p.line_range}] overlaps accepted patch starting at line ${lastAcceptedStart} — skipped`
    );
  }
}
```

---

### L2 · Reversed line range — `end < start` 🟡 P1

**Scenario:** Agent returns `[10, 3]`. Possibly caused by response truncation mid-object.

**Current behaviour:** `start <= end` check fires → patch silently dropped.

**Desired behaviour:** Swap and accept, log a warning. If the agent consistently reverses ranges, silent dropping means *all* its patches are discarded.

**Fix:**
```js
if (Number.isInteger(start) && Number.isInteger(end) && start > end) {
  [start, end] = [end, start];
  skippedReasons.push(`patch [${p.line_range}] had reversed range — swapped to [${start}, ${end}]`);
}
```

---

### L3 · Zero-indexed line number 🟢 P2

**Scenario:** Agent uses 0-based indexing: `[0, 0]` for the first line.

**Current behaviour:** `start >= 1` check filters it out. Correct outcome, but silently.

**Desired behaviour:** Log the rejection reason so repeated zero-indexing from the agent is visible in the QA report:
```js
skippedReasons.push(`patch [${p.line_range}] — zero-indexed? start < 1`);
```

---

### L4 · Floating-point line numbers 🟢 P2

**Scenario:** Agent returns `[3.7, 5.2]` (rare but possible).

**Current behaviour:** `Number.isInteger(3.7)` is `false` → filtered.

**Fix — round before the integer check:**
```js
start = Math.round(start);
end   = Math.round(end);
```

---

### L5 · Agent echoes line-number prefixes into `new_text` 🟡 P1

**Scenario:** Both agents receive numbered content (`5: Some text`). The editor may echo the prefix back: `"new_text": "5: Corrected text"`. Inserted verbatim, the committed file contains `5: Corrected text` — the file now has visible line-number artifacts.

**Current behaviour:** `new_text` is inserted verbatim. Corruption is silent and only visible in GitHub.

**Fix (Apply Patches node) — strip prefix before inserting:**
```js
const cleanLines = (text) =>
  (text ?? '').split('\n').map(l => l.replace(/^\d+:\s*/, ''));
```

---

### L6 · `insert_after` at the last line 🟢 P2 (non-issue)

**Scenario:** `insert_after` targets line 30 of a 30-line file.

**Current behaviour:** `lines.splice(end + 1, 0, ...)` — JavaScript's `splice` treats an out-of-bound index as an append. Correct behaviour, no fix needed. Documented for completeness.

---

### L7 · `replace` with empty `new_text` 🟢 P2

**Scenario:** Agent returns `{ "operation": "replace", "new_text": "" }`. Intended meaning is ambiguous — blank line, or delete?

**Current behaviour:** `''.split('\n')` → `['']` → replaced with a single empty line. Line is not deleted; it becomes blank.

**Fix — treat empty replace as delete:**
```js
if (patch.operation === 'replace' && (patch.new_text ?? '').trim() === '') {
  lines.splice(start, end - start + 1);
} else {
  lines.splice(start, end - start + 1, ...cleanLines(patch.new_text));
}
```

---

## Layer 4 — Data Threading

The workflow uses explicit node references (`$('Decode & Prepare Lines').first().json`) to re-attach file metadata after the QA Agent overwrites `$json`. This is a deliberate architectural choice — but it introduces its own failure surface.

### T1 · Node reference returns null 🟡 P1

**Scenario:** The node named `Decode & Prepare Lines` is renamed during editing. The reference silently returns null.

**Current behaviour:** Destructuring `null` throws a TypeError. Error message references an internal n8n line, not the workflow design. Hard to diagnose.

**Fix:**
```js
const decodeRef = $('Decode & Prepare Lines').first()?.json;
if (!decodeRef) {
  throw new Error(
    'Could not find output from "Decode & Prepare Lines". ' +
    'Check the node name has not been renamed.'
  );
}
```

---

### T2 · `originalLines` null or missing in downstream nodes 🔴 P0

**Scenario:** A threading failure upstream means `originalLines` arrives as `undefined` in `Parse & Validate Patches`.

**Current behaviour:** `originalLines.length` throws TypeError. Workflow crashes with a cryptic error in Node 8.

**Fix:**
```js
const originalLines = $input.first().json.originalLines;
if (!Array.isArray(originalLines) || originalLines.length === 0) {
  throw new Error(
    'originalLines is missing or empty — upstream data threading has failed. ' +
    'Check Parse QA JSON output.'
  );
}
```

---

## Layer 5 — Token Budget and Credit

### C1 · OpenRouter `finish_reason: "length"` not surfaced 🟡 P1

**Scenario:** The agent hit the token cap but returned what it had. The response is valid JSON — but incomplete. The workflow treats it as a successful full response.

**Current behaviour:** No distinction between a complete response and a token-capped one.

**Fix:** Read `choices[0].finish_reason` and propagate it to the QA report:
```js
const finishReason = apiResponse.choices[0].finish_reason;
// Surface in output:
parseWarning: finishReason === 'length' ? 'response_may_be_incomplete_at_token_limit' : null
```

---

### C2 · `max_tokens` accidentally removed 🟢 P2

**Scenario:** A future editor removes `max_tokens: 2000` from the agent payload. OpenRouter returns 402 on the next run.

**Current behaviour:** 402 → crash (see A2).

**Fix:** Add a comment in the HTTP body expression:
```
// max_tokens is REQUIRED for OpenRouter free tier. Do not remove.
```
Combined with the A2 guard, the 402 will surface a clear error rather than a cryptic crash.

---

## Summary Table

| ID | Category | Scenario | Severity | Status |
|----|----------|----------|----------|--------|
| G1 | GitHub API | File not found (404) | 🔴 P0 | Planned |
| G2 | GitHub API | Missing SHA | 🟡 P1 | Planned |
| G3 | GitHub API | SHA mismatch 409 | 🟡 P1 | Planned |
| G4 | GitHub API | Rate limiting | 🟢 P2 | Documented |
| G5 | GitHub API | File too large | 🟢 P2 | Documented |
| A1 | Agent Output | Truncated mid-JSON | 🔴 P0 | Planned |
| A2 | Agent Output | 402 / 5xx crash | 🔴 P0 | Planned |
| A3 | Agent Output | JSON wrapped in prose | 🟡 P1 | Planned |
| A4 | Agent Output | Non-`json` fence | 🟡 P1 | Planned |
| A5 | Agent Output | Object-wrapped array | 🟡 P1 | Planned |
| A6 | Agent Output | Empty response | 🟡 P1 | Planned |
| A7 | Agent Output | `line_number` singular | 🟢 P2 | ✅ Already handled |
| L1 | Line Numbers | Overlapping patches | 🔴 P0 | Planned |
| L2 | Line Numbers | Reversed range | 🟡 P1 | Planned |
| L3 | Line Numbers | Zero-indexed | 🟢 P2 | Planned |
| L4 | Line Numbers | Floating-point | 🟢 P2 | Planned |
| L5 | Line Numbers | Echoed prefix in `new_text` | 🟡 P1 | Planned |
| L6 | Line Numbers | `insert_after` at last line | 🟢 P2 | ✅ Already handled |
| L7 | Line Numbers | Empty `replace` | 🟢 P2 | Planned |
| T1 | Threading | Node reference null | 🟡 P1 | Planned |
| T2 | Threading | `originalLines` missing | 🔴 P0 | Planned |
| C1 | Token Budget | `finish_reason` not surfaced | 🟡 P1 | Planned |
| C2 | Token Budget | `max_tokens` accidentally removed | 🟢 P2 | Documented |

---

## Implementation Order

Fix in this sequence to avoid compounding failures during testing:

1. **Node 4** — G1 (file not found), G2 (SHA guard). These are the earliest possible crash points.
2. **Parse QA JSON** — A2 (API error guard), A1 (truncation recovery), A3+A4 (`extractJSON`), A5 (object unwrap), A6 (empty response), T1 (node reference null).
3. **Parse & Validate Patches** — A2, A1, A3+A4, A5, T2 (`originalLines` guard), L1 (overlap sweep), L2 (reversed range swap).
4. **Apply Patches** — L5 (strip echoed line-number prefixes from `new_text`).
5. **GitHub Commit File** — G3 (409 detection via downstream guard node).
6. **QA Agent + Editor Agent nodes** — Enable "Retry on Fail" (3 retries, 2s delay) for transient 5xx.
7. **Prepare QA Report** — Surface `skippedReasons`, `parseWarning`, `finishReason` in committed JSON.
