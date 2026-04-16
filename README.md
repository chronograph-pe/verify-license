# verify-license

A GitHub Action that cross-references [Dependency Track](https://dependencytrack.org/) license violations against the authoritative package registry to eliminate false positives and correct SBOM metadata.

## How it works

For each violation reported by Dependency Track, the action looks up the package in its authoritative registry and compares the license:

| Result | Meaning | Action taken |
|--------|---------|--------------|
| **Confirmed** | Registry agrees with what DT reported | Included in `confirmed` output |
| **Corrected** | Registry disagrees — SBOM had wrong metadata | `bom.xml` updated with correct license, `has_corrections` set to `true` |
| **Unverifiable** | Package not found in registry | Excluded from output, logged separately |

Only confirmed violations are returned, so downstream steps (PR comments, workflow failures) are never triggered by bad SBOM metadata.

## Inputs

| Input | Required | Description |
|-------|----------|-------------|
| `violations` | Yes | JSON array of violations from Dependency Track |
| `language` | Yes | Language of the scanned project (`node` or `ruby`) |
| `bom-file` | Yes | Path to `bom.xml` relative to the workspace root |

## Outputs

| Output | Description |
|--------|-------------|
| `confirmed` | JSON array of violations confirmed by the package registry |
| `has_corrections` | `true` if `bom.xml` was updated with corrected license metadata |

## Supported registries

| Language | Registry |
|----------|----------|
| `node` | [registry.npmjs.org](https://registry.npmjs.org) |
| `ruby` | [rubygems.org](https://rubygems.org/api/v1) |

## Usage

```yaml
- name: Verify License Violations
  id: verify
  uses: chronograph-pe/verify-license@<sha>
  with:
    violations: ${{ steps.get-violations.outputs.results }}
    language: node
    bom-file: my-app/bom.xml

- name: Re-upload corrected SBOM
  if: steps.verify.outputs.has_corrections == 'true'
  run: |
    curl -X POST "$DT_URL/api/v1/bom" \
      -H "X-Api-Key: $DT_TOKEN" \
      -F "bom=@my-app/bom.xml"

- name: Handle confirmed violations
  run: echo '${{ steps.verify.outputs.confirmed }}'
```

## Security

This action is designed to be safe to use in public repositories and forks.

- **No script injection** — all inputs (`violations`, `language`, `bom-file`) are passed to the Python script via environment variables, never interpolated directly into shell commands
- **Path traversal protection** — `bom-file` is resolved to an absolute path and validated to be within `GITHUB_WORKSPACE` before any file read or write occurs
- **URL encoding** — package names are URL-encoded before being used in registry API calls, preventing URL injection for both npm (scoped packages) and RubyGems
- **Read-only registry access** — all registry lookups are unauthenticated GET requests to public APIs; no credentials are required or accepted
- **Network timeouts** — all outbound requests are capped at 15 seconds
- **No third-party dependencies** — the action uses only Python's standard library and shell builtins; nothing is installed at runtime

## Pinning

Always pin this action to a full commit SHA rather than a branch or tag:

```yaml
uses: chronograph-pe/verify-license@<full-sha>  # not @main
```

This ensures your workflow is not affected by upstream changes to this repository.
