# P2 — Repository Control Plane v2 Architecture Overhaul

Status: active architecture program  
Program: `P2`  
Current stage: `P2S1 — Evidence and Contract Freeze`  
Decision posture: evidence before protocol, protocol before implementation, implementation spike before migration.

## 1. Purpose

P2 is a major architecture and product-protocol overhaul. It is not a language rewrite program and it does not pre-commit the project to Go, OCI, a new task tracker, or a new policy language.

The program exists because the v1 implementation has accumulated a large compatibility and migration surface while the durable product thesis can be stated much more narrowly:

> agent-onboard makes repository authority deterministic, portable, resolvable, and enforceable across coding-agent runtimes.

P2 must reduce the number of concepts an agent or maintainer must understand before useful work. Any implementation change that preserves or increases v1 complexity without a measured product benefit is a failed P2 outcome.

## 2. Baseline census

Measured from the source repository at the start of P2S1 after the first cold-start regression patch:

| Surface | Baseline |
| --- | ---: |
| CLI JavaScript modules | 138 |
| CLI JavaScript lines | 36,491 |
| Domain JavaScript modules | 124 |
| Domain JavaScript lines | 33,478 |
| `architecture` domain | 13,490 lines |
| `target` domain | 8,172 lines |
| `package` domain | 6,994 lines |
| `core` domain | 2,445 lines |
| `work-items` domain | 1,820 lines |
| `authority` domain | 454 lines |
| `runtime-composer.js` | 934 lines / 47,756 bytes |
| User-facing top-level command tokens | 26 |
| Runtime schema identifiers detected | 371 |
| `.agent-onboard` files | 132 |
| `.agent-onboard` JSON files | 113 |
| `.agent-onboard` JSONL files | 19 |
| `.agent-onboard` bytes | ~1.22 MB |

The import graph has no known source-module cycle. The problem is therefore not primarily circular dependencies. The dominant risks are public-surface breadth, state fragmentation, compatibility/history machinery living too close to product runtime, and a mutable composition model that makes service capability boundaries harder to reason about.

The `architecture` domain alone is roughly forty percent of domain LOC. P2 treats that as a strong signal that migration history has become part of the runtime architecture and must be classified before anything is ported.

## 3. Evidence from external cold-start testing

The first external trial found three concrete v1 issues:

1. `.npmrc` was ignored as npm evidence when no lockfile or `packageManager` field existed.
2. conventional lowercase `readme.md` was missed because root documentation detection was case-sensitive.
3. `agents --preview --target <path>` silently ignored an unsupported argument and used the current working directory.

`W1` fixes these without expanding the public `agents` command surface:

- `.npmrc` is npm package-manager evidence;
- root conventional docs are detected case-insensitively while preserving the actual filename;
- `agents` rejects unsupported arguments and therefore fails closed instead of routing implicitly.

The trial also exposed a product-measurement problem: an aggregate readiness score can over-reward installation of agent-onboard metadata and under-represent pre-existing repository operability. P2 must use evidence-backed checks before considering any aggregate readiness score.

## 4. Prior-art boundary

P2 does not compete by rebuilding adjacent layers.

### Instructions

`AGENTS.md` is a simple, tool-agnostic instruction convention and supports repository-local/nested guidance. P2 treats it as an instruction layer, not as machine-verifiable authority.

Reference: https://agents.md/

### Task and work state

Projects such as Beads already provide dependency-aware agent work tracking, ready-work discovery, claim/close workflows, and durable repository-associated state. P2 therefore does not require agent-onboard to own the canonical task database.

Reference: https://github.com/steveyegge/beads

### Specs and intent

Spec Kit and OpenSpec already occupy spec/plan/task/change workflows. P2 should discover or adapt to those systems rather than implement a competing specification framework.

References:

- https://github.com/github/spec-kit
- https://github.com/Fission-AI/OpenSpec

### Policy engines

OPA separates policy decision-making from policy enforcement. Cedar models authorization around principal, action, resource, and request context. Both are useful architectural references, but P2 v2 should not require users to learn Rego or Cedar for basic repository policy.

References:

- https://www.openpolicyagent.org/docs
- https://docs.cedarpolicy.com/

### OCI and attestations

OCI Image/Distribution 1.1 provides `artifactType`, `subject`, and referrers for attaching metadata artifacts to immutable OCI subjects. in-toto v1 statements bind typed predicates to subject digests. P2 should reuse those layers instead of inventing a proprietary attestation envelope or registry protocol.

References:

- https://opencontainers.org/posts/blog/2024-03-13-image-and-distribution-1-1/
- https://github.com/in-toto/attestation/tree/main/spec/v1
- https://docs.sigstore.dev/

## 5. Product boundary

### P2 owns

- repository operability discovery;
- repository-declared authority;
- deterministic authority resolution;
- plan-before-apply mutation contracts;
- evidence explaining a resolved decision;
- provider discovery for work/spec systems;
- adapters that translate resolved authority into runtime-specific enforcement surfaces;
- portable operability snapshots and attestable predicates.

### P2 does not own

- a general task tracker;
- a specification-driven-development framework;
- a multi-agent orchestrator;
- an LLM memory database;
- a generic policy language;
- an identity provider;
- a container runtime;
- a PKI/signing ecosystem;
- an OCI registry;
- a repository-context packing engine.

Non-goals are architectural constraints. Adding one requires an explicit P2 decision record rather than incidental implementation growth.

## 6. Authority model

P2 separates three layers that v1 documentation can currently blur:

### Declared

Repository-owned policy describes intended decisions.

Initial decision vocabulary:

- `allow`
- `review`
- `deny`

### Resolved

The core deterministically evaluates a request against repository policy. Resolution is local, LLM-free, explainable, and fail-closed for unknown safety-critical state.

A request conceptually contains:

- principal;
- action;
- resource/scope;
- request context;
- policy source;
- identity assurance.

The shape is influenced by Cedar's principal/action/resource/context model without adopting Cedar syntax as the default repository format.

### Enforced

An execution adapter applies a resolved decision using capabilities available in a specific runtime. A repository declaration is not described as enforced unless an integration can demonstrate the enforcement boundary.

Examples include coding-agent permission/sandbox surfaces, MCP tool policy, CI gates, or an optional OCI runtime adapter.

The core must report enforcement level explicitly, for example `declared_only`, `resolved`, or an adapter-specific enforced state.

## 7. Identity assurance

P2 must distinguish asserted identity from verified identity.

A string such as `agent:codex` is not cryptographic authentication. Policy decisions may still use it, but the result must carry an assurance level.

Possible assurance classes:

- `anonymous`;
- `asserted`;
- `verified`.

Verified identity may later come from CI OIDC, Sigstore identity, workload identity, or another adapter. None of those systems is required for local repository inspection.

## 8. Protocol v2 hypothesis

This is a hypothesis for P2S2, not a frozen contract.

The canonical first-read machine surface should target no more than two files:

```text
AGENTS.md
.agent-onboard/control.json
.agent-onboard/authority.json
```

`AGENTS.md` remains prose instructions. `control.json` is a compact discovery/root document. `authority.json` is structured current policy.

A possible `control.json` shape:

```json
{
  "schema": "aob.dev/control/v2",
  "authority": "authority.json",
  "work": { "provider": "github" },
  "spec": { "provider": "none" }
}
```

Provider state is not copied into the control file. Local v1 work-items may become one provider among GitHub Issues, Beads, or future integrations.

Optional JSONL may remain appropriate for append-only audit/evidence, but no event log becomes canonical merely because v1 already has one.

Protocol principles:

- Markdown = meaning/instructions;
- JSON = current structured truth;
- JSONL = optional append-only history/evidence;
- filenames are part of the discovery API;
- canonical surface must be much smaller than implementation surface;
- no registry/network dependency is required to understand a local repo.

## 9. Core architecture hypothesis

If Go passes the P2S3 gate, the target dependency direction is:

```text
CLI
  -> application use cases
      -> domain
          -> ports/interfaces
              <- filesystem / providers / adapters
```

No mutable service-locator equivalent should replace `Object.assign(context, service)`.

Capabilities should be explicit. For example, a read-only doctor use case receives a read interface and cannot write by construction; mutation use cases receive a separate write capability only after authority has been resolved.

The first Go spike implements read-only behavior only:

- status;
- profile;
- doctor;
- resolve.

Go is accepted only if it materially simplifies the equivalent slice while preserving semantic compatibility. Runtime speed alone is not sufficient justification.

## 10. OCI integration hypothesis

OCI is an integration and distribution layer, not the canonical live repository database.

Repository truth flows outward:

```text
repository -> resolved control snapshot -> OCI artifact / attestation
```

Local inspection must remain fully operable without Docker, containerd, ORAS, an OCI registry, or network access.

### OCI artifact use

A future adapter may package a control/operability snapshot as an OCI artifact and attach it to an image, binary, or other OCI subject through the OCI 1.1 subject/referrers model.

### Attestation use

P2 should define only an agent-operability predicate. It should reuse an in-toto Statement v1 envelope to bind the predicate to immutable subject digests, and reuse Sigstore/GitHub attestation mechanisms for signing/verification where available.

P2 must not create proprietary equivalents for:

- OCI registries;
- in-toto statements;
- signing key formats;
- transparency logs.

### OCI runtime use

An optional runtime adapter may compile resolved authority into OCI runtime restrictions where the runtime has appropriate primitives. This is an enforcement adapter, not a dependency of the core.

## 11. P2 execution program

### P2S1 — Evidence and Contract Freeze

Goal: determine what v1 actually promises before changing the protocol.

#### P2S1M1 — Current Reality Baseline

- `W1` External cold-start detection hardening.
- `W2` Architecture baseline census and P2 decision dossier.
- `W3` Command/schema/state classification and v1 semantic oracle.

#### P2S1M2 — External Fixture Corpus

Freeze representative repositories instead of following remote heads:

- small Node library;
- Node application;
- Python package;
- Go CLI;
- monorepo;
- minimal repo;
- messy legacy repo;
- repo with nested AGENTS.md;
- repo with external work provider;
- repo with spec/change provider.

Measure detection correctness, false positives/negatives, files/bytes read, output stability, and source-tree mutation hashes.

#### Gate G1 — public v1 understood

G1 passes only when each significant command/schema/state surface is classified as one of:

- public semantic contract;
- compatibility projection;
- derived cache/index;
- audit/history;
- internal/release-dev;
- migration history eligible for retirement.

No P2 protocol deletion occurs before G1.

### P2S2 — Protocol v2 RFC

Produce decision records for:

1. product boundary/non-goals;
2. canonical files;
3. authority decision model;
4. declared/resolved/enforced semantics;
5. principal assurance;
6. work provider interface;
7. spec provider interface;
8. audit/event semantics;
9. v1/v2 compatibility projection;
10. OCI mapping;
11. in-toto predicate;
12. runtime adapter contract.

#### Gate G2 — protocol smaller than v1

G2 requires:

- no more than two canonical machine-readable first-read files;
- deterministic local authority resolution;
- no mandatory task/spec duplication;
- offline inspection;
- fail-closed unknown authority;
- a migration/projection strategy for selected v1 contracts.

### P2S3 — Go Core Parity Spike

Implement only the frozen read-only slice and run it side-by-side with v1 across the fixture corpus.

#### Gate G3 — Go / no-Go

Go wins only if all selected semantic contracts reach parity, read-only operations mutate zero files, dependencies are explicit, supported cross-platform builds are demonstrated, npm UX remains viable, and implementation complexity is materially lower than the equivalent JS slice.

If Go fails the gate, protocol simplification continues using JS. P2 is not blocked on a language choice.

### P2S4 — Authority Kernel

Implement typed action/resource/principal/context resolution, decision explanation, identity assurance, and policy-source evidence.

### P2S5 — Plan / Apply and Providers

Every mutation follows:

```text
Inspect -> Plan -> Resolve authority -> Review if required -> Apply -> Evidence
```

Introduce work/spec provider interfaces without requiring every provider to support mutation.

### P2S6 — OCI and Attestation

Prototype local OCI layout/inspection first, then remote attach/pull only after the predicate and media-type contract stabilizes.

#### Gate G4 — independently verifiable artifact chain

Demonstrate:

```text
source -> control snapshot -> OCI digest -> typed attestation -> independent verification
```

### P2S7 — Runtime Enforcement Adapters

Adapters translate resolved authority to external runtimes. Core remains runtime-neutral.

### P2S8 — CLI Reset

Target a budget of 5–8 primary commands. Advanced, migration, and release-dev surfaces move behind subcommands or source tooling instead of remaining permanent top-level product concepts.

### P2S9 — Distribution and Supply Chain

If Go wins, canonical binaries come from reproducible release builds. npm remains a thin distribution/launcher experience where useful. Release artifacts should use existing provenance/attestation ecosystems rather than custom signing formats.

### P2S10 — Cutover and Legacy Retirement

P2 is successful only when legacy code/state can be deleted or isolated. A final system containing the entire v1 runtime plus a new Go runtime is a failed migration outcome.

## 12. Metrics and budgets

P2 treats complexity as a release metric alongside correctness.

Track at every gate:

- CLI/runtime LOC;
- module/package count;
- public command count;
- public schema count;
- canonical state-file count;
- first-read bytes/files;
- dependency count;
- fixture correctness;
- read-only mutation count;
- compatibility coverage.

Desired direction is monotonic reduction for public concepts and canonical state. Growth requires a recorded rationale.

## 13. Safety and compatibility principles

- Unknown safety-critical authority fails closed.
- Read-only commands do not receive hidden write behavior.
- `--write` remains explicit during migration.
- Plan output is evidence, not permission.
- Instructions are not described as enforcement.
- Derived indexes never become authority merely for performance.
- No binary state backend is introduced before protocol/state ownership is frozen.
- No external provider becomes mandatory for local inspection.
- No v1 public semantic contract is removed before its classification and migration disposition are recorded.

## 14. Release 0.1.2 foundation

The P2 foundation release intentionally does not change the existing runtime release-line identifier. P2S1 exists to freeze and classify the v1 public contract before a new release-line contract is declared.

The source release contains:

- the external cold-start regression fixes from `W1`;
- this P2 architecture dossier and measured baseline;
- P2 work-item lifecycle state;
- package version `0.1.2`;
- no Go runtime and no OCI runtime dependency yet.

This keeps the first P2 release reversible and evidence-oriented.

## 15. Immediate next work

After this dossier is validated, `W3` becomes the next work item:

> classify the 26 command tokens, runtime schemas, and repository state artifacts; build the semantic v1 oracle used by G1.

Only after G1 passes should P2S2 freeze the v2 protocol or admit a Go parity spike.

## 16. P2S1M1 W3 v1 semantic oracle freeze

`W3` freezes the v1 command/schema/state census below as a prefix-compressed, machine-readable semantic oracle. Prefix compression changes representation only: expand `runtime_schema_prefix` and `state_path_prefix` before comparing identifiers.

The frozen census contains:

- 26 top-level command tokens: 18 public semantic contract, 6 compatibility projection, 2 internal/release-dev;
- 372 packaged-runtime schema identifiers: 254 public semantic contract, 26 derived cache/index, 27 audit/history, 59 internal/release-dev, 6 migration-history eligible for retirement;
- 134 pre-W3 `.agent-onboard` repository-state artifacts: 9 public semantic contract, 9 derived cache/index, 22 audit/history, 21 internal/release-dev, 73 migration-history eligible for retirement;
- 8 read-only semantic probes covering status, work-item schema/template validation, authority state/index checks, architecture check, release surface check, and target profile.

The oracle records G1 classification completion but does **not** authorize protocol deletion. `P2S1M2` external fixture evidence remains required before any v1 protocol deletion or P2S2 cutover decision.

```json
{"schema":"agent-onboard-p2-v1-semantic-oracle-freeze-001","work_item_id":"W3","source_package_version":"0.1.2","vocabulary":{"public_semantic_contract":"public semantic contract","compatibility_projection":"compatibility projection","derived_cache_index":"derived cache/index","audit_history":"audit/history","internal_release_dev":"internal/release-dev","migration_history_retire":"migration history eligible for retirement"},"census":{"command_tokens":26,"runtime_schema_identifiers":372,"repository_state_artifacts":134,"semantic_probes":8},"commands_by_class":{"public_semantic_contract":["agents","authority","bridge","check","ci","claim","contracts","contributor","create","guard","help","init","issue","mcp","status","target","version","work-items"],"compatibility_projection":["--help","--version","-h","-v","target-config","target-instance"],"derived_cache_index":[],"audit_history":[],"internal_release_dev":["architecture","release"],"migration_history_retire":[]},"runtime_schema_prefix":"agent-onboard-","runtime_schemas_by_class":{"public_semantic_contract":["agents-result-001","authority-command-error-001","authority-path-001","cli-error-001","guard-boundary-check-result-001","guard-command-error-001","guard-plan-001","init-result-001","public-agents-bridge-check-001","public-agents-bridge-error-001","public-agents-bridge-plan-001","public-agents-bridge-write-001","public-ai-discovery-001","public-ai-discovery-error-001","public-authority-command-adapter-authority-result-001","public-authority-command-adapter-extraction-001","public-authority-command-adapter-extraction-check-result-001","public-authority-command-adapter-extraction-module-001","public-authority-command-adapter-extraction-result-001","public-authority-command-adapter-instance-001","public-authority-command-adapter-run-result-001","public-authority-first-read-check-result-001","public-authority-first-read-result-001","public-authority-live-state-shard-001","public-authority-policies-state-shard-001","public-boundary-guard-enforcement-seed-contract-001","public-bundled-authority-domain-view-001","public-bundled-claims-domain-view-001","public-bundled-core-domain-view-001","public-bundled-work-items-domain-view-001","public-check-fast-progress-jsonl-001","public-check-fast-result-001","public-check-plan-001","public-check-plan-fast-error-001","public-ci-surface-001","public-ci-surface-error-001","public-cli-runtime-de-monolith-planning-001","public-cli-runtime-de-monolith-planning-check-result-001","public-cli-runtime-de-monolith-planning-result-001","public-closed-gate-artifact-compaction-apply-001","public-closed-gate-artifact-compaction-apply-check-result-001","public-closed-gate-artifact-compaction-apply-result-001","public-closed-gate-artifact-compaction-dry-run-001","public-closed-gate-artifact-compaction-dry-run-check-result-001","public-closed-gate-artifact-compaction-dry-run-result-001","public-closed-gate-artifact-compaction-plan-001","public-closed-gate-artifact-compaction-plan-check-result-001","public-closed-gate-artifact-compaction-plan-result-001","public-closed-gate-raw-artifact-prune-apply-admission-001","public-closed-gate-raw-artifact-prune-apply-admission-check-result-001","public-closed-gate-raw-artifact-prune-apply-admission-result-001","public-closed-gate-raw-artifact-prune-dry-run-001","public-closed-gate-raw-artifact-prune-dry-run-check-result-001","public-closed-gate-raw-artifact-prune-dry-run-result-001","public-closed-gate-raw-artifact-prune-planning-001","public-closed-gate-raw-artifact-prune-planning-check-result-001","public-closed-gate-raw-artifact-prune-planning-result-001","public-command-router-001","public-command-router-check-result-001","public-command-router-result-001","public-command-surface-catalog-001","public-command-surface-error-001","public-contract-check-001","public-contract-output-file-validation-001","public-contract-output-validation-001","public-contract-spine-001","public-contracts-error-001","public-contributor-admission-dry-run-001","public-contributor-admission-error-001","public-core-command-adapter-extraction-001","public-core-command-adapter-extraction-check-result-001","public-core-command-adapter-extraction-module-001","public-core-command-adapter-extraction-result-001","public-core-command-adapter-help-result-001","public-core-command-adapter-instance-001","public-core-command-adapter-run-result-001","public-core-config-guard-service-instance-001","public-core-config-guard-service-seed-001","public-create-dry-run-001","public-create-dry-run-error-001","public-domain-service-facades-001","public-domain-service-facades-check-result-001","public-domain-service-facades-result-001","public-exact-artifact-oracle-contract-001","public-exact-artifact-oracle-result-001","public-full-test-runner-completion-001","public-full-test-runner-completion-check-result-001","public-full-test-runner-completion-result-001","public-installed-authority-state-shard-parity-contract-001","public-installed-authority-state-shard-parity-result-001","public-installed-first-read-contract-service-seed-001","public-installed-package-parity-smoke-result-001","public-issue-intake-classification-001","public-issue-intake-error-001","public-mcp-bridge-plan-001","public-mcp-bridge-plan-error-001","public-modular-runtime-package-inclusion-plan-001","public-modular-runtime-package-inclusion-plan-check-result-001","public-modular-runtime-package-inclusion-plan-result-001","public-no-mutation-runtime-boundary-001","public-operator-guide-001","public-operator-guide-error-001","public-package-command-adapter-extraction-001","public-package-command-adapter-extraction-check-result-001","public-package-command-adapter-extraction-module-001","public-package-command-adapter-extraction-result-001","public-package-command-adapter-instance-001","public-package-command-adapter-run-result-001","public-package-coordinate-service-seed-001","public-package-fast-runner-engine-001","public-package-keyword-taxonomy-compaction-001","public-package-keyword-taxonomy-compaction-check-result-001","public-package-keyword-taxonomy-compaction-result-001","public-package-runtime-service-instance-001","public-package-runtime-service-seed-001","public-package-surface-service-instance-002","public-package-surface-service-seed-002","public-packaged-router-port-inclusion-001","public-packaged-router-port-inclusion-check-result-001","public-packaged-router-port-inclusion-result-001","public-quickstart-001","public-quickstart-error-001","public-router-command-adapter-delegation-expansion-001","public-router-command-adapter-delegation-expansion-check-result-001","public-router-command-adapter-delegation-expansion-result-001","public-runtime-agents-bridge-service-001","public-runtime-command-registry-extraction-001","public-runtime-contracts-001","public-runtime-guard-service-instance-001","public-runtime-service-partition-seed-001","public-source-domain-extraction-rehearsal-001","public-source-domain-extraction-rehearsal-check-result-001","public-source-domain-extraction-rehearsal-result-001","public-source-domain-module-partition-plan-001","public-source-domain-module-partition-plan-check-result-001","public-source-domain-module-partition-plan-result-001","public-source-module-authority-second-slice-001","public-source-module-claims-bundle-parity-001","public-source-module-claims-bundle-parity-check-result-001","public-source-module-claims-bundle-parity-result-001","public-source-module-claims-first-slice-001","public-source-module-claims-first-slice-check-result-001","public-source-module-claims-first-slice-result-001","public-source-module-claims-installed-fallback-smoke-001","public-source-module-claims-installed-fallback-smoke-check-result-001","public-source-module-claims-installed-fallback-smoke-result-001","public-source-module-claims-runtime-bridge-001","public-source-module-claims-runtime-bridge-check-result-001","public-source-module-claims-runtime-bridge-result-001","public-source-module-core-first-slice-001","public-source-module-extraction-adapter-boundary-001","public-source-module-extraction-adapter-boundary-check-result-001","public-source-module-extraction-adapter-boundary-result-001","public-source-module-extraction-authority-bundle-parity-001","public-source-module-extraction-authority-bundle-parity-check-result-001","public-source-module-extraction-authority-bundle-parity-result-001","public-source-module-extraction-authority-runtime-bridge-001","public-source-module-extraction-authority-runtime-bridge-check-result-001","public-source-module-extraction-authority-runtime-bridge-result-001","public-source-module-extraction-bundle-parity-001","public-source-module-extraction-bundle-parity-check-result-001","public-source-module-extraction-bundle-parity-result-001","public-source-module-extraction-first-slice-001","public-source-module-extraction-first-slice-check-result-001","public-source-module-extraction-first-slice-result-001","public-source-module-extraction-installed-fallback-smoke-001","public-source-module-extraction-installed-fallback-smoke-check-result-001","public-source-module-extraction-installed-fallback-smoke-result-001","public-source-module-extraction-runtime-bridge-001","public-source-module-extraction-runtime-bridge-check-result-001","public-source-module-extraction-runtime-bridge-result-001","public-source-module-extraction-second-slice-first-slice-001","public-source-module-extraction-second-slice-first-slice-check-result-001","public-source-module-extraction-second-slice-first-slice-result-001","public-source-module-extraction-second-slice-plan-001","public-source-module-extraction-second-slice-plan-check-result-001","public-source-module-extraction-second-slice-plan-result-001","public-source-module-work-items-bundle-parity-001","public-source-module-work-items-bundle-parity-check-result-001","public-source-module-work-items-bundle-parity-result-001","public-source-module-work-items-first-slice-001","public-source-module-work-items-first-slice-check-result-001","public-source-module-work-items-first-slice-result-001","public-source-module-work-items-installed-fallback-smoke-001","public-source-module-work-items-installed-fallback-smoke-check-result-001","public-source-module-work-items-installed-fallback-smoke-result-001","public-source-module-work-items-runtime-bridge-001","public-source-module-work-items-runtime-bridge-check-result-001","public-source-module-work-items-runtime-bridge-result-001","public-target-command-adapter-extraction-module-001","public-target-command-adapter-instance-001","public-target-command-adapter-run-result-001","public-target-command-adapter-target-result-001","public-target-governance-budget-check-001","public-target-governance-budget-contract-001","public-target-governance-preview-001","public-target-handoff-preview-001","public-target-handoff-readiness-check-001","public-target-handoff-readiness-reason-001","public-target-memory-descriptor-001","public-target-memory-preview-001","public-target-onboarding-dry-run-fixture-result-001","public-target-onboarding-explicit-write-result-001","public-target-onboarding-fixture-matrix-002","public-target-onboarding-installed-package-smoke-result-001","public-target-onboarding-post-publish-verification-handoff-001","public-target-onboarding-published-package-acceptance-result-001","public-target-onboarding-real-target-repo-trial-gate-result-001","public-target-onboarding-real-target-trial-result-001","public-target-onboarding-surface-plan-001","public-target-runtime-inventory-001","public-target-runtime-namespace-001","public-target-runtime-namespace-check-result-001","public-target-runtime-namespace-result-001","public-target-work-items-preview-001","public-thin-cli-router-route-result-001","public-version-reference-policy-001","public-version-reference-policy-check-result-001","public-work-items-command-adapter-extraction-module-001","public-work-items-command-adapter-instance-001","public-work-items-command-adapter-run-result-001","public-work-items-runtime-service-instance-001","public-work-items-runtime-service-seed-001","status-001","target-authority-map-001","target-bootstrap-result-001","target-config-001","target-config-file-validation-001","target-config-schema-response-001","target-config-template-response-001","target-config-template-validation-001","target-doctor-result-001","target-instance-takeover-result-001","target-metadata-check-result-001","target-metadata-plan-result-001","target-metadata-write-result-001","target-profile-result-001","target-repair-result-001","target-runtime-command-error-001","target-runtime-namespace-001","target-runtime-project-001","target-work-items-001","work-items-append-result-001","work-items-claim-result-001","work-items-close-result-001","work-items-file-validation-001","work-items-init-result-001","work-items-list-response-001","work-items-mine-response-001","work-items-next-response-001","work-items-schema-response-001","work-items-summary-response-001","work-items-template-response-001","work-items-template-validation-001"],"compatibility_projection":[],"derived_cache_index":["public-authority-compact-index-001","public-authority-compact-index-check-result-001","public-authority-compact-index-result-001","public-authority-first-read-index-001","public-authority-first-read-index-check-result-001","public-authority-indexes-state-shard-001","public-closed-gates-index-001","public-closed-gates-index-preview-001","public-package-source-manifest-001","public-package-source-manifest-check-result-001","public-package-source-manifest-service-instance-001","public-source-manifest-hash-cache-budget-report-001","public-source-manifest-service-seed-003","public-target-governance-index-drift-check-001","public-target-governance-index-materialization-dry-run-001","public-target-governance-index-materialization-write-001","public-target-governance-index-refresh-after-mutation-001","target-claims-index-001","target-content-manifest-002","target-manifest-001","target-manifest-drift-check-result-001","target-manifest-init-result-001","target-manifest-refresh-result-001","target-work-items-index-001","work-items-governance-index-refresh-error-001","work-items-governance-index-refresh-skipped-001"],"audit_history":["public-architecture-m1-closure-m2-seed-001","public-architecture-m1-closure-m2-seed-check-result-001","public-architecture-m1-closure-m2-seed-result-001","public-authority-closed-gate-event-001","public-claim-ledger-append-result-001","public-claim-ledger-entry-001","public-claim-ledger-lifecycle-result-001","public-claim-ledger-validation-001","public-closed-gate-archive-reader-001","public-closed-gate-archive-reader-check-result-001","public-closed-gate-archive-reader-result-001","public-closed-gate-archive-record-001","public-readme-first-read-history-split-plan-001","public-readme-first-read-history-split-plan-check-result-001","public-readme-first-read-history-split-plan-result-001","public-readme-history-archive-split-apply-001","public-readme-history-archive-split-apply-check-result-001","public-readme-history-archive-split-apply-result-001","public-readme-history-archive-split-dry-run-001","public-readme-history-archive-split-dry-run-check-result-001","public-readme-history-archive-split-dry-run-result-001","public-readme-history-index-preview-001","public-source-domain-extraction-stabilization-closure-review-001","public-source-domain-extraction-stabilization-closure-review-check-result-001","public-source-domain-extraction-stabilization-closure-review-result-001","work-item-closure-record-001","work-items-closure-archive-write-001"],"internal_release_dev":["architecture-command-error-001","public-architecture-check-result-001","public-architecture-command-adapter-architecture-result-001","public-architecture-command-adapter-extraction-001","public-architecture-command-adapter-extraction-check-result-001","public-architecture-command-adapter-extraction-module-001","public-architecture-command-adapter-extraction-result-001","public-architecture-command-adapter-instance-001","public-architecture-command-adapter-run-result-001","public-architecture-map-001","public-architecture-map-result-001","public-authority-state-sharding-seed-001","public-authority-state-sharding-seed-check-result-001","public-authority-state-sharding-seed-result-001","public-claims-domain-source-extraction-plan-001","public-claims-domain-source-extraction-plan-check-result-001","public-claims-domain-source-extraction-plan-result-001","public-clean-compaction-baseline-001","public-clean-compaction-baseline-check-result-001","public-clean-compaction-baseline-result-001","public-clean-compaction-catalog-001","public-clean-compaction-catalog-check-result-001","public-clean-compaction-catalog-result-001","public-installed-parity-architecture-smoke-001","public-installed-parity-architecture-smoke-result-001","public-package-command-adapter-release-result-001","public-package-runtime-service-release-result-001","public-package-surface-preservation-001","public-package-surface-preservation-check-result-001","public-package-surface-preservation-result-001","public-release-check-001","public-release-check-result-007","public-release-check-result-019","public-release-clean-closed-gates-runtime-slice-001","public-release-contract-039","public-release-contract-response-001","public-release-fixture-matrix-022","public-release-fixture-response-001","public-release-plan-005","public-release-target-repo-product-check-result-001","public-runtime-guard-service-decomposition-001","public-source-extraction-golden-output-freeze-001","public-source-extraction-golden-output-freeze-check-result-001","public-source-extraction-golden-output-freeze-result-001","public-thin-cli-router-seed-001","public-thin-cli-router-seed-check-result-001","public-thin-cli-router-seed-module-001","public-thin-cli-router-seed-result-001","public-thin-entrypoint-rehearsal-vector-result-001","public-thin-entrypoint-router-cutover-application-001","public-thin-entrypoint-router-cutover-application-check-result-001","public-thin-entrypoint-router-cutover-application-result-001","public-thin-entrypoint-router-cutover-rehearsal-001","public-thin-entrypoint-router-cutover-rehearsal-check-result-001","public-thin-entrypoint-router-cutover-rehearsal-result-001","public-work-items-domain-source-extraction-plan-001","public-work-items-domain-source-extraction-plan-check-result-001","public-work-items-domain-source-extraction-plan-result-001","release-command-error-001"],"migration_history_retire":["public-compatibility-command-port-instance-001","public-compatibility-command-port-run-result-001","public-compatibility-command-port-seed-001","public-compatibility-command-port-seed-check-result-001","public-compatibility-command-port-seed-module-001","public-compatibility-command-port-seed-result-001"]},"state_path_prefix":".agent-onboard/","repository_state_artifacts_by_class":{"public_semantic_contract":["authority-path.json","metadata-policy.json","project.json","state/live-authority.json","state/live/work-items.json","state/policies.json","storage-backend-policy.json","target.json","work-items.json"],"compatibility_projection":[],"derived_cache_index":["authority-index.json","claims.index.json","closed-gates.index.json","readme-history.index.json","state/indexes.json","state/indexes/closed-gates.index.json","state/indexes/work-items.index.json","work-items.index.json","work-items.projection.json"],"audit_history":["claims.jsonl","closed-gates.archive.jsonl","state/closed-gates.jsonl","state/closures/closed-gate-closures.jsonl","state/closures/work-items-closures.jsonl","state/closures/work-items/index.json","state/closures/work-items/manifest.json","state/closures/work-items/payloads/by-milestone/P1S1M1/part-001.jsonl","state/closures/work-items/payloads/by-milestone/P1S1M2/part-001.jsonl","state/closures/work-items/payloads/by-milestone/P1S2M1/part-001.jsonl","state/closures/work-items/payloads/by-milestone/P1S3M1/part-001.jsonl","state/closures/work-items/payloads/by-milestone/P1S3M2/part-001.jsonl","state/closures/work-items/payloads/by-milestone/P1S3M3/part-001.jsonl","state/closures/work-items/payloads/by-milestone/P1S3M4/part-001.jsonl","state/closures/work-items/payloads/by-milestone/P1S3M5/part-001.jsonl","state/closures/work-items/payloads/by-milestone/P1S3M5/part-002.jsonl","state/closures/work-items/payloads/by-milestone/P1S3M6/part-001.jsonl","state/closures/work-items/payloads/by-milestone/P1S3M7/part-001.jsonl","state/closures/work-items/payloads/by-milestone/P1S3M7/part-002.jsonl","state/closures/work-items/payloads/by-milestone/P2S1M1/part-001.jsonl","state/events/closed-gates.jsonl","state/events/work-items.jsonl"],"internal_release_dev":["packaged-router-port-inclusion.json","runtime-namespace.json","runtime-state-architecture-growth-arrest.json","source-size-budget-ratchet.json","state/authority/work-items/items/by-milestone/P1S1M1.json","state/authority/work-items/items/by-milestone/P1S1M2.json","state/authority/work-items/items/by-milestone/P1S2M1.json","state/authority/work-items/items/by-milestone/P1S3M1.json","state/authority/work-items/items/by-milestone/P1S3M2.json","state/authority/work-items/items/by-milestone/P1S3M3.json","state/authority/work-items/items/by-milestone/P1S3M4.json","state/authority/work-items/items/by-milestone/P1S3M5.json","state/authority/work-items/items/by-milestone/P1S3M6.json","state/authority/work-items/items/by-milestone/P1S3M7.json","state/authority/work-items/items/by-milestone/P2S1M1.json","state/authority/work-items/items/index.json","state/authority/work-items/manifest.json","state/authority/work-items/milestones.json","state/authority/work-items/programs.json","state/authority/work-items/stages.json","state/authority/work-items/vocabulary.json"],"migration_history_retire":["architecture-catalog-sharding.json","architecture-command-adapter-extraction.json","architecture-command-runner-extraction.json","architecture-runtime-service-residual-reduction.json","authority-command-adapter-extraction.json","cli-runtime-de-monolith-planning.json","cli-runtime-planning-service-extraction.json","closed-gate-state-layout-migration.json","command-adapter-extraction-service-extraction.json","compatibility-command-port-seed.json","contracts-command-service-extraction.json","core-command-adapter-extraction.json","core-surface-command-runner-extraction.json","exact-artifact-oracle-service-extraction.json","full-test-runner-service-extraction.json","god-file-budget-enforcement-closure.json","modular-runtime-package-inclusion-plan.json","package-command-adapter-extraction.json","package-surface-service-extraction.json","public-architecture-m1-closure-m2-seed.json","public-closed-gate-raw-artifact-prune-apply-admission.json","release-check-service-extraction.json","router-command-adapter-delegation-expansion.json","router-cutover-runtime-service-near-god-reduction.json","router-cutover-service-extraction.json","router-seed-service-extraction.json","runtime-agents-bridge-service-decomposition.json","runtime-check-fast-service-decomposition.json","runtime-command-registry-extraction.json","runtime-composer-decomposition.json","runtime-composer-god-file-exit.json","runtime-composer-residual-slice-reduction.json","runtime-guard-service-decomposition.json","runtime-mcp-bridge-service-decomposition.json","runtime-release-service-decomposition.json","runtime-surface-service-residual-reduction.json","source-domain-extraction-stabilization-closure-review.json","source-domain-runtime-service-near-god-reduction.json","source-extraction-golden-outputs.json","source-extraction-rehearsal.json","source-extraction-runtime-service-residual-reduction.json","source-module-extraction-adapter-boundary.json","source-module-extraction-authority-bundle-parity.json","source-module-extraction-authority-runtime-bridge.json","source-module-extraction-bundle-parity.json","source-module-extraction-claims-bundle-parity.json","source-module-extraction-claims-first-slice.json","source-module-extraction-claims-installed-fallback-smoke.json","source-module-extraction-claims-plan.json","source-module-extraction-claims-runtime-bridge.json","source-module-extraction-first-slice.json","source-module-extraction-installed-fallback-smoke.json","source-module-extraction-runtime-bridge.json","source-module-extraction-second-slice-first-slice.json","source-module-extraction-second-slice-plan.json","source-module-extraction-work-items-bundle-parity.json","source-module-extraction-work-items-first-slice.json","source-module-extraction-work-items-installed-fallback-smoke.json","source-module-extraction-work-items-plan.json","source-module-extraction-work-items-runtime-bridge.json","source-module-residual-service-extraction.json","source-partition-plan.json","state/live/closed-gates.json","target-command-adapter-extraction.json","target-command-runner-extraction.json","target-metadata-service-split.json","target-onboarding-acceptance-service-extraction.json","target-repo-product-refactor-plan.json","thin-cli-router-seed.json","thin-entrypoint-router-cutover-application.json","thin-entrypoint-router-cutover-rehearsal.json","work-item-ledger-compaction-migration.json","work-items-runtime-service-partition-seed.json"]},"semantic_probes":[{"id":"status","argv":["status"],"expected_schema":"agent-onboard-status-001","observed_schema":"agent-onboard-status-001","status":"ok","pass":true},{"id":"work-items-schema","argv":["work-items","--schema"],"expected_schema":"agent-onboard-work-items-schema-response-001","observed_schema":"agent-onboard-work-items-schema-response-001","status":"ok","pass":true},{"id":"work-items-template-validation","argv":["work-items","--validate-template"],"expected_schema":"agent-onboard-work-items-template-validation-001","observed_schema":"agent-onboard-work-items-template-validation-001","status":"ok","pass":true},{"id":"authority-state-check","argv":["authority","--state-check"],"expected_schema":"agent-onboard-public-authority-state-sharding-seed-check-result-001","observed_schema":"agent-onboard-public-authority-state-sharding-seed-check-result-001","status":"ok","pass":true},{"id":"authority-index-check","argv":["authority","--index-check"],"expected_schema":"agent-onboard-public-authority-compact-index-check-result-001","observed_schema":"agent-onboard-public-authority-compact-index-check-result-001","status":"ok","pass":true},{"id":"architecture-check","argv":["architecture","--check"],"expected_schema":"agent-onboard-public-architecture-check-result-001","observed_schema":"agent-onboard-public-architecture-check-result-001","status":"ok","pass":true},{"id":"release-surface-check","argv":["release","--surface-check"],"expected_schema":"agent-onboard-public-package-surface-preservation-check-result-001","observed_schema":"agent-onboard-public-package-surface-preservation-check-result-001","status":"ok","pass":true},{"id":"target-profile","argv":["target","profile","--json","--target","."],"expected_schema":"agent-onboard-target-profile-result-001","observed_schema":"agent-onboard-target-profile-result-001","status":"ok","pass":true}],"gate":{"g1_classification_complete":true,"external_fixture_corpus_complete":false,"protocol_deletion_authorized":false,"note":"W3 freezes the v1 surface classification. External fixture evidence in P2S1M2 remains required before protocol deletion."},"content_digest_sha256":"831cae85c4a101a09145d4491331019749f22022f2a6c2c3249a79926c3545ee"}
```

## 17. P2S1M2 W1 external fixture corpus contract

W1 freezes the corpus vocabulary and measurement record before any external repository snapshot is admitted. It does **not** download repositories, follow remote heads, or claim uncaptured fixture evidence. Each later fixture must be immutable by digest (and by exact revision when sourced from a repository snapshot), and measurement runs remain network-free.

The deterministic measurement contract covers the six quantities required by the dossier: detection correctness including false positives/negatives, files read, bytes read, output stability, and source-tree mutation hashes. The eight W3 semantic probes are referenced rather than duplicated so a later corpus run measures the already-frozen v1 semantic oracle.

```json
{"schema":"agent-onboard-p2-external-fixture-corpus-contract-001","work_item_id":"W1","source_package_version":"0.1.3","purpose":"freeze the external fixture classes and deterministic measurement record before any corpus snapshot is admitted","fixture_classes":[{"id":"node-library","description":"small Node.js library"},{"id":"node-application","description":"Node.js application"},{"id":"python-package","description":"Python package"},{"id":"go-cli","description":"Go command-line application"},{"id":"monorepo","description":"multi-package monorepo"},{"id":"minimal-repo","description":"minimal repository with only the smallest expected onboarding surface"},{"id":"messy-legacy-repo","description":"legacy repository with ambiguous or stale metadata"},{"id":"nested-agents-repo","description":"repository with nested AGENTS.md authority boundaries"},{"id":"external-work-provider-repo","description":"repository declaring an external work provider"},{"id":"spec-change-provider-repo","description":"repository declaring a spec/change provider"}],"snapshot_contract":{"remote_heads_forbidden":true,"fixture_manifest_required_fields":["fixture_id","fixture_class","source_kind","expected_detection","tree_digest_sha256"],"source_kind_enum":["synthetic","vendored","repository_snapshot"],"repository_snapshot_requires_immutable_revision":true,"path_rules":{"relative_only":true,"separator":"/","dot_dot_forbidden":true,"ordering":"UTF-8 bytewise ascending"},"file_digest":"sha256(raw file bytes)","tree_digest":"sha256(concat(path + NUL + decimal_byte_length + NUL + file_sha256_hex + LF) for all files in path order)","network_during_measurement":false},"probe_set":{"source":"P2S1M1 W3 semantic oracle","ids":["status","work-items-schema","work-items-template-validation","authority-state-check","authority-index-check","architecture-check","release-surface-check","target-profile"],"read_only_expected":true},"measurement_schema":{"schema":"agent-onboard-p2-external-fixture-measurement-001","required":["fixture_id","fixture_class","probe_id","expected_detection","observed_detection","detection_outcome","files_read_count","bytes_read","files_read_digest_sha256","stdout_sha256","stderr_sha256","exit_code","repeat_count","output_stable","pre_tree_digest_sha256","post_tree_digest_sha256","mutation_detected","pass"],"expected_detection_enum":["positive","negative"],"observed_detection_enum":["positive","negative"],"detection_outcome_rule":"positive/positive=tp; negative/negative=tn; negative/positive=fp; positive/negative=fn","detection_outcome_enum":["tp","tn","fp","fn"],"files_read_digest":"sha256(concat(path + NUL + decimal_bytes_read + LF) for unique read paths in UTF-8 bytewise order)","stdout_sha256":"sha256(stdout bytes after CRLF-to-LF normalization only)","stderr_sha256":"sha256(stderr bytes after CRLF-to-LF normalization only)","repeat_count_minimum":2,"output_stable_rule":"all repeated exit_code/stdout_sha256/stderr_sha256 triples are identical","mutation_detected_rule":"pre_tree_digest_sha256 != post_tree_digest_sha256","pass_rule":"detection_outcome in [tp,tn] AND output_stable=true AND mutation_detected=false for read-only probes","canonical_record_order":["fixture_id","probe_id","run_index"],"timestamps_excluded_from_semantic_digest":true},"aggregate_schema":{"schema":"agent-onboard-p2-external-fixture-corpus-report-001","required":["fixture_count","fixture_class_count","probe_count","measurement_count","tp","tn","fp","fn","false_positive_rate","false_negative_rate","total_files_read","total_bytes_read","unstable_measurement_count","mutation_count","measurements_digest_sha256"],"measurements_digest":"sha256(LF-joined canonical JSON measurement records sorted by fixture_id, probe_id, run_index)","division_by_zero_rule":"rate is null when its denominator is zero"},"acceptance":{"fixture_class_count":10,"probe_count":8,"corpus_snapshot_required_in_this_work_item":false,"measurement_schema_frozen":true,"external_fixture_corpus_complete":false,"protocol_deletion_authorized":false,"next_corpus_capture_requires_separate_work_item":true}}
```

W1 freezes only the contract. `external_fixture_corpus_complete` remains false and protocol deletion remains unauthorized until later work items capture representative immutable fixtures and produce measurements conforming to this schema.

P2S1M2 admission/closure adds exactly two canonical authority files: one item shard and one closure-payload shard. The clean baseline moves 350→352 source files and 134→136 `.agent-onboard` files; command, schema, package-surface, and generalized size ceilings do not increase.

