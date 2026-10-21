# Dependency Policy Format (DPF), draft v0.0

Status: draft for discussion. Licence: CC-BY-4.0.

## 1. Scope

A DPF policy is a YAML (or JSON) document that states which software components an organisation accepts in a given context, and under which conditions. It is evaluated against a software bill of materials (SBOM) in CycloneDX or SPDX format, optionally enriched with vulnerability and licence metadata keyed by Package URL (PURL). The output is a decision (accept or reject) and a list of findings, each traceable to a rule and to the data that triggered it.

DPF does not describe components, does not store advisories and does not define how an engine obtains metadata. Those are the roles of CycloneDX, SPDX, OSV, OpenVEX and the metadata services an engine chooses to use.

## 2. Terms

| Term | Meaning |
|---|---|
| Component | A software package identified by a PURL, as listed in an SBOM |
| Direct dependency | A component the root of the SBOM depends on without intermediaries |
| Transitive dependency | Any other component reachable from the root |
| Advisory | A vulnerability record identified by an OSV, CVE or GHSA identifier |
| Finding | A violation of one rule by one component, with evidence |
| Exception | A rule that suppresses findings for a stated reason until a stated date |

## 3. Document structure

```yaml
dpf: "0.0"                 # format version, required
metadata:                  # optional
  name: string
  owner: string
  updated: date
defaults:                  # optional, applied when a rule does not say otherwise
  scope: all | direct | transitive
  on_unknown: reject | warn | accept
licenses:                  # section 4
packages:                  # section 5
vulnerabilities:           # section 6
profiles:                  # section 7, optional
```

Unknown top-level keys are an error. Unknown keys inside a section are an error unless prefixed with `x-`.

## 4. Licence rules

```yaml
licenses:
  allow: [SPDX-expression, ...]
  deny:  [SPDX-expression, ...]
  exceptions:
    - purl: pkg:npm/left-pad@1.3.0   # exact or with version range
      allow: MIT
      reason: string                 # required
      expires: "2027-06-30"            # required
```

Semantics. A component is evaluated against the licence declared in the SBOM and, if the engine has it, the licence detected by a metadata source. If the two differ, the engine reports a finding of kind `license-mismatch` and evaluates the stricter of the two unless `defaults.on_unknown` says otherwise. `deny` takes precedence over `allow`. An SPDX expression in `allow` or `deny` is matched by licence expression equivalence, not by string comparison. A component whose licence is unknown is handled by `defaults.on_unknown`.

Justification: cargo-deny `[licenses].allow/deny`, LicenseFinder `permitted_licenses/restricted_licenses`, GitHub dependency review `allow-licenses/deny-licenses`, liccheck `authorized_licenses/unauthorized_licenses`.

## 5. Package rules

```yaml
packages:
  deny:
    - purl: pkg:npm/event-stream      # version omitted: all versions
      versions: ">=3.3.6 <3.3.7"      # optional, semver or ecosystem range
      reason: string
  allow_only:                         # optional allow list, exclusive
    - purl: pkg:pypi/requests
  sources:                            # optional, allowed registries per type
    pypi: ["https://pypi.org/simple"]
```

Semantics. `deny` rejects any component whose PURL matches, within the version range if given. `allow_only`, when present, rejects every component not listed. `sources` rejects components whose PURL qualifier `repository_url` is not in the list for that type.

Justification: cargo-deny `[bans]`, Maven Enforcer `bannedDependencies`, GitHub dependency review `deny-packages`, NuGet package source mapping, Composer `config.policy` block lists.

## 6. Vulnerability rules

```yaml
vulnerabilities:
  fail_above: medium                  # none | low | medium | high | critical
  severity_source: cvss               # cvss | source   (how severity is read)
  require_fix_available: false        # if true, only fail when a fixed version exists
  ignore:
    - id: CVE-2024-12345              # or GHSA-..., OSV-...
      purl: pkg:maven/org.example/lib # optional scope
      reason: string                  # required
      expires: "2027-01-31"             # required
      justification: not_affected     # optional, OpenVEX vocabulary
```

Semantics. For each component the engine obtains the set of advisories from its metadata source(s). A finding is produced for each advisory whose severity is above `fail_above`, unless an `ignore` entry matches by identifier (and by PURL, if given) and has not expired. An expired `ignore` is reported as a finding of kind `expired-exception` and no longer suppresses anything. `justification` uses the OpenVEX status justifications so that accepted exceptions can be exported as OpenVEX statements.

Justification: Snyk `.snyk` ignore (reason, expiry, path), OSV-Scanner `[[IgnoredVulns]]` (id, reason, ignoreUntil), Grype ignore rules, Trivy `.trivyignore.yaml` (statement, expired_at), Dependency-Check suppressions (until), Safety policy.

## 7. Scope and profiles

Every rule accepts an optional `scope: all | direct | transitive`. Profiles allow a policy to carry named variants (for example `production` and `development`) that override sections of the base policy. An engine evaluates exactly one profile, chosen by the caller, or the base policy if none is named.

```yaml
profiles:
  production:
    vulnerabilities:
      fail_above: low
```

## 8. Output

An engine returns a decision and a list of findings:

```yaml
decision: reject
findings:
  - kind: vulnerability | license | package | license-mismatch | expired-exception | unknown-component
    purl: string
    rule: string          # path of the rule in the policy, e.g. vulnerabilities.fail_above
    evidence: object      # advisory id, severity, detected licence, data source and timestamp
```

Engines should also be able to produce SARIF for forges and OpenVEX for accepted vulnerability exceptions. The shape of those exports will be specified once the conformance suite exists.

## 9. Conformance

A conforming implementation passes the conformance suite (policy, SBOM, expected decision and findings), to be published in this repository under Apache-2.0. Until the suite exists, nothing is conformant and this document is only a proposal.

## 10. Open questions

1. Whether `allow_only` for packages is worth having, or whether it belongs in a separate lockfile-level mechanism.
2. How to express path or file scope (as `.snyk` does) without tying the format to one ecosystem's layout.
3. Whether severity should be read from CVSS base score, from the advisory source's own label, or from a configurable mapping.
4. Whether to adopt Composer's `config.policy` list protocol as the transport for `packages.deny` and `vulnerabilities.ignore` when served by a repository.
5. The relationship with OpenVEX: exceptions exported to OpenVEX, and OpenVEX documents imported as exceptions, or one direction only.
