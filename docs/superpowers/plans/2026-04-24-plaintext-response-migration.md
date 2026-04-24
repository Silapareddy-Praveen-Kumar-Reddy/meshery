# Meshery Server Plain-Text Response Migration

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Eliminate every plain-text HTTP response Meshery Server emits where a JSON response is expected by clients (UI / `mesheryctl` / other repos in the ecosystem), so that structured errors carry MeshKit metadata (code, severity, cause, remediation) on the wire and RTK Query's default baseQuery can parse them without crashing.

**Architecture:** Build on the existing `writeJSONError` helper at [utils.go:14](server/handlers/utils.go) by introducing a MeshKit-aware companion `writeMeshkitError(w, err, status)` that serializes the full `errors.Error` structure to JSON. Migrate all 513 `http.Error` call sites across 43 files and 13 raw `fmt.Fprint`/`w.Write` plain-text sites to the new helpers. Add a `forbidigo` lint guard to prevent regression. Update internal docs and (cross-repo) the UI baseQuery in `meshery/schemas` to surface the richer error shape.

**Tech Stack:**
- Go 1.21+, `net/http`, `encoding/json`
- MeshKit errors: `github.com/meshery/meshkit/errors`
- Gorilla Mux router
- golangci-lint (`forbidigo` linter) for regression guard
- UI: RTK Query (consumer), baseQuery from `@meshery/schemas`

**Scope source of truth (catalogued during analysis):**
- 513 `http.Error()` call sites across 43 files in `server/handlers/` + `server/models/*provider*.go`
  - 405 wrap a MeshKit error (code present, but body is plain text that strips the code)
  - 89 pass a bare string (no code at all — worst offenders)
  - 19 other (inline errors, legacy)
- 13 raw plain-text writes that aren't `http.Error` (Category A)
- ~8 `w.Write([]byte("{}"))` sites missing `Content-Type: application/json` (Category B)
- Legitimately excluded (Category C): SSE streams, k8s healthz probes, binary/tar downloads, HTML UI, HTTP redirects, GraphQL enum `MarshalJSON` impls

---

## File Structure

**New / modified foundation files:**

| File | Purpose |
|------|---------|
| `server/handlers/utils.go` | Extend with `writeMeshkitError(w, err, status)` and `writeJSONMessage(w, payload, status)`; keep `writeJSONError` for bare strings |
| `server/handlers/utils_test.go` | Add tests for the new helpers — Content-Type, status, body shape, MeshKit metadata serialization |
| `server/handlers/response.go` *(new)* | *(optional)* If `utils.go` grows unwieldy, split response helpers here. Only create if Task 1 pushes `utils.go` past ~200 lines |
| `.golangci.yml` | Add `forbidigo` rule banning `http.Error` in `server/handlers/` and `server/models/` (with allowlist for legitimate uses) |
| `docs/pages/project/contributing/error-contract.md` *(new)* | Document the server error response shape and the migration rationale |

**Modified handler files (in priority order — migration waves described below):**

| Wave | Files | `http.Error` count | Reason for priority |
|------|-------|--------------------|---------------------|
| Wave 0 (Phase 3) | `design_engine_handler.go`, `load_test_handler.go`, `k8sconfig_handler.go`, `load_test_preferences_handler.go`, `performance_profiles_handler.go`, `meshery_pattern_handler.go` (one site), `component_handler.go` (one site), `database_handlers.go`, `user_handler.go` (success paths) | 13 raw sites | These are raw `fmt.Fprint`/`w.Write` plain-text writes with no MeshKit code at all |
| Wave 0b (Phase 4) | `prometheus_handlers.go`, `grafana_handlers.go`, `mesh_ops_handlers.go`, `k8sconfig_handler.go` | ~8 sites | `w.Write([]byte("{}"))` with missing `Content-Type: application/json` |
| Wave 1 (Phase 5) | `middlewares.go` (4), `server/models/remote_provider.go` (5), `server/models/default_local_provider.go` (2) | 11 | Middleware and provider paths run on every request — fix first |
| Wave 2 (Phase 6a) | `keys_handler.go` (2), `schedule_handlers.go` (6), `schema_handlers.go` (4), `session_sync_handler.go` (1), `server_events_configuration_handler.go` (1), `server_spec_handler.go` (1), `provider_handler.go` (4), `organization_handler.go` (2), `common_handlers.go` (6), `extensions.go` (3) | 30 | Small files — good warm-up for reviewers |
| Wave 3 (Phase 6b) | `user_handler.go` (16), `workspace_handlers.go` (24), `environments_handlers.go` (15), `credentials_handlers.go` (11), `connections_handlers.go` (28), `contexts_handler.go` (8), `database_handlers.go` (10), `relationship_handlers.go` (3), `validate.go` (3), `remote_components_handler.go` (?), `performance_profiles_handler.go` (5), `fetch_results_handler.go` (14), `mesh_ops_handlers.go` (20), `meshsync_handler.go` (7), `evaluation_tracker.go` (?) | ~170 | Mid-size files |
| Wave 4 (Phase 6c) | `meshery_pattern_handler.go` (69), `meshery_application_handler.go` (35), `prometheus_handlers.go` (30), `grafana_handlers.go` (29), `meshery_filter_handler.go` (25), `component_handler.go` (22), `load_test_preferences_handler.go` (20), `load_test_handler.go` (18), `load_test_meshes_handler.go` (1), `fetch_results_handler.go` (14), `k8sconfig_handler.go` (13), `component_generation.go` (3), `component_generator_helper.go` (2), `design_engine_handler.go` (7), `design_import.go` (5), `policy_relationship_handler.go` (6), `events_streamer.go` (22 non-SSE), `middlewares.go` if any remain | ~300+ | Large files — should be multiple PRs each |

**Cross-repo touchpoints (called out in the plan, executed as separate PRs):**

| Repo | File(s) | Change |
|------|---------|--------|
| `meshery/schemas` | Files backing `@meshery/schemas/mesheryApi` (the RTK Query baseQuery) | Update error transform to surface MeshKit `code`, `severity`, `probable_cause`, `suggested_remediation` fields on the RTK Query `error` object |
| `meshery/docs` | `docs/pages/reference/contributing/error-codes.mdx` or equivalent | Reference the new response contract from the user-facing contributing docs |

---

## Orientation for the engineer

Read this section before starting.

**What breaks today.** Go's `http.Error(w, msg, status)` sets `Content-Type: text/plain; charset=utf-8` and writes the message as plain text. Meshery UI uses RTK Query; its default baseQuery from `@meshery/schemas/mesheryApi` dispatches on `Content-Type`. When it sees `text/plain` (or any non-JSON type) the default code path tries to read the body as JSON anyway on certain error branches, and crashes with `SyntaxError: Unexpected token 'S', "Status Cod"... is not valid JSON`. That masks the real error and breaks the standard RTK Query error shape that components rely on.

**Why we can't just "switch the UI to read text on errors."** The server already returns JSON for many success responses and a few error responses. The inconsistency is the actual defect. Standardizing on JSON means:
1. Clients never have to probe Content-Type before parsing.
2. Error codes (MeshKit's `meshery-server-XXXX`) survive the network boundary, which is what lets `mesheryctl`, Kanvas, Meshery Cloud, and the UI display consistent, actionable messages.
3. Future structured-error tooling (docs generation, error telemetry, automated remediation links) has a wire-stable input.

**Why we can't just do a one-line `s/http.Error/writeJSONError/`.** Three reasons:
1. `writeJSONError` as it exists today only takes a bare `string` message. 405 of the 513 call sites wrap a MeshKit `*errors.Error` — collapsing that to its `.Error()` string throws away the code, cause, and remediation. We need a new helper that preserves the structure.
2. 89 call sites pass a bare string (no MeshKit error at all). These need new MeshKit codes introduced alongside the migration. Don't just JSON-encode the bare string — promote it to a proper error.
3. Some call sites are actually fine as plain text (SSE, healthz, binary downloads). Don't touch those.

**The target wire shape.** Error bodies after migration:
```json
{
  "error": "Human-readable message (MeshKit ShortDescription)",
  "code": "meshery-server-1033",
  "severity": "ERROR",
  "probable_cause": ["..."],
  "suggested_remediation": ["..."],
  "long_description": ["..."]
}
```
Bare-string fallback (for messages with no MeshKit error):
```json
{ "error": "Human-readable message" }
```

**Existing helper we're extending.** [server/handlers/utils.go:14-19](server/handlers/utils.go) defines `writeJSONError(w, message, status)`. Tests live at [server/handlers/utils_test.go](server/handlers/utils_test.go). Keep that function signature stable — 4 existing callers depend on it.

**MeshKit accessors we'll use** (from `github.com/meshery/meshkit/errors`):
- `errors.GetCode(err) string` — returns `"meshery-server-1033"` or the `NoneString` default
- `errors.GetSeverity(err) Severity` — returns one of `None`, `Emergency`, `Alert`, `Critical`, `Error`, ...
- `errors.GetSDescription(err) string` — joined ShortDescription
- `errors.GetLDescription(err) string` — joined LongDescription
- `errors.GetCause(err) string` — joined ProbableCause
- `errors.GetRemedy(err) string` — joined SuggestedRemediation

These work on both `*Error` and `*ErrorV2`. They never panic on nil.

**Testing philosophy.** Every new helper gets a unit test with assertions on Content-Type, status code, and JSON parseability. Per-handler full regression tests are out of scope — there are 513 sites, we can't hand-test them all. The `forbidigo` lint guard (Phase 2) is the structural guarantee. Spot-check tests on representative endpoints in each wave.

---

## Phase 1 — Foundation: helpers + tests + docs

### Task 1: Extend response helpers

**Files:**
- Modify: `server/handlers/utils.go`
- Test: `server/handlers/utils_test.go`

- [ ] **Step 1: Write the failing test for `writeMeshkitError` (MeshKit error serialization)**

Add to `server/handlers/utils_test.go`:

```go
func TestWriteMeshkitError_SerializesMeshKitStructure(t *testing.T) {
	rec := httptest.NewRecorder()

	// Use an actual MeshKit error from the server's catalog.
	err := ErrGetResult(fmt.Errorf("record not found"))

	writeMeshkitError(rec, err, http.StatusNotFound)

	resp := rec.Result()
	defer func() { _ = resp.Body.Close() }()

	if resp.StatusCode != http.StatusNotFound {
		t.Fatalf("expected status %d, got %d", http.StatusNotFound, resp.StatusCode)
	}
	if ct := resp.Header.Get("Content-Type"); ct != "application/json; charset=utf-8" {
		t.Errorf("expected Content-Type application/json, got %q", ct)
	}
	if nosniff := resp.Header.Get("X-Content-Type-Options"); nosniff != "nosniff" {
		t.Errorf("expected X-Content-Type-Options: nosniff, got %q", nosniff)
	}

	var decoded struct {
		Error                string   `json:"error"`
		Code                 string   `json:"code"`
		Severity             string   `json:"severity"`
		ProbableCause        []string `json:"probable_cause"`
		SuggestedRemediation []string `json:"suggested_remediation"`
		LongDescription      []string `json:"long_description"`
	}
	if decodeErr := json.NewDecoder(resp.Body).Decode(&decoded); decodeErr != nil {
		t.Fatalf("body did not parse as JSON: %v", decodeErr)
	}
	if decoded.Code != ErrGetResultCode {
		t.Errorf("expected code %q, got %q", ErrGetResultCode, decoded.Code)
	}
	if decoded.Error == "" {
		t.Errorf("expected non-empty error message")
	}
}

func TestWriteMeshkitError_NilFallsBackToGenericMessage(t *testing.T) {
	rec := httptest.NewRecorder()
	writeMeshkitError(rec, nil, http.StatusInternalServerError)

	var decoded map[string]interface{}
	if err := json.NewDecoder(rec.Body).Decode(&decoded); err != nil {
		t.Fatalf("body did not parse as JSON: %v", err)
	}
	if decoded["error"] == nil || decoded["error"] == "" {
		t.Errorf("expected non-empty error field for nil input")
	}
}

func TestWriteMeshkitError_NonMeshkitErrorStillJSON(t *testing.T) {
	rec := httptest.NewRecorder()
	writeMeshkitError(rec, fmt.Errorf("plain stdlib error"), http.StatusBadRequest)

	if ct := rec.Header().Get("Content-Type"); ct != "application/json; charset=utf-8" {
		t.Errorf("expected JSON Content-Type even for non-MeshKit errors, got %q", ct)
	}

	var decoded map[string]interface{}
	if err := json.NewDecoder(rec.Body).Decode(&decoded); err != nil {
		t.Fatalf("body did not parse as JSON: %v", err)
	}
	if decoded["error"] != "plain stdlib error" {
		t.Errorf("expected error field to contain original message, got %v", decoded["error"])
	}
}
```

Add the required imports to the existing `import` block: `"fmt"` (already present is `"encoding/json"`, `"net/http"`, `"net/http/httptest"`, `"testing"`).

- [ ] **Step 2: Run the tests to verify they fail**

Run: `cd server && go test ./handlers/ -run TestWriteMeshkitError -v`
Expected: FAIL with `undefined: writeMeshkitError`

- [ ] **Step 3: Implement `writeMeshkitError` in `server/handlers/utils.go`**

Modify the imports at the top of `server/handlers/utils.go` to add MeshKit errors:

```go
package handlers

import (
	"encoding/json"
	"errors"
	"net/http"
	"strconv"

	meshkiterrors "github.com/meshery/meshkit/errors"
)
```

Then add below the existing `writeJSONError` (keep that function unchanged):

```go
// errorResponse is the wire shape for all non-2xx responses from Meshery Server.
// Fields mirror github.com/meshery/meshkit/errors.Error; omitempty keeps the
// body small for bare-string errors that carry no MeshKit metadata.
type errorResponse struct {
	Error                string   `json:"error"`
	Code                 string   `json:"code,omitempty"`
	Severity             string   `json:"severity,omitempty"`
	ProbableCause        []string `json:"probable_cause,omitempty"`
	SuggestedRemediation []string `json:"suggested_remediation,omitempty"`
	LongDescription      []string `json:"long_description,omitempty"`
}

// writeMeshkitError writes a JSON error response that preserves MeshKit error
// metadata (code, severity, probable cause, remediation) when err is (or wraps)
// a *meshkiterrors.Error or *meshkiterrors.ErrorV2. Non-MeshKit errors still
// produce a JSON body — they just carry only the .Error() string, matching
// writeJSONError's shape so clients never see plain text from this package.
//
// Prefer this over http.Error for every handler error path. RTK Query's
// baseQuery dispatches on Content-Type and crashes on plain-text bodies that
// happen to start with a letter (e.g. "Status Code: 404 ...").
func writeMeshkitError(w http.ResponseWriter, err error, status int) {
	w.Header().Set("Content-Type", "application/json; charset=utf-8")
	w.Header().Set("X-Content-Type-Options", "nosniff")
	w.WriteHeader(status)

	resp := errorResponse{}
	if err == nil {
		resp.Error = http.StatusText(status)
		_ = json.NewEncoder(w).Encode(resp)
		return
	}

	resp.Error = err.Error()

	// Populate MeshKit fields only when the error carries them. GetCode etc.
	// return the "None" sentinel for non-MeshKit errors; treat that as absent.
	if code := meshkiterrors.GetCode(err); code != "" && code != "None" {
		resp.Code = code
		resp.Severity = severityString(meshkiterrors.GetSeverity(err))
		if short := meshkiterrors.GetSDescription(err); short != "" && short != "None" {
			// Use ShortDescription as the user-facing `error` when available —
			// err.Error() on a MeshKit error concatenates every field with pipes.
			resp.Error = short
		}
		if long := meshkiterrors.GetLDescription(err); long != "" && long != "None" {
			resp.LongDescription = []string{long}
		}
		if cause := meshkiterrors.GetCause(err); cause != "" && cause != "None" {
			resp.ProbableCause = []string{cause}
		}
		if remedy := meshkiterrors.GetRemedy(err); remedy != "" && remedy != "None" {
			resp.SuggestedRemediation = []string{remedy}
		}
	}

	_ = json.NewEncoder(w).Encode(resp)
}

// severityString converts a MeshKit Severity enum to the string label used on
// the wire. Kept here (not in MeshKit) because MeshKit's Severity.String is
// not yet exported in all versions we pin.
func severityString(s meshkiterrors.Severity) string {
	switch s {
	case meshkiterrors.Emergency:
		return "EMERGENCY"
	case meshkiterrors.Alert:
		return "ALERT"
	case meshkiterrors.Critical:
		return "CRITICAL"
	case meshkiterrors.Fatal:
		return "FATAL"
	default:
		return "ERROR"
	}
}

// writeJSONMessage encodes an arbitrary payload as JSON with the given status
// code. Use for success responses that currently write a bare string (e.g.
// "Database reset successful") — promote them to a structured message.
func writeJSONMessage(w http.ResponseWriter, payload interface{}, status int) {
	w.Header().Set("Content-Type", "application/json; charset=utf-8")
	w.WriteHeader(status)
	_ = json.NewEncoder(w).Encode(payload)
}
```

Before pasting, verify `meshkiterrors.Fatal` exists in the pinned MeshKit version. If it doesn't, remove that case and default covers it. Check via:

```bash
go doc github.com/meshery/meshkit/errors Severity
```

If any severity constant name differs, adjust the switch. The `errors` stdlib import in the helpers file is used implicitly by future tasks — if linters flag it as unused at this stage, drop it for now and re-add in Task 3.

- [ ] **Step 4: Run the tests to verify they pass**

Run: `cd server && go test ./handlers/ -run TestWriteMeshkitError -v`
Expected: PASS (3 tests)

Also run the existing writeJSONError tests to confirm no regression:

Run: `cd server && go test ./handlers/ -run TestWriteJSONError -v`
Expected: PASS (2 existing tests)

- [ ] **Step 5: Run full handlers package tests**

Run: `cd server && go test ./handlers/...`
Expected: PASS. If anything fails that didn't fail before Step 3, the cause is likely a name collision or unused import — fix before committing.

- [ ] **Step 6: Verify the server still builds**

Run: `cd server && go build ./...`
Expected: no output, exit 0.

- [ ] **Step 7: Commit**

```bash
cd /Users/l/code/meshery/.claude/worktrees/affectionate-curran-08a480
git add server/handlers/utils.go server/handlers/utils_test.go
git commit -s -m "$(cat <<'EOF'
feat(server): add writeMeshkitError + writeJSONMessage response helpers

Introduces writeMeshkitError(w, err, status) which serializes MeshKit
error metadata (code, severity, probable cause, remediation) to a JSON
body, and writeJSONMessage(w, payload, status) for success responses
that currently write bare strings. Both set Content-Type: application/json
so RTK Query's default baseQuery can parse them.

Prepares the ground for the sweep replacing http.Error across
server/handlers/ and server/models/*provider*.go — clients of Meshery
Server have been seeing "Unexpected token 'S', \"Status Cod\"..."
JSON parse errors from the UI's RTK Query layer because the server
responds with text/plain on many error paths.

Preserves the existing writeJSONError signature; existing callers
(workspace_handlers, environments_handlers) are unaffected.
EOF
)"
```

---

### Task 2: Document the error response contract

**Files:**
- Create: `docs/pages/project/contributing/error-contract.md`

- [ ] **Step 1: Check how neighboring docs are structured**

Run: `ls docs/pages/project/contributing/` and skim one file, e.g. `contributing-error.md` if it exists, for the front matter and heading conventions. Match whatever the existing pages use (Jekyll front matter, Hugo, etc.).

- [ ] **Step 2: Write the contract document**

Create `docs/pages/project/contributing/error-contract.md` with (adjust front matter to match neighbors):

```markdown
---
layout: page
title: HTTP Error Response Contract
permalink: project/contributing/error-contract
abstract: How Meshery Server responds to error conditions over HTTP, and how clients should parse those responses.
language: en
type: project
category: contributing
---

# HTTP Error Response Contract

Every non-2xx HTTP response from Meshery Server carries a JSON body with
`Content-Type: application/json; charset=utf-8`. Clients should parse the body
as JSON before surfacing errors to users.

## Shape

```json
{
  "error": "Human-readable short description",
  "code": "meshery-server-1033",
  "severity": "ERROR",
  "probable_cause": ["Connection to the remote provider timed out."],
  "suggested_remediation": ["Verify that Meshery Cloud is reachable."],
  "long_description": ["Full technical details suitable for logs."]
}
```

### Fields

| Field | Required | Notes |
|-------|----------|-------|
| `error` | yes | User-facing message. For MeshKit errors, this is the ShortDescription. |
| `code` | when available | MeshKit error code (e.g. `meshery-server-1033`). Stable across releases. Use for telemetry, i18n lookup, and programmatic handling. |
| `severity` | when available | One of `EMERGENCY`, `ALERT`, `CRITICAL`, `FATAL`, `ERROR`. |
| `probable_cause` | optional | Array of strings. |
| `suggested_remediation` | optional | Array of strings. Surface to users when present. |
| `long_description` | optional | Array of strings. Suitable for developer logs; may contain stack-style detail. |

Fields marked "when available" are omitted (via `omitempty`) for errors that
originated outside the MeshKit error catalog.

## Client contract

- Do not rely on plain-text error bodies — they are always JSON.
- When `code` is present, prefer it over string matching on `error`.
- When `suggested_remediation` is non-empty, surface it alongside `error`.
- When the body is not valid JSON, treat the response as a bug and report
  the offending endpoint; do not attempt a text fallback.

## Producing errors in handlers

Use `writeMeshkitError(w, err, status)` in `server/handlers/utils.go`:

```go
if err != nil {
    h.log.Error(ErrGetResult(err))
    writeMeshkitError(w, ErrGetResult(err), http.StatusNotFound)
    return
}
```

For bare-string errors without a MeshKit code, use `writeJSONError(w, msg, status)`.
Every bare-string error is a candidate for promotion to a MeshKit error —
prefer adding a code when fixing an adjacent bug.

Do not use `http.Error` in handlers or provider code. It writes
`Content-Type: text/plain` and strips MeshKit metadata, which crashes
RTK Query's default baseQuery on the UI.

Legitimate exceptions (enforced by `.golangci.yml` allowlist):
- SSE stream handlers (`Content-Type: text/event-stream`)
- Kubernetes healthz probes (plain text is the probe contract)
- Binary/tar/YAML downloads
- HTTP redirects (no body)
```

- [ ] **Step 3: Commit**

```bash
git add docs/pages/project/contributing/error-contract.md
git commit -s -m "docs(server): add HTTP error response contract"
```

---

## Phase 2 — Regression guard

### Task 3: Add `forbidigo` lint rule against `http.Error` in handlers and provider code

**Files:**
- Modify: `.golangci.yml` (create if absent)
- Verify: Current lint config location via `ls -la .golangci* server/.golangci* 2>/dev/null`

- [ ] **Step 1: Locate the existing golangci-lint config**

Run:
```bash
ls -la .golangci* 2>/dev/null; ls -la server/.golangci* 2>/dev/null
```

Record which file (if any) is authoritative. If none exists, create `.golangci.yml` at repo root. If `server/.golangci.yml` exists but root doesn't, modify `server/.golangci.yml` instead. Update the paths in the steps below accordingly.

- [ ] **Step 2: Add the `forbidigo` linter to the config**

If the config file doesn't yet enable `forbidigo`, add it under `linters.enable`. Then under `linters-settings.forbidigo`, add:

```yaml
linters-settings:
  forbidigo:
    forbid:
      - p: '^http\.Error$'
        msg: "Use writeMeshkitError(w, err, status) or writeJSONError(w, msg, status) from server/handlers/utils.go. http.Error writes Content-Type: text/plain and breaks RTK Query's default baseQuery. See docs/pages/project/contributing/error-contract.md."
    exclude-godoc-examples: true
    analyze-types: true

issues:
  exclude-rules:
    # Legitimate plain-text responders — keep http.Error here if used.
    - path: 'server/handlers/k8s_healthz_handler\.go'
      linters: [forbidigo]
    - path: 'server/handlers/events_streamer\.go'
      linters: [forbidigo]
    - path: 'server/handlers/load_test_handler\.go'
      linters: [forbidigo]
      # TODO: narrow to SSE section only once Phase 6 migrates the non-SSE paths.
    # UI handler serves static HTML; any http.Error there relates to
    # asset resolution, not API surface — keep off the hook.
    - path: 'server/handlers/ui_handler\.go'
      linters: [forbidigo]
```

If `forbidigo` is already enabled with existing rules, append the new `p`/`msg` pair to the existing list rather than replacing it.

- [ ] **Step 3: Run the linter to confirm it fires**

Run:
```bash
cd server && golangci-lint run --disable-all -E forbidigo ./handlers/... ./models/...
```

Expected: lots of violations (we haven't migrated yet). Confirm at least one violation per file with an `http.Error`. If the linter returns zero violations, the config didn't load — fix config discovery before proceeding.

- [ ] **Step 4: Don't fail CI yet — make the rule advisory**

In the same config file, add under `issues`:

```yaml
issues:
  new: false  # report all pre-existing violations
  # ...
```

And in CI (typically `.github/workflows/*.yml`), we do NOT want CI to start failing on all 513 existing violations mid-migration. Check the existing workflow:

Run: `grep -l golangci .github/workflows/*.yml`

Open the relevant workflow and confirm whether golangci-lint currently runs. If it does and is blocking:

Option A (recommended): Add `--new-from-rev=origin/master` to the golangci-lint invocation so only new violations fail. Existing ones are surfaced but not blocking.

Option B: Temporarily add the forbidigo check under an opt-in flag until migration completes.

Document the choice in the commit message.

- [ ] **Step 5: Commit**

```bash
git add .golangci.yml .github/workflows/
git commit -s -m "$(cat <<'EOF'
ci(server): forbid http.Error in handlers/models via forbidigo

Adds a golangci-lint forbidigo rule banning http.Error in
server/handlers/ and server/models/ — these paths must return
JSON so RTK Query's default baseQuery can parse error bodies
without crashing.

Rule is applied only to new code (--new-from-rev=origin/master)
so existing violations are surfaced as advisory while the
migration lands. Once all migration waves complete, flip the
rule to blocking in a follow-up.

Allowlists SSE, healthz, UI-static, and the SSE portion of
load_test_handler — those legitimately emit non-JSON content.
EOF
)"
```

---

## Phase 3 — Fix raw plain-text writes (Category A)

These 13 sites are NOT `http.Error` calls — they use `fmt.Fprint(f|ln)(w, ...)` or `w.Write([]byte(...))` to write plain text. Fix before the `http.Error` sweep because some of them overlap (e.g. a handler both raw-writes an error AND http.Errors elsewhere).

### Task 4: Fix raw plain-text writes in `design_engine_handler.go`

**Files:**
- Modify: `server/handlers/design_engine_handler.go`

- [ ] **Step 1: Read the current state**

Read `server/handlers/design_engine_handler.go` around lines 60-80 to confirm context of lines 65 and 76.

- [ ] **Step 2: Replace line 65 — `fmt.Fprintf(rw, "failed to read request body: %s", err)`**

Current:
```go
fmt.Fprintf(rw, "failed to read request body: %s", err)
```

Replace with:
```go
writeMeshkitError(rw, ErrRequestBody(err), http.StatusBadRequest)
return
```

(Verify `ErrRequestBody` exists in [server/handlers/error.go](server/handlers/error.go); it's code `meshery-server-1023`. If the surrounding code already has a `return` immediately after, drop the one we're adding.)

- [ ] **Step 3: Replace line 76 — `fmt.Fprintf(rw, "failed to unmarshal request body: %s", err)`**

Current:
```go
fmt.Fprintf(rw, "failed to unmarshal request body: %s", err)
```

Replace with:
```go
writeMeshkitError(rw, ErrDecoding(err, "design engine request"), http.StatusBadRequest)
return
```

Confirm `ErrDecoding` exists in `error.go` (`meshery-server-1049`). If it takes a different signature, adjust the call to match — look at other call sites of `ErrDecoding` in the codebase for the pattern.

- [ ] **Step 4: Build and test**

Run: `cd server && go build ./... && go test ./handlers/ -count=1`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add server/handlers/design_engine_handler.go
git commit -s -m "fix(server): return JSON errors from design_engine_handler

Replaces raw fmt.Fprintf plain-text writes with writeMeshkitError so
RTK Query can parse the error body. Adds proper MeshKit codes
(ErrRequestBody, ErrDecoding) where plain strings were used."
```

### Task 5: Fix raw plain-text writes in `load_test_handler.go` (non-SSE only)

**Files:**
- Modify: `server/handlers/load_test_handler.go`

- [ ] **Step 1: Read the target line**

Read `server/handlers/load_test_handler.go` around line 44.

- [ ] **Step 2: Replace line 44**

Current:
```go
fmt.Fprintf(w, "failed to read request body: %s", err)
```

Replace with:
```go
writeMeshkitError(w, ErrRequestBody(err), http.StatusBadRequest)
return
```

Do NOT touch the SSE section (lines 348, 374-376) — those are legitimate `data: ...\n\n` event-stream writes.

- [ ] **Step 3: Build, test, commit**

```bash
cd server && go build ./... && go test ./handlers/ -count=1
git add server/handlers/load_test_handler.go
git commit -s -m "fix(server): return JSON error from load_test_handler request-body read

Leaves the SSE section (Content-Type: text/event-stream) untouched."
```

### Task 6: Fix raw plain-text writes in `k8sconfig_handler.go`

**Files:**
- Modify: `server/handlers/k8sconfig_handler.go`

- [ ] **Step 1: Read lines 270-345**

Four sites: lines 275, 295, 305, 338. Read 25 lines around each to understand the handler's error model.

- [ ] **Step 2: Replace line 275 — `fmt.Fprintf(w, "failed to get the token for the user")`**

Replace with:
```go
writeMeshkitError(w, ErrRetrieveUserToken(fmt.Errorf("no token for user")), http.StatusUnauthorized)
return
```

Verify `ErrRetrieveUserToken` exists in `error.go` (`meshery-server-1050`). Adjust import of `"fmt"` if not present.

- [ ] **Step 3: Replace line 295 — `fmt.Fprintf(w, "failed to get kubernetes context for the given ID")`**

Replace with:
```go
writeMeshkitError(w, ErrInvalidKubeContext(fmt.Errorf("not found")), http.StatusNotFound)
return
```

Verify `ErrInvalidKubeContext` (`meshery-server-1097`) exists.

- [ ] **Step 4: Replace line 305 — `fmt.Fprintf(w, "failed to get kubernetes config for the user")`**

Replace with:
```go
writeMeshkitError(w, ErrInvalidKubeConfig(fmt.Errorf("no kube config for user")), http.StatusNotFound)
return
```

- [ ] **Step 5: Replace line 338 — `w.Write([]byte(http.StatusText(http.StatusAccepted)))`**

This is a success path writing "Accepted" as plain text. Promote to JSON:

Current:
```go
w.WriteHeader(http.StatusAccepted)
w.Write([]byte(http.StatusText(http.StatusAccepted)))
```

Replace with:
```go
writeJSONMessage(w, map[string]string{"status": "accepted"}, http.StatusAccepted)
```

- [ ] **Step 6: Build, test, commit**

```bash
cd server && go build ./... && go test ./handlers/ -count=1
git add server/handlers/k8sconfig_handler.go
git commit -s -m "fix(server): return JSON responses from k8sconfig_handler

Migrates four raw plain-text writes (3 error paths + 1 accepted-status
success) to writeMeshkitError / writeJSONMessage. Promotes bare error
strings to MeshKit codes (ErrRetrieveUserToken, ErrInvalidKubeContext,
ErrInvalidKubeConfig)."
```

### Task 7: Fix remaining Category A raw-text sites in one pass

**Files (one commit each, or grouped if a reviewer prefers — judgment call):**
- `server/handlers/load_test_preferences_handler.go:72`
- `server/handlers/performance_profiles_handler.go:34`
- `server/handlers/meshery_pattern_handler.go:1792`
- `server/handlers/component_handler.go:1321`
- `server/handlers/database_handlers.go:180`
- `server/handlers/user_handler.go:165, 178`

- [ ] **Step 1: `load_test_preferences_handler.go:72` — `w.Write([]byte(tid))`**

This writes a raw UUID as the response body. Promote to JSON:

Replace the block that currently ends with `w.Write([]byte(tid))` with:
```go
writeJSONMessage(w, map[string]string{"test_uuid": tid}, http.StatusOK)
```

Confirm the Content-Type was previously being set; if `w.Header().Set("Content-Type", ...)` precedes the write, remove that line since `writeJSONMessage` sets it.

- [ ] **Step 2: `performance_profiles_handler.go:34` — `fmt.Fprintf(rw, ErrRequestBody(err).Error(), err)`**

Replace with:
```go
writeMeshkitError(rw, ErrRequestBody(err), http.StatusBadRequest)
return
```

- [ ] **Step 3: `meshery_pattern_handler.go:1792` — `fmt.Fprintf(rw, "%s", err)`**

Context: read 20 lines around line 1792. This is almost certainly an error path. Replace with:
```go
writeMeshkitError(rw, err, http.StatusInternalServerError)
return
```

If context shows a more specific MeshKit wrapper (e.g. `ErrSavePattern`), use that instead.

- [ ] **Step 4: `component_handler.go:1321` — `fmt.Fprintln(rw, message)` (404 not found)**

Context: the message is something like "<model name> has not been found". Replace the block with:
```go
writeJSONError(rw, message, http.StatusNotFound)
return
```

(Use `writeJSONError` here because `message` is already a built string, not a MeshKit error. If there's a MeshKit `ErrModelNotFound` style code in scope, promote to `writeMeshkitError` instead.)

- [ ] **Step 5: `database_handlers.go:180` — `fmt.Fprint(w, "Database reset successful")`**

Success path emitting a bare string with Content-Type: application/json already set (lying to the client). Replace:

```go
w.Header().Set("Content-Type", "application/json")
fmt.Fprint(w, "Database reset successful")
```

With:

```go
writeJSONMessage(w, map[string]string{"message": "Database reset successful"}, http.StatusOK)
```

- [ ] **Step 6: `user_handler.go:165, 178` — `"Design shared"` / `"Filter shared"`**

Read lines 160-185. Each is a success path emitting a bare string. Replace each:

```go
fmt.Fprint(w, "Design shared")
```

With:

```go
writeJSONMessage(w, map[string]string{"message": "Design shared"}, http.StatusOK)
```

Same for "Filter shared".

- [ ] **Step 7: Build, test, commit (one commit per file is fine)**

```bash
cd server && go build ./... && go test ./handlers/ -count=1
git add server/handlers/load_test_preferences_handler.go server/handlers/performance_profiles_handler.go server/handlers/meshery_pattern_handler.go server/handlers/component_handler.go server/handlers/database_handlers.go server/handlers/user_handler.go
git commit -s -m "fix(server): return JSON from remaining raw plain-text response sites

Category A sweep: replaces fmt.Fprint / w.Write plain-text bodies with
writeMeshkitError / writeJSONMessage / writeJSONError across six files.
Success paths now emit {\"message\": \"...\"} instead of bare strings;
error paths carry MeshKit codes where applicable."
```

---

## Phase 4 — Category B: Content-Type fix pass

Eight `w.Write([]byte("{}"))` sites write valid JSON but don't set Content-Type. Fix to set it explicitly.

### Task 8: Set `Content-Type: application/json` on the 8 empty-object writes

**Files:**
- Modify: `server/handlers/prometheus_handlers.go` (lines 239, 271, 471)
- Modify: `server/handlers/grafana_handlers.go` (lines 144, 329)
- Modify: `server/handlers/mesh_ops_handlers.go` (line 100)
- Modify: `server/handlers/k8sconfig_handler.go` (line 232)

- [ ] **Step 1: Define a small helper for the empty-JSON-object case**

In `server/handlers/utils.go`, add:

```go
// writeJSONEmptyObject writes "{}" with Content-Type: application/json.
// Several handlers return an empty object to indicate "no data, but request
// succeeded"; this helper centralizes the pattern and guarantees the header.
func writeJSONEmptyObject(w http.ResponseWriter, status int) {
	w.Header().Set("Content-Type", "application/json; charset=utf-8")
	w.WriteHeader(status)
	_, _ = w.Write([]byte("{}"))
}
```

Add a unit test verifying Content-Type is set.

- [ ] **Step 2: Replace each call site**

For each of the 8 lines listed in the "Files" block, replace:
```go
w.Write([]byte("{}"))
```
with:
```go
writeJSONEmptyObject(w, http.StatusOK)
```

If the preceding code sets Content-Type manually or calls `w.WriteHeader`, remove those lines — the helper handles both. Verify no logic is lost (read 10 lines of context for each).

- [ ] **Step 3: Build, test, commit**

```bash
cd server && go build ./... && go test ./handlers/ -count=1
git add server/handlers/utils.go server/handlers/utils_test.go server/handlers/prometheus_handlers.go server/handlers/grafana_handlers.go server/handlers/mesh_ops_handlers.go server/handlers/k8sconfig_handler.go
git commit -s -m "fix(server): set Content-Type on empty-object JSON responses

Eight call sites wrote w.Write([]byte(\"{}\")) without setting the
Content-Type header, leaving clients to guess whether the body is
text or JSON. Introduces writeJSONEmptyObject helper to centralize
the pattern."
```

---

## Phase 5 — Middleware and provider layer (highest blast radius)

These run on every authenticated request. Migrate before the handler sweep so every downstream request benefits.

### Task 9: Migrate `server/handlers/middlewares.go`

**Files:**
- Modify: `server/handlers/middlewares.go`

- [ ] **Step 1: Catalog the sites**

Lines to fix: 163 (KubernetesMiddleware err → http.Error 500), 191 (commented-out, skip), 203 (ServiceUnavailable "Remote provider temporarily unavailable"), 213 (UNAUTHORIZED ErrGetUserDetails).

- [ ] **Step 2: Write a test for the transient-provider path**

Middleware tests should exist at [server/handlers/middlewares_test.go](server/handlers/middlewares_test.go). Add:

```go
func TestSessionInjectorMiddleware_TransientProviderErrorReturnsJSON(t *testing.T) {
	// Stub provider that returns a transient "Could not reach remote provider" error
	// from GetUserDetails. Verify the response is:
	//  - status 503
	//  - Content-Type: application/json
	//  - body parses to {"error": "Remote provider temporarily unavailable..."}
	//
	// Follow the existing test patterns in this file for constructing the handler
	// and stub provider. If no pattern exists, use httptest.NewRecorder and a
	// minimal provider mock — look at middlewares_test.go for conventions.
}
```

Fill in the body by mirroring the nearest existing middleware test. Run: `go test ./handlers/ -run TestSessionInjectorMiddleware -v`. Expected: FAIL (body is plain text today).

- [ ] **Step 3: Migrate line 163**

Replace:
```go
http.Error(w, err.Error(), http.StatusInternalServerError)
```
with:
```go
writeMeshkitError(w, err, http.StatusInternalServerError)
```

- [ ] **Step 4: Migrate line 203**

Replace:
```go
http.Error(w, "Remote provider temporarily unavailable. Please try again.", http.StatusServiceUnavailable)
```
with (promote to a proper MeshKit error — add a new code if needed):

First, add a new error code to [server/handlers/error.go](server/handlers/error.go) near `ErrReadSessionPersistor`:
```go
const ErrTransientProviderCode = "meshery-server-19XX" // assign the next free code

func ErrTransientProvider(err error) error {
	return errors.New(
		ErrTransientProviderCode,
		errors.Alert,
		[]string{"Remote provider temporarily unavailable"},
		[]string{fmt.Sprintf("Meshery Cloud (the remote provider) did not respond: %v", err)},
		[]string{"Network connectivity issue, Cloud outage, or firewall blocking egress."},
		[]string{"Retry the request. If this persists, check https://status.meshery.io and verify Meshery Server can reach Meshery Cloud."},
	)
}
```

Run `grep -n 'ErrRead.*Code' server/handlers/error.go` to find the next unused code in the sequence. Use that number in `ErrTransientProviderCode`. Assigning codes is cheap; don't reuse.

Then replace line 203 with:
```go
writeMeshkitError(w, ErrTransientProvider(err), http.StatusServiceUnavailable)
```

- [ ] **Step 5: Migrate line 213**

Replace:
```go
http.Error(w, ErrGetUserDetails(err).Error(), http.StatusUnauthorized)
```
with:
```go
writeMeshkitError(w, ErrGetUserDetails(err), http.StatusUnauthorized)
```

- [ ] **Step 6: Run tests**

```bash
cd server && go test ./handlers/ -count=1
```
Expected: the new test passes; existing middleware tests still pass.

- [ ] **Step 7: Commit**

```bash
git add server/handlers/middlewares.go server/handlers/middlewares_test.go server/handlers/error.go
git commit -s -m "$(cat <<'EOF'
fix(server): JSON error responses from auth middleware

Migrates SessionInjectorMiddleware and KubernetesMiddleware to
writeMeshkitError so 401/503/500 responses carry MeshKit metadata
instead of plain text. Adds ErrTransientProvider code for the
"Remote provider temporarily unavailable" path so clients can
programmatically distinguish it from an auth failure.

Middleware runs on every authenticated request, so this is the
single highest-impact site in the plain-text response migration.
EOF
)"
```

### Task 10: Migrate `server/models/remote_provider.go` and `server/models/default_local_provider.go`

**Files:**
- Modify: `server/models/remote_provider.go`
- Modify: `server/models/default_local_provider.go`

- [ ] **Step 1: Catalog and classify**

Run: `grep -n 'http.Error' server/models/remote_provider.go server/models/default_local_provider.go`

There should be 7 `http.Error` sites total (5 + 2). For each, read 20 lines around to determine whether it's in a request path that clients consume as JSON (most) or a redirect/callback path (where redirects dominate and the http.Error is an edge case).

- [ ] **Step 2: Migrate each site**

For MeshKit-wrapped errors: `http.Error(w, ErrXxx(err).Error(), status)` → `writeMeshkitError(w, ErrXxx(err), status)`.

For bare strings: consider promoting to a new MeshKit code (use the same pattern as Task 9 Step 4) or, for transient/debug messages that clearly need no code, `writeJSONError(w, msg, status)`.

Note: `server/models/` imports from `server/handlers/` would create a cycle. Verify the import graph. If `server/handlers/utils.go` helpers can't be called from `server/models/`, promote `writeMeshkitError` / `writeJSONError` to a new package, e.g. `server/models/httputil/httputil.go`, then import from both.

```bash
grep -l '"github.com/meshery/meshery/server/handlers"' server/models/*.go
```

If any model files already import handlers, it's safe to call the helpers directly. Otherwise, add a new file `server/models/httputil/httputil.go` with the same two helpers (don't re-implement — factor the code so handlers imports httputil too). This may be its own small task.

- [ ] **Step 3: Build, test, commit**

```bash
cd server && go build ./... && go test ./models/... -count=1
git add server/models/
git commit -s -m "fix(server): JSON error responses from provider layer"
```

---

## Phase 6 — Handler migration waves

513 `http.Error` calls minus the ones already migrated in Phases 3 & 5 leaves roughly 485 to rewrite. This is mechanical work. It MUST be split into multiple PRs — one per wave, and within Wave 4, one per large file.

### Pattern to apply mechanically

For each `http.Error(w, <expr>, <status>)` call:

1. If `<expr>` is `ErrXxx(err).Error()` → `writeMeshkitError(w, ErrXxx(err), <status>)`
2. If `<expr>` is `err.Error()` where `err` is a MeshKit error → `writeMeshkitError(w, err, <status>)`
3. If `<expr>` is a bare string literal:
   - **Preferred:** add a MeshKit error code for it in `server/handlers/error.go` (next free `meshery-server-XXXX`) and use `writeMeshkitError`.
   - **Fallback:** `writeJSONError(w, "<string>", <status>)`. Add a TODO comment: `// TODO(error-code): promote to MeshKit code`.

Each wave gets its own PR with a commit per file. Within a file, one commit is fine — they're mechanical edits.

### Wave 1 — Middleware & providers (Phase 5) ✓ already scoped above

### Wave 2 — Small files

**Task 11:** `keys_handler.go` (2 sites)
**Task 12:** `schedule_handlers.go` (6 sites)
**Task 13:** `schema_handlers.go` (4 sites)
**Task 14:** `session_sync_handler.go` (1 site)
**Task 15:** `server_events_configuration_handler.go` (1 site)
**Task 16:** `server_spec_handler.go` (1 site)
**Task 17:** `provider_handler.go` (4 sites)
**Task 18:** `organization_handler.go` (2 sites)
**Task 19:** `common_handlers.go` (6 sites)
**Task 20:** `extensions.go` (3 sites)

For each task in this wave, the steps are:

- [ ] **Step 1:** `grep -n 'http.Error' server/handlers/<file>.go` to enumerate sites.
- [ ] **Step 2:** Apply the pattern above to each site. Read surrounding code for each to verify no logic change.
- [ ] **Step 3:** `cd server && go build ./... && go test ./handlers/ -count=1`
- [ ] **Step 4:** Commit as `fix(server): JSON errors in <file>`.

Bundle all Wave 2 files into one PR titled `fix(server): JSON errors across small handler files (Wave 2)`.

### Wave 3 — Mid-size files

**Tasks 21-34:** `user_handler.go` (16), `workspace_handlers.go` (24), `environments_handlers.go` (15), `credentials_handlers.go` (11), `connections_handlers.go` (28), `contexts_handler.go` (8), `database_handlers.go` (10 — minus the one fixed in Phase 3), `relationship_handlers.go` (3), `validate.go` (3), `performance_profiles_handler.go` (5), `fetch_results_handler.go` (14), `mesh_ops_handlers.go` (20 — minus the one fixed in Phase 4), `meshsync_handler.go` (7).

*(Files with zero `http.Error` sites — `remote_components_handler.go`, `evaluation_tracker.go` — are out of scope for Phase 6 but may still be touched in Phase 7 if they contain uncoded errors emitted via other paths.)*

Same pattern. Open a separate PR per ~5 files so review stays tractable.

Each task's checklist:

- [ ] Enumerate sites with grep.
- [ ] Apply migration pattern to each.
- [ ] **For uncoded sites**, add new MeshKit error codes in `server/handlers/error.go`. Run `grep -c '^\s*Err.*Code\s*=\s*"meshery-server-' server/handlers/error.go` to find the next free code number.
- [ ] Build + test after each file.
- [ ] Commit per file.

### Wave 4 — Large files (one PR per file)

**Task 36:** `meshery_pattern_handler.go` (69 sites) — own PR
**Task 37:** `meshery_application_handler.go` (35 sites) — own PR
**Task 38:** `prometheus_handlers.go` (30 sites) — own PR
**Task 39:** `grafana_handlers.go` (29 sites) — own PR
**Task 40:** `meshery_filter_handler.go` (25 sites) — own PR
**Task 41:** `component_handler.go` (22 sites) — own PR
**Task 42:** `events_streamer.go` (22 sites, excluding SSE section) — own PR
**Task 43:** `load_test_preferences_handler.go` (20 sites) — own PR
**Task 44:** `load_test_handler.go` (18 sites, excluding SSE) — own PR
**Task 45:** `k8sconfig_handler.go` (13 sites — minus 4 fixed in Phase 3) — own PR
**Task 46:** `design_engine_handler.go` (7 sites — minus 2 fixed in Phase 3), `design_import.go` (5), `component_generation.go` (3), `component_generator_helper.go` (2), `policy_relationship_handler.go` (6), `load_test_meshes_handler.go` (1) — one bundled PR

Each task's checklist matches Wave 2's. Additionally, for the largest files:
- [ ] **Verify no SSE / streaming sections were touched.** Before committing, grep the file for `text/event-stream` or `http.Flusher` and confirm those sections are untouched.
- [ ] **Run a smoke test in a live server** for at least one migrated endpoint — `curl -i <endpoint-that-returns-error>` and verify `Content-Type: application/json` plus a parseable body. Document this in the PR description.

---

## Phase 7 — Introduce MeshKit codes for the 89 uncoded errors

This phase OVERLAPS with Phase 6 — do it inline while migrating each file. The plan calls it out separately so it's explicit in the PR description.

### Pattern for each uncoded site

In `server/handlers/error.go`:

1. Pick the next free code number. Run:
```bash
grep -oE 'meshery-server-[0-9]+' server/handlers/error.go | sort -u | tail -5
```
Use the next integer.

2. Add a constant and constructor:
```go
const ErrWhateverCode = "meshery-server-19XX"

func ErrWhatever(err error) error {
	return errors.New(
		ErrWhateverCode,
		errors.Alert,
		[]string{"Short description"},
		[]string{err.Error()},      // or []string{"static long description"}
		[]string{"Probable cause"},
		[]string{"Suggested remediation"},
	)
}
```

3. Use `writeMeshkitError(w, ErrWhatever(err), status)` at the call site.

4. In the PR description, list every new error code added so reviewers can sanity-check the numbering and remediation copy.

---

## Phase 8 — UI consumer update (cross-repo: `meshery/schemas`)

Per the multi-repo awareness directive in [CLAUDE.md](/Users/l/.claude/CLAUDE.md), coordinate the schema change ahead of or alongside consumer changes.

### Task 48: Update RTK Query baseQuery to surface structured errors

**Repo:** `meshery/schemas` (not this repo)

**Files:** The file backing `@meshery/schemas/mesheryApi` that configures `fetchBaseQuery` or a custom `baseQuery`. Find it via:
```bash
grep -rn 'fetchBaseQuery\|createApi' ../schemas/ 2>/dev/null || echo "clone meshery/schemas locally"
```

- [ ] **Step 1:** Add a `transformErrorResponse` that extracts the new fields:

```ts
transformErrorResponse: (response: FetchBaseQueryError) => {
  if (response.data && typeof response.data === 'object' && 'error' in response.data) {
    const body = response.data as {
      error: string;
      code?: string;
      severity?: string;
      probable_cause?: string[];
      suggested_remediation?: string[];
      long_description?: string[];
    };
    return {
      status: response.status,
      message: body.error,
      code: body.code,
      severity: body.severity,
      probableCause: body.probable_cause,
      suggestedRemediation: body.suggested_remediation,
      longDescription: body.long_description,
    };
  }
  return response;
},
```

- [ ] **Step 2:** Update the UI notistack / toast error renderer (in `ui/` of meshery repo) to display `suggestedRemediation` when present.

- [ ] **Step 3:** Version + release `meshery/schemas`, update `package.json` in the meshery UI to point at the new version.

- [ ] **Step 4:** Cross-reference: in the PRs for Phase 6 waves, once the new schema version is published, mention it in the PR description so reviewers know the UI will start consuming the richer shape.

---

## Phase 9 — Flip the lint guard to blocking

After all waves complete:

### Task 49: Make `forbidigo: http.Error` blocking on all new code

- [ ] **Step 1:** Verify zero `http.Error` occurrences remain outside the allowlist:
```bash
grep -rn 'http\.Error(' server/handlers/ server/models/*provider*.go \
  | grep -v 'k8s_healthz_handler\|events_streamer\|load_test_handler:34[0-9]\|load_test_handler:37' \
  | wc -l
```
Expected: `0`.

- [ ] **Step 2:** Flip the guard to blocking. This is a 4-part change, not a one-liner:
  1. Remove `--new-from-rev=origin/master` from the `golangci-server` job args in `.github/workflows/go-testing-ci.yml`.
  2. Revert the checkout step's `fetch-depth: 0` back to the default (shallow); after step 1, the base history is no longer needed.
  3. Narrow the blanket allowlists for `server/handlers/events_streamer.go` and `server/handlers/load_test_handler.go` in `.github/.golangci.yml`. Blanket file-path regexes can't express "only the SSE section" — move to inline `//nolint:forbidigo` comments on specific call sites within the SSE sections, then drop those files from the YAML allowlist. The non-SSE sites should have been migrated by Task 42 / Task 44 at this point; verify zero remain before dropping the exclusion.
  4. Confirm `forbidigo` fails CI on any new `http.Error` in scope by pushing a throwaway test commit that introduces one, verifying CI fails, then reverting.

- [ ] **Step 3:** Open PR: `ci(server): enforce forbidigo http.Error ban repo-wide`.

---

## Self-review checklist

Before running the plan, engineer should confirm:

- [ ] Phase 1 (foundation) is ONE PR. The helpers and tests must land first.
- [ ] Phase 2 (lint guard) is the second PR. It must land before any handler sweep so new work doesn't re-introduce `http.Error`.
- [ ] Phase 3 & 4 (raw-text fixes + Content-Type) are small, independent PRs.
- [ ] Phase 5 (middleware + provider) is one focused PR — highest blast radius.
- [ ] Phase 6 is a wave of PRs. **Do not bundle all handlers into one PR — the diff will be un-reviewable.**
- [ ] For every new MeshKit error code introduced in Phase 6/7, the PR description lists it explicitly.
- [ ] SSE / healthz / binary / redirect paths are never touched (Category C).
- [ ] Phase 8 (cross-repo UI) is coordinated — the schemas change goes first or alongside, never after.
- [ ] Phase 9 (flip lint to blocking) is the last PR, after all waves are green.

## Execution notes

- **Total PR count:** approximately 20-25 PRs across all phases.
- **Estimated wall-clock effort:** 3-5 engineering days of focused work for the mechanical migration, plus 1-2 days for the foundation/lint/docs scaffolding, plus cross-repo UI coordination.
- **Merging order constraints:**
  - Phase 1 → Phase 2 → (Phase 3, Phase 4, Phase 5 in parallel) → Phase 6 waves → Phase 9.
  - Phase 8 (UI) can start any time after Phase 1 lands.
- **Rollback:** each PR is independent and revertable. The helpers in Phase 1 are additive; the lint rule in Phase 2 is advisory until Phase 9.
- **CI gating:** verify `go build ./...`, `go test ./handlers/... ./models/...`, and `golangci-lint run` all pass before each merge.
