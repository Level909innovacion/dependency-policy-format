# DPF, Dependency Policy Format

**Status: draft v0.0, open for discussion. Nothing here is stable.**

DPF is a proposal for an open, engine-independent format to write rules about software dependencies: which licences are acceptable, which vulnerabilities have been assessed and accepted (with a reason and an expiry date), which packages are banned, and how those rules apply to direct and transitive dependencies.

## Why

Open standards exist to *describe* components: PURL for identity, CycloneDX and SPDX for inventories, OSV for advisories, OpenVEX for exploitability statements. There is no standard to write *rules* about them. Every tool that enforces dependency policy (cargo-deny, Trivy, Grype, OSV-Scanner, Dependency-Check, Snyk, LicenseFinder, GitHub dependency review, ORT and others) keeps its rules in its own file format. An organisation running three scanners maintains the same decisions three times, in three incompatible syntaxes.

Andrew Nesbitt surveyed about forty such tools in March 2026 and found no shared format ([The Fragmented World of Dependency Policy](https://nesbitt.io/2026/03/19/the-fragmented-world-of-dependency-policy.html)). In May 2026 he pointed to Composer 2.10's `config.policy` as a reasonable candidate for a cross-tool *protocol* for block lists. DPF aims at the layer above: a portable file that expresses licence, vulnerability and package rules once, reusing existing identifiers (SPDX licence IDs, PURL, OSV and CVE IDs, CVSS), and that can be served through protocols such as Composer's.

## What is in this repository

| Path | Content | Licence |
|---|---|---|
| `spec/dpf-v0.0.md` | Specification draft: data model and semantics | CC-BY-4.0 (`LICENSE-SPEC`) |
| `schema/dpf.schema.json` | JSON Schema for policy files | Apache-2.0 (`LICENSE`) |
| `examples/` | Worked policy files | Apache-2.0 |
| `CONTRIBUTING.md` | How to comment, and the project's stance on generative AI | |

Planned, not yet here: a conformance test suite (policy, SBOM, expected decisions), a reference implementation in Rego for Open Policy Agent, adapters for Conftest (CI) and Kyverno (Kubernetes admission), converters from existing formats, and a CRA policy profile.

## Design principles

1. Reuse identifiers, invent no vocabulary: SPDX licence expressions, PURL, OSV and CVE identifiers, CVSS severities.
2. Every construct must be justified by at least two existing tools. The specification is written from the converters backwards.
3. Exceptions carry a reason, an expiry date and a scope. An exception without expiry is a design smell the format discourages.
4. The policy file is the decision. Engines (OPA, Kyverno, a scanner's own code) are implementations, and a conformance suite, not the reference implementation, defines correct behaviour.
5. Decisions should be exportable as OpenVEX so that an accepted vulnerability becomes a document other tools understand.

## How to comment

Open an issue. The first discussion thread is [issue #1](../../issues/1). We are particularly interested in hearing from maintainers of scanners, package managers and forges who would consider reading this format, and from anyone who thinks it should not exist because something equivalent already does.

## Who

Level 909 Soluciones Tecnológicas S.L., Seville, Spain. This draft is being prepared as part of a proposal to the [NLnet CodeSupply](https://nlnet.nl/codesupply/) open call. Whether or not that proposal is funded, the specification stays open under the licences above.
