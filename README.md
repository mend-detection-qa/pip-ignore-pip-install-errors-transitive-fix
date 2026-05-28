# pip-ignore-pip-install-errors-transitive-fix

Regression test project for [SCA-5589](https://mend-io.atlassian.net/browse/SCA-5589).

## What this tests

When `python.ignorePipInstallErrors=true` is configured and `pip install -r requirements.txt` **fails** (due to a bogus package in the manifest), the Unified Agent falls back to downloading each dependency line-by-line. Before the fix in SCA-5589, this fallback path caused transitive dependencies to be **incorrectly reported as direct** at the top level of the update-request.

## Project structure

| File | Purpose |
|------|---------|
| `requirements.txt` | 1 real direct dep (`neo4j-graphrag==1.6.0`) + 1 bogus dep to force pip failure |
| `.whitesource` | Mend scan trigger config |
| `wss-unified-agent.config` | UA config with `python.ignorePipInstallErrors=true` |
| `autotest_config.json` | Validation rules (see key assertions below) |
| `expected-tree.json` | Human-readable expected tree documentation |

## Key assertions (autotest_config.json)

### ✅ CRITICAL — fix validation

1. **`expected_direct_dependencies_number: {eq: 1}`** — Only `neo4j-graphrag` must be direct. Pre-fix this was `13`.
2. **`direct_dependencies_names_list: {eq: ["neo4j_graphrag-1.6.0-py3-none-any.whl"]}`** — Exact match: no other package at top level.
3. **`transitive_dependencies_names_list: {contains: ["pypdf", "pydantic", "fsspec", "neo4j"]}`** — These must appear as *transitive*, not direct.
4. **`expected_transitive_dependencies_number: {gte: 4}`** — Pre-fix this was `0` (all were at top level).

### Log line validation
- `python.ignorePipInstallErrors=true` must appear → confirms the flag is active
- `pip install command failed` must appear → confirms the fallback was actually triggered

## Bug scenario

```
requirements.txt:
  neo4j-graphrag==1.6.0          ← real direct dep
  this-package-does-not-exist==99.99.99  ← forces pip install failure
```

- pip tries `pip install -r requirements.txt` → **FAILS**
- UA detects failure, `ignorePipInstallErrors=true` → per-line fallback fires
- Fallback downloads each package individually into numbered subfolders
- **Pre-fix**: all downloaded wheels (including transitives of neo4j-graphrag) land
  in the same folder level → reported as top-level directs
- **Post-fix**: pip log lines `Collecting <pkg> (from <parent>)` are parsed to
  reconstruct the parent-child relationship correctly

## How to interpret results

| `direct_count` | `transitive_count` | Interpretation |
|---|---|---|
| 1 | ≥4 | ✅ Fix is working |
| 13 | 0 | ❌ Bug still present |
| 1 | 0 | ⚠️ Only direct found — check scanner logs |
| 0 | 0 | 🔴 Scanner resolution failed entirely |
