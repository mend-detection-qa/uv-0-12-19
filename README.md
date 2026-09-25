# Probe: version-constraint-arbitrary-equality

**Pattern:** `version-constraint-arbitrary-equality`
**PM:** uv 0.12.19
**Status:** untested (generated 2026-09-25)
**Schema version:** 1.0

## Purpose

This probe exercises two uv 0.12.19 features simultaneously:

1. **`===` arbitrary equality operator fix** (uv PR #21983): The PEP 440
   arbitrary equality operator `===` performs exact string matching
   against the version string in package metadata (no normalization,
   no epoch stripping). Before the fix, `===21.3` did not correctly
   match `packaging` published with version string `"21.3"`. After the
   fix, it resolves correctly to `packaging 21.3`.

2. **Omission of unused resolution settings** (uv PR #21944): For a
   simple, single-environment project with no platform markers or
   conditional dependencies, the generated `uv.lock` no longer contains
   `resolution-markers` or `supported-markers` fields in the lockfile
   header. The header is minimal: only `version` and `requires-python`.

## Files

```
pyproject.toml           PEP 621 manifest with ===21.3 specifier
uv.lock                  Lockfile without resolution-markers or
                         supported-markers fields
.python-version          Mend Python version detection file (takes
                         precedence over pyproject.toml)
src/arbitrary_equality_probe/__init__.py  Minimal source stub
expected-tree.json       Ground-truth dependency tree
README.md                This file
```

## Dependencies

| Package    | Version | Specifier  | Source   |
|------------|---------|------------|----------|
| iniconfig  | 2.0.0   | ==2.0.0    | registry |
| packaging  | 21.3    | ===21.3    | registry |
| pyparsing  | 3.3.3   | transitive | registry |

`packaging 21.3` requires `pyparsing!=3.0.5,>=2.0.2`, resolved to
`pyparsing 3.3.3` (latest stable). `iniconfig 2.0.0` has no
dependencies. `pyparsing 3.3.3` has no required dependencies.

## What Mend must detect

All three packages (`iniconfig`, `packaging`, `pyparsing`) present
in the dependency tree with correct versions and `source = "registry"`.
No markers on any package — the lockfile omits `resolution-markers`
and `supported-markers` entirely.

Mend must:
- Parse `pyproject.toml` without error despite the `===` operator
  in the dependency specifier (valid PEP 440 syntax).
- Parse `uv.lock` without failing when `resolution-markers` and
  `supported-markers` are absent from the lockfile header.
- Resolve `packaging` to version `21.3` (not normalize to `21.3.0`
  or fail to match the `===` specifier).
- Traverse the `packaging` -> `pyparsing` transitive edge.

## Mend failure modes

- `===` operator causes a parse error in Mend's dependency-specifier
  parser (operator not recognized as valid PEP 440).
- `packaging` matched at a different version because Mend treated
  `===` semantics as normalized `==`.
- Lockfile parser fails or returns empty tree when `resolution-markers`
  is absent (assumes field is always present).
- `pyparsing` missing from tree because Mend could not derive the
  transitive edge without `[package.metadata]`.

## Python version detection

**Mend uses `.python-version` (PIP precedence chain).**

File `.python-version` declares `3.11`. This takes higher precedence
than `pyproject.toml`'s `requires-python = ">=3.11"`. Both are
consistent and declare Python 3.11.

## Mend config

**Bucket B — no `.whitesource` (dynamic Python detection covers it;
uv tool itself not pinnable via install-tool).**

uv's dynamic version detection reads `requires-python` from
`[project]` in `pyproject.toml` and `.python-version` (which takes
higher PIP-chain precedence). The uv tool itself is NOT in the
`install-tool` list — only `python` is pinnable. Since this probe
does not target a specific Python version mismatch, no `.whitesource`
is emitted.

## Sources

- https://github.com/astral-sh/uv/pull/21983 (=== fix)
- https://github.com/astral-sh/uv/pull/21944 (omit unused
  resolution settings)
