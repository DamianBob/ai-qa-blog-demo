# Task 2 — Two-Agent AI QA Workflow in n8n

**Task:** Two-Agent AI QA for a Markdown Blog Post in GitHub (Advanced)
**n8n Instance:** https://n8n-97i5.onrender.com
**Workflow:** AI Blog QA Pipeline (ID: `zadDEU66iTk0AfXB`)
**GitHub Repo:** https://github.com/DamianBob/ai-qa-blog-demo

---

## Choice of Task

I picked Task 2 because it sits at the intersection of the things I find most interesting complex automations and complicated things. It's perfect for me if I can make it as interesting as possible.

It also mirrors a pattern I care about: Having agents and AI handle complicated things and humans handle simple things and oversight.


---

## Workflow Design

### Architecture (18 nodes)


[Webhook Trigger] ─┐
                   ├──▶ [Get Reset SHA] ──▶ [Reset to Original]
[Manual Trigger] ──┘              (HTTP GET)        (HTTP PUT hardcoded original)
                                                           │
                                                           ▼
                                                    [Set: Config]
                                                    owner / repo / filePath / branch
                                                           │
                                                           ▼
                                                  [GitHub GET File]
                                                  (base64 content + SHA)
                                                           │
                                                           ▼
                                              [Decode & Prepare Lines]
                                              prepend line numbers to each line
                                                           │
                                                           ▼
                                          [QA Agent — Claude Sonnet 4.6]
                                          returns JSON array of issues
                                                           │
                                                           ▼
                                              [Parse QA JSON]
                                              strip fences, validate array
                                              + thread originalLines/sha/filePath
                                                           │
                                                           ▼
                                        [Editor Agent — Claude Sonnet 4.6]
                                        returns JSON array of patch operations
                                                           │
                                                           ▼
                                          [Parse & Validate Patches]
                                          filter OOB, sort descending
                                                           │
                                                           ▼
                                             [IF: validPatches.length > 0]
                                              /                        \
                                           TRUE                       FALSE
                                             ▼                          ▼
                                    [Apply Patches]               [Skip & Flag]
                                    bottom-to-top                 Set node, no commit
                                             │
                                             ▼
                                  [GitHub Commit File]
                                  PUT updated demo-post.md
                                             │
                                             ▼
                                   [Get QA Report SHA]
                                   GET qa_report.json (ignore 404)
                                             │
                                             ▼
                                  [Prepare QA Report]
                                  Buffer.from().toString('base64')
                                             │
                                             ▼
                               [GitHub Commit QA Report]
                               PUT posts/qa_report.json


### Design Principles

**Single-responsibility nodes.** Each code node does exactly one transformation decode, parse, validate, apply. This makes individual nodes testable in the n8n editor and keeps failures localized.

**Config node at the top.** `Set: Config` holds `owner`, `repo`, `filePath`, and `branch` as named values. Changing the target file or repo requires touching exactly one node.

**Reset node for demo reliability.** The first two nodes after the trigger GET the current SHA of `demo-post.md` and PUT the hardcoded original broken version back. This means every single button press starts from a known state, the workflow is self-resetting.

**Data is threaded through node references, not re-fetched.** Because the QA Agent sits between `Decode & Prepare Lines` and `Parse QA JSON`, its response overwrites `$json` for downstream nodes. To avoid losing the original file metadata (SHA, line array, numbered content), `Parse QA JSON` explicitly reads from `$('Decode & Prepare Lines').first().json` and re-attaches those fields to its own output. This keeps all downstream nodes working off a single, complete item without any Merge nodes or extra GET calls.

---

## Integration Fluency

### GitHub Integration (real API calls)

All GitHub interactions use the Contents API directly over HTTP Request nodes — no GitHub node type, which keeps the integration explicit and inspectable.

- **GET** `/repos/{owner}/{repo}/contents/{path}` — returns base64-encoded content and the file's SHA. Both are extracted and passed downstream.
- **PUT** `/repos/{owner}/{repo}/contents/{path}` — creates or updates a file. Requires the current SHA if the file already exists; omitting it on an existing file returns a 422.
- The `Authorization: token <PAT>` credential is stored as an n8n HTTP Header Auth credential and injected at the node level, not hardcoded in any expression.

The QA report commit is handled separately from the blog post commit, with its own GET-before-PUT to retrieve the existing SHA (or tolerate a 404 on first run). This keeps the report commit idempotent regardless of run history.

**Body serialisation detail.** n8n's HTTP Request node in `Using JSON` mode expects the body expression to resolve to a plain object, not a JSON string. `Prepare QA Report` outputs `qaReportBody` as a pre-serialised string (so it can be inspected in the editor). `GitHub Commit QA Report` therefore uses `{{ JSON.parse($json.qaReportBody) }}` as its JSON body expression, which deserialises the string back into an object before the node serialises it again for the HTTP call. This ensures `Content-Type: application/json` is set and GitHub receives `message`, `content`, and `sha` as proper top-level fields.

### OpenRouter / Claude API (real calls)

Both AI agents call `https://openrouter.ai/api/v1/chat/completions` using the standard OpenAI-compatible interface. The model is `anthropic/claude-sonnet-4-6`.

One practical issue encountered: the OpenRouter free key has a per-request token budget. Without an explicit `max_tokens` cap, the API returns a 402 because Claude Sonnet's default context ceiling exceeds the available credits. Setting `max_tokens: 2000` in both agent payloads resolves this — it's the right fix anyway since the blog post and patch list are both short.

---

## AI / LLM Usage

### Prompt Design

Both agents receive the blog post with **line numbers prepended** to every line:

```
1: # Why Artificial Intelligence is Changing Everything in 2024
2:
3: Artificial intelligence is a technology that has became very important...
```

This is the critical design decision. Without line numbers, the model would have no reliable way to reference specific locations for patches, and the editor would have to guess at positions. With them, the QA agent can return `"line_range": [3, 3]` and the editor can return `"line_range": [3, 3], "operation": "replace", "new_text": "..."` — both grounded in the same coordinate system.

### QA Agent Persona and Output Contract

The QA agent is instructed to act as a professional blog post editor with a specific output contract — a JSON array with no surrounding text, no markdown fences:

```json
[
  { "line_range": [3, 3], "issue_type": "grammar", "description": "..." },
  { "line_range": [6, 7], "issue_type": "clarity", "description": "..." }
]
```

The `issue_type` is constrained to an enum (`grammar`, `spelling`, `tone`, `clarity`, `structure`, `formatting`) to keep the output structured and to make the QA report filterable downstream.

In testing, the QA agent found **30 issues** in the demo post — grammar errors, subject-verb agreement problems, unclear hedging language, hyperbolic tone, and a factual overreach about AI accuracy percentages.

### Editor Agent Persona and Output Contract

The editor receives the full numbered post *and* the QA issues array. This is deliberate: the editor needs both to produce patches that fix the right lines for the right reasons. Its output contract is stricter — a JSON array of patch operations only:

```json
[
  { "line_range": [1, 1], "operation": "replace", "new_text": "..." },
  { "line_range": [15, 15], "operation": "delete" }
]
```

Temperature is set lower for the editor (0.1 vs 0.2 for QA) because patch generation is a deterministic transformation task, not a creative one. Hallucinated line numbers or extra prose in the output would corrupt the file.

```js
const cleaned = raw.replace(/^```json\n?/, '').replace(/\n?```$/, '');
```

---

## Robustness and Pragmatism

### Conflict-Free Patch Application

This is the core technical challenge in the workflow. If you apply a patch that replaces 3 lines with 2, every line number below that point shifts down by 1. A second patch targeting line 20 would now hit line 19.

The fix: sort patches **descending by start line** before applying. By working bottom-to-top, each patch operates on an untouched region of the array. Later patches (lower line numbers) are unaffected by earlier ones.

```js
validPatches.sort((a, b) => b.line_range[0] - a.line_range[0]);
```

### Out-of-Bounds Filtering

Before applying, every patch is validated:
- `start >= 1` and `end <= totalLines`
- `start <= end`
- `operation` is `replace`, `insert_after`, or `delete`

Patches that fail validation are counted as `skippedCount` and included in the commit message. This means a partially corrupt editor response still produces a useful commit rather than crashing the workflow.

### Fallback Branch

If the editor returns no valid patches — empty array, parse failure, or all patches OOB — the IF node routes to a **Skip & Flag** Set node that records the reason. Nothing is committed to GitHub. This is a deliberate no-op rather than an error; in some contexts a file may genuinely need no edits, and writing an empty commit is worse than writing none.

### Idempotency

The workflow is safe to run multiple times:
- **Blog post:** each run resets the file to the original (via the Reset to Original node), so the QA pipeline always has a known starting point.
- **QA report:** the workflow GETs the existing `qa_report.json` SHA before each PUT. If the file exists, the SHA is included (preventing 422); if not (first run), it's omitted (creating the file). The report is always overwritten with the current run's data — it is not appended.
- **GitHub SHA tracking:** the SHA captured at GET time is used in the PUT. If the file was modified externally between those two steps, GitHub will reject the PUT with a 409 conflict. This is the correct behavior — it surfaces a race condition rather than silently overwriting external changes.

### Known Limitations and Trade-offs

**Token budget.** `max_tokens: 2000` is enough for a short blog post but would be too small for longer documents. A production version would calculate an appropriate ceiling based on input length, or use streaming.

**No retry logic.** If OpenRouter returns a transient 5xx, the workflow fails. An exponential backoff retry loop (using n8n's built-in retry or a Loop node) would fix this for production use.

**Patches don't overlap, but they might conflict semantically.** The validator checks line ranges don't overlap, but it's possible for two patches to produce semantically incoherent text when both apply to adjacent lines. A smarter validator would check for semantic overlap too.

**Committing directly to main.** For a real blog workflow, the correct pattern is: create a branch, apply edits, open a pull request with the QA report as the PR description, and let a human approve before merging. The current setup commits directly to main for demo simplicity.

**Webhook registration.** n8n's public API activation endpoint marks a workflow active in the database but doesn't register webhooks in the process's in-memory registry — that only happens when you toggle the workflow from the UI. The workflow therefore uses a Manual Trigger for one-click execution from the editor, with the Webhook Trigger available once the in-memory registry is populated via the UI toggle.

---

## Edge Case Thinking

A full breakdown is in [`EDGE_CASES.md`](./EDGE_CASES.md). This section captures the reasoning behind the approach.

### The core insight: LLM output is an untrusted API

Most automation workflows treat node output as reliable. An HTTP response either has the right schema or the workflow errors out. With LLM agents in the pipeline, the failure surface is different: the model will almost always return *something*, but that something may deviate from the contract in subtle, silent ways. A response that is technically valid JSON but structured as `{"issues": [...]}` instead of `[...]` will pass through `JSON.parse` cleanly and then silently discard all data at the `!Array.isArray` check. No error. No warning. A QA report that says zero issues.

This means defensive parsing is not optional — it is the core correctness concern of any agent-in-the-middle workflow.

### How failure modes were identified

Each layer of the pipeline was treated as an independent trust boundary:

1. **GitHub API** — well-behaved but not infallible. The critical insight here is that the SHA from the initial GET is the only concurrency guard in the pipeline. If it is missing or stale, the PUT either silently overwrites external changes (if the SHA is absent) or fails with a 409 (if it is stale). Both must be caught before they reach the commit step.

2. **Agent output parsing** — the highest-risk layer. The failure modes were enumerated by asking: *what are all the ways a language model can technically comply with a JSON instruction without actually returning what you expect?* The list is longer than intuition suggests: trailing prose, non-`json` fence specifiers, object-wrapped arrays, partial truncation mid-element, echoed line-number prefixes in replacement text. Each is a realistic Claude behaviour under specific prompt or token-pressure conditions.

3. **Patch application logic** — the subtlest layer. The line-number coordinate system the two agents share is a snapshot of the file at a single point in time. Any patch that inserts or deletes lines changes the coordinate system for every subsequent patch. The bottom-to-top sort solves the sequential drift problem. But it does not solve the *overlap* problem: two patches targeting `[5, 8]` and `[7, 10]` both pass OOB validation individually, yet applying both produces silently corrupt content. The fix — an overlap sweep after the descending sort — requires understanding that the sort is a prerequisite for the overlap check, not a replacement for it.

4. **Data threading** — an n8n-specific concern. Because the QA Agent sits in the middle of the pipeline, its response overwrites `$json` for all downstream nodes. The original file metadata (SHA, line array, numbered content) has to be re-attached explicitly using a named node reference. This reference is a string — if the node is renamed during editing, the reference silently returns null and the next destructuring throws a TypeError with no useful context. Guarding this reference with an optional chain and a descriptive error message is the difference between a debuggable failure and a 20-minute mystery.

### Priority rationale

**P0 (silent corruption)** is worse than a crash. A crash stops the pipeline and produces an error the operator can act on. Silent corruption produces a committed file that looks correct but isn't, or a QA report that shows zero issues when there are thirty. The two most dangerous P0 cases in this pipeline are:

- **Truncated JSON recovery** — without it, hitting `max_tokens` mid-array looks identical to finding no issues. The QA report is confidently wrong.
- **Overlapping patch detection** — without it, a dense paragraph with multiple issues gets patched with silently corrupted content. The committed file looks plausible but is wrong in ways that are hard to notice without diffing line-by-line.

**P1 (wrong but doesn't crash)** cases are mostly agent output deviations. The `extractJSON` helper — which finds the outermost `[`…`]` boundaries regardless of surrounding prose — handles four P1 cases at once (A3, A4, A5, A7 partially). A single well-designed extraction function is more robust than a sequence of specific-case regexes, because it degrades gracefully under combinations of deviations.

**P2 (nice-to-have)** cases are logged not because they are likely, but because they are the failure modes that would be hard to diagnose in production. Floating-point line numbers and zero-indexed ranges from an agent suggest a systematic prompt failure, not a one-off glitch. Surfacing them in `skippedReasons` in the QA report turns a silent pattern into an observable signal.

---

## Running the Workflow

1. Open **https://n8n-97i5.onrender.com**
2. Open **"AI Blog QA Pipeline"**
3. Click **▶ Test workflow**

The workflow will: reset the blog post → QA it → edit it → commit the fixed version → commit the QA report.

Check results at: https://github.com/DamianBob/ai-qa-blog-demo

---

## Credentials

| Credential | Type | Used by |
|---|---|---|
| `GitHub PAT` | HTTP Header Auth (`Authorization: token …`) | Get Reset SHA, Reset to Original, GitHub GET File, GitHub Commit File, Get QA Report SHA, GitHub Commit QA Report |
| `OpenRouter API Key` | HTTP Header Auth (`Authorization: Bearer …`) | QA Agent, Editor Agent |
