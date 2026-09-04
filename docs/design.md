# Release Tooling and Process Design

Status: Implemented

Last updated: 2026-07-28

## Background

This document is for historical reference - it was created while seeking to replace the brittle and failure-prone release process of the [compiler](https://github.com/0xMiden/compiler) based on `release-plz`, with a bespoke tool that would provide significantly increased confidence in the release process, and provide us a path to immutable releases and Trusted Publishing-based deployment.

While the current `release-tool` has evolved from the original design, which was tailored specifically to the compiler, many of the core design decisions still underpin the implementation today. Key features:

* Release one or more release units (read: groups) from a repo which contains a Cargo package or workspace.
* Support release units which are or contain prebuilt artifacts/assets
* Native support for template bundles
* Support for granular versioning, managed at the unit level
* Designed for Trusted Publishing, immutable releases, and GitHub build attestations
* Release configuration can be linted/verified before publishing ever happens
* Every step is recoverable/resumable - if something goes wrong, it can be fixed, and the release retried from where it failed

## 1. Summary

This document proposes replacing `release-plz` with release tooling owned by
the compiler repository. The tooling is intentionally specific to this
repository: it knows about the compiler, Rust SDK, project templates, the three
released executables, and the ordering constraints between those units.
Generality beyond those needs is not a goal.

The design has four main parts:

1. A small Rust release tool with a deterministic planner and a resumable
   executor.
2. Thin `Makefile.toml` tasks for local version, SemVer, changelog, packaging,
   and release preparation operations.
3. A small set of GitHub Actions workflows that run the same reusable
   preparation logic in pull requests, rehearsals, and production.
4. Explicit maintainer checkpoints for release preparation, protected tag
   creation, environment approval, and recovery decisions.

The repository has three independently releasable units:

- The compiler crates and the `midenc`, `cargo-miden`, and `miden-objtool`
  executables.
- The Rust SDK crates under `sdk/`.
- A single project-template bundle maintained in this repository.

The SDK may be released without the compiler, the compiler may be released
without the SDK when its dependency requirements are already satisfied, and
both may be released together. In a combined release, SDK crates are always
published and verified before compiler crates.

Routine crates.io publication uses Trusted Publishing exclusively. GitHub
releases are created as drafts, populated and verified before publication, and
made immutable only after every selected crate and artifact has been verified.
The preferred high-fidelity rehearsal publishes uniquely versioned packages to
`staging.crates.io` and exercises disposable GitHub drafts; the same GitHub
draft rehearsal with registry publication disabled is the fallback.

## 2. Goals

The release system must:

- Use crates.io Trusted Publishing for routine production publication.
- Use GitHub immutable releases and release attestations.
- Generate explicit build provenance attestations for released artifacts.
- Build and attest `midenc`, `cargo-miden`, and `miden-objtool` for:
  - `aarch64-apple-darwin`
  - `x86_64-unknown-linux-gnu`
- Release compiler, SDK, and template units independently or in supported
  combinations.
- Enforce SDK-before-compiler publication when both are selected.
- Manage shared compiler versions and independently versioned crate versions
  without ad hoc manifest editing.
- Detect SemVer-breaking changes that are not accompanied by an appropriate
  version bump.
- Support agent-assisted changelog preparation without running an LLM in a
  privileged release workflow.
- Keep the substantive release logic reusable locally and in CI.
- Detect release-tool, manifest, packaging, and workflow regressions on the
  pull request that introduces them.
- Be resumable after partial crates.io publication.
- Produce precise diagnostics and a durable record of what was planned,
  attempted, observed, and finalized.

## 3. Non-goals

The release tool does not need to:

- Be usable by arbitrary Rust workspaces.
- Infer every policy from Cargo metadata.
- Replace Cargo's package-building behavior or reimplement the crates.io upload
  protocol.
- Generate final changelog prose without maintainer review.
- Make multi-crate crates.io publication atomic; crates.io does not provide
  transactions or rollback.
- Hide production-only uncertainty behind a nominal `--dry-run` mode.
- Automatically make discretionary release decisions such as whether a change
  is user-visible, whether a release should proceed after an external outage,
  or whether a failed partial release should be resumed immediately.
- Rewrite or delete already-published immutable releases.

## 4. Terminology and Release Units

### 4.1 Release unit

A release unit is a product-level group validated and orchestrated together.
It is not necessarily a single version domain.

| Unit | Version model | Primary tag | GitHub release | Changelog |
| --- | --- | --- | --- | --- |
| Compiler | Shared workspace version | `vX.Y.Z` | Compiler release | Root `CHANGELOG.md` |
| Rust SDK | Independently versioned crates; `miden` is the aggregate SDK release version | `sdk/vX.Y.Z` | SDK release | Per-crate changelogs, aggregated into release notes |
| Templates | Independent compatibility SemVer | `templates/vX.Y.Z` | Template release | Template changelog |

The compiler keeps the existing unprefixed `vX.Y.Z` tag convention to avoid
unnecessary ecosystem migration.

Every independently published SDK crate also receives a protected baseline tag
of the form `crate/<crate-name>/vX.Y.Z`. These tags do not create GitHub
releases. They provide unambiguous changelog and SemVer commit ranges.

Every SDK release bumps the aggregate `miden` crate version. That version names
the SDK GitHub release even when some independently versioned leaf crates are
unchanged.

### 4.2 Release candidate

A release candidate is a reviewed commit whose versions, dependency
requirements, lockfile, changelogs, and template metadata are intended for
publication. It is not a crates.io prerelease unless the selected versions
explicitly contain prerelease identifiers.

The candidate contains `.release/release.toml`, a committed declaration of:

- Selected release units.
- Expected package and bundle versions.
- Expected primary and per-crate tags.
- Template compatibility track and expected bundle digest.
- Changelog and release-note targets.

Production derives its scope from this declaration. Workflow inputs cannot
widen or narrow it.

### 4.3 Release subject and executor

The release subject is the protected tag and commit whose source is packaged
and published. The release executor is the reviewed commit on protected `main`
that supplies `release.yml`, the reusable workflows, and the release-tool
binary.

These commits are normally close or identical, but are recorded separately. A
newer compatible executor may resume a partially published release subject
without moving its tag or changing its sealed plan.

### 4.4 Release plan

A release plan is a canonical machine-readable description of one attempted
release subject executed by one reviewed executor. It includes inputs, expected
outputs, publication order, digests, and preconditions.

### 4.5 Release intent

A release intent is the canonical pre-build description of the subject and
executor commits, committed unit scope, versions, dependency order, tags, and
expected artifact names. Preparation consumes the intent and seals a release
plan only after the exact packages and artifacts exist and their digests are
known.

### 4.6 Rehearsal projection

A rehearsal projection is a deterministic transformation of a production
intent into unique staging versions and registry references. It must preserve
package membership, dependency edges, dependency stages, release-unit ordering,
binary source inputs, and template checks. It does not claim that staging
`.crate` bytes or permissions are identical to production.

### 4.7 Release journal

A release journal is an append-only record of executor operations and observed
external state. It supports diagnosis and resumption, but registry and GitHub
state remain authoritative.

## 5. Responsibility Boundaries

### 5.1 One-time maintainer or administrator work

The following work cannot be performed safely or completely by repository code
alone:

- Enable immutable releases for the compiler repository.
- Create protected release environments:
  - `release-production`
  - `crates-io-staging`, if the staging spike succeeds
- Configure required reviewers, prevent self-review, disable administrator
  bypass where supported, and restrict production deployment to protected
  `main`. The workflow separately validates its release-subject tag input.
- Configure branch and tag rulesets for:
  - `v*`
  - `sdk/v*`
  - `templates/v*`
  - `crate/*/v*`
- Require the `release / gate` status and release-infrastructure CODEOWNERS.
- Configure a Trusted Publisher for every production crate using the production
  top-level workflow and environment.
- Enable Trusted-Publishing-only mode for every production crate after its
  publisher is verified.
- Bootstrap any brand-new crate once using a short-lived, narrowly scoped token;
  crates.io cannot configure a Trusted Publisher before the crate exists.
- Determine whether `staging.crates.io` accounts, crate ownership, third-party
  dependencies, and Trusted Publishing are sufficiently usable for this
  repository.
- Bootstrap staging crates and publishers if staging rehearsal is adopted.
- Seed protected per-crate baseline tags for the latest already-published SDK
  versions before changelog and SemVer automation begins using them.
- Import template history from the existing template repositories and leave
  their published tags readable for old `cargo-miden` versions.
- Archive or mark the old template repositories read-only only after the
  in-repository resolver has shipped.
- Disable every legacy publication trigger before the first production pilot.
- Remove the long-lived production crates.io token as soon as Trusted
  Publishing is configured and its auth-only check succeeds.

These settings must be recorded in the release runbook. Where GitHub exposes a
read-only API, a scheduled configuration-drift audit should compare the live
configuration with checked-in expectations. Settings without a suitable API
remain a periodic manual audit.

### 5.2 Per-release maintainer work

A maintainer is responsible for:

- Deciding which release units and versions should be released.
- Running version tasks and reviewing their diffs.
- Running `changelog-prompt`, invoking an agent if desired, and reviewing all
  changelog changes.
- Preparing and merging the release-candidate pull request.
- Reviewing the committed `.release/release.toml` scope declaration.
- Confirming that release notes, versions, dependency changes, and template
  compatibility claims are correct.
- Creating or approving creation of the protected release tags at the reviewed
  commit.
- Dispatching the production workflow from protected `main` with the intended
  protected release-subject tag. The workflow does not accept a release-unit
  override.
- Reviewing the final release plan, crate list, artifact list, digests, and
  outstanding warnings.
- Approving entry to the protected production environment.
- Deciding whether to retry or pause after an external-service or partial
  publication failure.
- Supervising the first production releases performed by the new tooling.

The tool may generate commands and validate their effects, but it must not
silently decide to publish, change release scope, bypass an approval, or proceed
past a conflicting external state.

### 5.3 Automated work

The release tool and workflows are responsible for:

- Discovering the configured package graph and rejecting unclassified
  packages.
- Computing dependency order within and between release units.
- Editing versions and local dependency requirements transactionally.
- Updating and validating `Cargo.lock`.
- Selecting SemVer baselines and running configured checks.
- Producing changelog prompts and structural changelog validation.
- Building and verifying exact `.crate` package closures.
- Building, packaging, hashing, transferring, and smoke-testing executables.
- Producing template bundles and testing compatibility matrices.
- Attesting already-built bytes in jobs that do not execute release-subject
  code.
- Generating the canonical release plan and journal.
- Enforcing a repository-wide production mutation lock and rejecting
  overlapping active plans.
- Creating and populating GitHub draft releases.
- Obtaining temporary crates.io credentials in protected jobs.
- Publishing one crate at a time in dependency order.
- Polling registry state and reconciling resumptions by version and checksum.
- Verifying every selected crate and artifact.
- Running clean consumer resolution, install, generated-template, and
  downloaded-draft-asset smoke tests before finalization.
- Publishing draft GitHub releases only after all required postconditions pass.
- Producing post-release verification results and actionable diagnostics.

## 6. Proposed Repository Layout

```text
.release/
├── config.toml
├── release.toml
├── schema-version
├── semver-waivers.toml
└── workflow-policy.toml

tools/release/
├── Cargo.toml
├── src/
└── tests/
    ├── fixtures/
    ├── snapshots/
    └── protocol/

tools/cargo-miden/templates/
├── bundle.toml
├── project/
├── program/
├── account/
├── note/
├── tx-script/
└── auth-component/

.github/workflows/
├── release-ci.yml
├── release-verify.yml
├── release-rehearsal.yml
└── release.yml
```

`tools/release` is a private workspace package with `publish = false`.

`.release/config.toml` is the authoritative release policy. It explicitly
classifies packages and does not infer release membership from directory names
or broad workspace globs.

`.release/release.toml` is the reviewed declaration for the current release
candidate. Git history preserves prior declarations; a later release replaces
it.

An illustrative configuration shape is:

```toml
schema-version = 1

[units.compiler]
version-source = "workspace"
tag = "v{version}"
changelog = "CHANGELOG.md"
publish-after-if-selected = ["sdk"]

[units.sdk]
aggregate-package = "miden"
tag = "sdk/v{miden-version}"

[units.templates]
manifest = "tools/cargo-miden/templates/bundle.toml"
tag = "templates/v{version}"

[[packages]]
name = "midenc"
unit = "compiler"
publish = true

[[packages]]
name = "miden-base"
unit = "sdk"
publish = true
changelog = "sdk/base/CHANGELOG.md"

[[packages]]
name = "midenc-integration-tests"
unit = "private"
publish = false

[[artifacts]]
package = "midenc"
binary = "midenc"
unit = "compiler"
targets = ["aarch64-apple-darwin", "x86_64-unknown-linux-gnu"]
```

The exact schema will be chosen during implementation, but it must make every
package classification and released artifact explicit.

An illustrative release-candidate declaration is:

```toml
schema-version = 1
units = ["sdk", "compiler", "templates"]

[compiler]
version = "0.10.0"
tag = "v0.10.0"

[sdk]
version = "0.14.0"
tag = "sdk/v0.14.0"

[sdk.packages]
miden = "0.14.0"
miden-base = "0.13.1"

[templates]
version = "2.0.0"
tag = "templates/v2.0.0"
track = 2
bundle-sha256 = "..."
```

The version tasks may update this declaration, but the maintainer reviews and
commits it with the release candidate.

## 7. Package and Workspace Policy

Before replacing the current workflow:

- Replace broad workspace member globs with explicit members.
- Set every package to either:
  - `publish = false`, or
  - an explicit allowlist of accepted registries.
- Audit currently publishable internal crates and mark them private unless
  external publication is intentional.
- Reject a publishable crate that has a normal or build dependency on a private
  crate.
- Reject unversioned path or git dependencies in normalized release packages.
- Require every publishable package to contain crates.io-required metadata.
- Require every package to be classified exactly once in `.release/config.toml`.
- Require all configured package paths and changelogs to exist.

If staging publication requires a named alternate registry, the initial staging
spike will determine whether public manifests should allow both
`crates-io` and `crates-io-staging`, or whether staging-only manifest changes
should be applied in an isolated temporary checkout. Production manifests must
never be modified in-place by a rehearsal.

## 8. Release Tool Architecture

### 8.1 Compiler-specific design

The tool may encode repository-specific concepts directly:

- The three release units.
- The three released executable packages.
- SDK-before-compiler ordering.
- Compiler workspace-version inheritance.
- SDK independent versions and aggregate `miden` version.
- Miden template compatibility tracks.
- The repository's tag naming and changelog layout.

It should not introduce a general plug-in system, generic release DSL, or
support for unrelated package managers unless an actual repository requirement
emerges.

Repository policy should still be data-driven where it improves reviewability,
particularly package classification and artifact inventory.

### 8.2 Functional core and effectful shell

The planner is a deterministic function of:

- The release-subject tag and commit.
- The release-executor commit.
- The committed `.release/release.toml` scope.
- Workspace/package metadata.
- Release configuration.
- Git history and tags.
- A captured read-only snapshot of relevant crates.io and GitHub state.

It produces a canonical `release-intent.json`. Planning cannot publish, create
tags, create releases, or request credentials.

Preparation consumes that intent, creates and verifies the exact `.crate`,
binary, archive, and template outputs, and then seals `release-plan.json` with
their digests. A sealed plan cannot be edited; any changed input or output
requires a new intent and plan.

The executor consumes the plan through explicit adapters:

```text
CargoRunner
RegistryClient
GitHubClient
AttestationVerifier
Clock
Sleeper
JournalStore
```

Production uses real adapters. Unit and protocol tests use scripted fakes and
local HTTP servers. Staging is represented by a rehearsal projection derived
from the production intent, not by a separate planner. Projection validation
proves that package membership, dependency stages, unit ordering, binary inputs,
and template checks remain unchanged even though versions and registry
references differ.

### 8.3 Plan contents

The release plan includes:

- Schema version and release-tool version.
- Release-subject tag/commit and clean-tree assertion.
- Release-executor commit and supported plan-schema range.
- Candidate-declaration digest.
- Selected release units.
- Expected primary and per-crate tags.
- Old and new package versions.
- Complete publishable package closure.
- Stable topological publication stages.
- Registry baselines and expected absence/presence of versions.
- `.crate` package paths, sizes, normalized metadata, and SHA-256 digests.
- Binary and template build inputs.
- Final asset names and SHA-256 digests.
- Changelog files and expected version headings.
- Required GitHub draft releases and planned payload asset inventories; the
  write-once plan does not attempt to include its own digest.
- Required attestations and expected signer workflow.
- External configuration preconditions that can be checked automatically.
- Warnings requiring explicit maintainer acknowledgement.

Timestamps are excluded from canonical plan identity. A plan digest is computed
over canonical serialized content.

Cargo does not accept an already-created `.crate` archive as input to
`cargo publish`. Production therefore runs Cargo from the same clean tagged
source and pinned toolchain used during preparation, and treats the prepared
package digest as an expected postcondition. The published crates.io checksum
must match it. Package generation is repeated before the first irreversible
operation to measure byte stability under the pinned Cargo/toolchain.
Credential-bearing publication uses `--no-verify` because the package closure
has already been built and tested without credentials; this prevents package
build scripts from running while a crates.io token is present.
Uploading a prebuilt archive directly through the registry web API would
provide a stricter byte-identity guarantee, but reimplementing that upload path
is outside the initial design and would require a separate review.

### 8.4 Journal and resumability

Each operation records:

- Operation identifier and plan digest.
- Start and completion observations.
- External object identifiers.
- Expected and observed versions/digests.
- Retry classification and diagnostics.

The journal is diagnostic, not blindly trusted. Before skipping work on resume,
the executor queries external state:

- A crate step is complete only if the expected version exists with the expected
  checksum.
- A draft asset step is complete only if the expected name and digest exist.
- A finalization step is complete only if the release is immutable and its
  release/asset attestations verify.

Conflicting state is a hard failure. Missing state is eligible for retry.

Release control data is divided as follows:

- Payload assets are binaries, archives, template bundles, checksums, and
  release notes.
- `release-plan-<digest>.json` is write-once, does not list its own digest, and
  is copied to every selected draft.
- Journal segments use unique
  `release-journal-<attempt>-<sequence>.json` names and are never overwritten.
- One selected coordination draft owns recovery journal segments; compiler is
  preferred when selected, otherwise templates, otherwise SDK.
- `release-manifest.json` is generated immediately before finalization. It
  lists every public payload and write-once control asset but excludes itself.

Journal segments are removed from the coordination draft before finalization
and retained with the GitHub Actions run. The public immutable release retains
the sealed plan and final manifest. Post-finalization verification cannot alter
an immutable release and is stored in the workflow record.

### 8.5 Executor compatibility and recovery

Normal releases use the executor currently on protected `main` and package the
subject tag in a separate checkout. The sealed plan records the subject commit
and the executor that created the plan. Every journal segment also records the
executor for that attempt, so a recovery executor is auditable without
rewriting the sealed plan.

If publication is partial and the executor is defective:

1. Fix and review the executor on `main`.
2. Run compatibility tests against the older sealed plan schema.
3. Dispatch recovery with the original release-subject tag and plan digest.
4. Allow only operations missing from that sealed plan.

Recovery may not change versions, unit scope, tags, package checksums, artifact
digests, or final release inventory. An incompatible plan-schema change
requires an explicit, tested plan migration that preserves those fields.

### 8.6 Concurrency

Production uses one repository-wide mutation concurrency group from external
state capture through finalization. Cancellation is disabled; later runs queue.
Rehearsals use a separate group.

The executor also queries active drafts and plans and rejects any overlap in
crate/version, primary or baseline tag, or draft release. This prevents
independent SDK and compiler dispatches from bypassing cross-unit ordering.

### 8.7 Commands

The initial command surface should be small and phase-oriented:

```console
release-tool lint
release-tool set-workspace-version [VERSION]
release-tool set-crate-version CRATE [VERSION]
release-tool semver-checks [OPTIONS]
release-tool changelog-prompt TARGET [RANGE]
release-tool plan --candidate .release/release.toml --subject-ref REF \
  --output release-intent.json
release-tool prepare --intent release-intent.json --output release-plan.json
release-tool verify-packages --plan PLAN
release-tool stage-github --plan PLAN
release-tool publish-crates --plan PLAN
release-tool resume --subject-tag TAG --plan-digest DIGEST
release-tool finalize --plan PLAN
release-tool verify --plan PLAN
```

Commands that cross irreversible boundaries must require:

- A canonical plan produced from the current exact commit.
- A clean working tree.
- An explicit target registry.
- A matching protected GitHub environment in CI.
- Explicit confirmation locally; routine production publication is expected to
  occur only in GitHub Actions.

## 9. Version Management

### 9.1 `set-workspace-version`

Invocation:

```console
cargo make set-workspace-version -- [VERSION]
```

If `VERSION` is absent, `X.Y.Z` becomes `X.(Y+1).0`.

The task:

- Updates the root workspace version.
- Updates every compiler package inheriting the workspace version.
- Updates all intra-repository dependency requirements for those packages.
- Does not alter independently versioned SDK crates except where their
  dependency requirements intentionally reference compiler crates.
- Refreshes `Cargo.lock`.
- Fails on downgrades, invalid SemVer, stale old requirements, unexpected
  package membership, or unrelated manifest changes.
- Prints a summary of changed packages and dependents requiring review.

### 9.2 `set-crate-version`

Invocation:

```console
cargo make set-crate-version -- CRATE [VERSION]
```

The crate name is required. The version defaults to the next minor.

The task:

- Updates the selected independently versioned crate.
- Updates every intra-repository version requirement referring to it.
- Refreshes `Cargo.lock`.
- Reports dependent packages whose published metadata changed and which may
  therefore need their own version bump.
- Does not silently cascade version bumps.
- Rejects private packages, unknown packages, workspace-versioned compiler
  crates, downgrades, and version collisions.

### 9.3 CI version policy

`release-tool lint` compares the pull-request merge base with the proposed tree
and checks:

- Version changes are monotonic and complete.
- All local dependency requirements agree with selected versions.
- `Cargo.lock` is current.
- A breaking public API change has an appropriate manifest version.
- Every independently versioned published crate has a valid baseline.
- A release-candidate version does not already exist with conflicting content.
- Aggregate SDK release metadata is consistent.

A source change does not require final changelog prose on every pull request.
Final changelog headings, dates, and links are enforced when validating a
release candidate.

## 10. SemVer Validation

The `Makefile.toml` task is:

```console
cargo make semver-checks -- [--unit compiler|sdk] [--package CRATE]
```

The task runs `cargo-semver-checks` separately for every configured changed
publishable library:

- The default baseline is the latest published crates.io version.
- An unchanged version with a breaking API change fails.
- A changed version is evaluated according to Cargo SemVer rules, including
  pre-1.0 compatibility.
- Each crate declares the feature sets and supported target triples whose public
  APIs must be checked. Native Linux and macOS jobs cover the currently
  supported platform-dependent APIs; crates with additional public `cfg`
  surfaces list them explicitly.
- Exit status for a SemVer violation is distinguished from a tool/build
  failure.
- A new crate may skip its baseline only if it is explicitly marked new and its
  bootstrap state is acknowledged.

The `cargo-semver-checks` version and compatible Rust toolchain are pinned.

False-positive waivers are allowed only in `.release/semver-waivers.toml` and
must identify:

- Crate.
- Baseline and candidate version/range.
- Specific lint.
- Rationale.
- Reviewer.
- Expiration or removal condition.

Tool failures and blanket crate-wide suppression cannot be waived.

## 11. Changelogs and Release Notes

### 11.1 Changelog ownership

- Compiler workspace-versioned crates share the root `CHANGELOG.md`.
- The root changelog groups entries under user-facing product headings:
  - Compiler and `midenc`
  - `cargo-miden`
  - `miden-objtool`
  - Libraries and public APIs
  - Migration and breaking changes
- Independently versioned SDK crates keep individual changelogs.
- SDK GitHub release notes aggregate significant per-crate changes.
- Templates keep one template changelog.
- Stale compiler per-crate changelogs are frozen or removed as release sources
  of truth after migration.

### 11.2 `changelog-prompt`

Invocation:

```console
cargo make changelog-prompt -- TARGET [BASE..HEAD]
```

The task emits a prompt to stdout and performs no external write.

The prompt contains:

- Exact base and head commits.
- Changed packages and relevant paths.
- Commit and pull-request summaries.
- Old and proposed versions.
- Target changelog files and expected section structure.
- Instructions to omit merges, release PRs, typo-only changes, and non-user-
  visible test or refactor commits.
- Instructions to identify user impact, migration work, security impact, and
  affected tools.
- Instructions not to invent behavior and to preserve existing changelog style.

Default ranges are:

- Previous compiler tag to `HEAD` for the compiler unit.
- Previous per-crate baseline tag to `HEAD` for independently versioned crates.
- Previous compatible template tag to `HEAD` for templates.

The maintainer may pipe the prompt to an agent, but reviews and commits the
result manually. No LLM is invoked by the release workflow.

### 11.3 Structural validation

Release-candidate validation requires:

- Correct version headings and links.
- No placeholder text.
- Required breaking/migration sections when SemVer reports breaking changes.
- Every bumped independently versioned crate has a corresponding changelog.
- Aggregated release notes reference the selected crate/template versions.

## 12. Template Ownership and Distribution

The two external template repositories are consolidated into one bundle under
`tools/cargo-miden/templates/`.

`bundle.toml` records:

- Template bundle version.
- Bundle format/schema version.
- Accepted `cargo-miden` compatibility range.
- Compiler/SDK compatibility claims used by CI.
- Template inventory.
- Expected generated-project checks.

Template SemVer defines compatibility tracks:

- Patch: compatible fixes and dependency corrections.
- Minor: additive changes usable by existing clients in the track.
- Major: renderer/schema changes or generated projects requiring incompatible
  compiler/SDK behavior.

Each `cargo-miden` release:

- Embeds the in-tree bundle as its offline/reproducible fallback.
- Embeds an accepted template version range.
- May query the compiler repository's GitHub releases for `templates/v*`.
- Selects the highest stable immutable compatible release.
- Verifies the release asset digest before extraction.
- Caches by version and digest.
- Falls back to a compatible cache and then the embedded bundle.
- Supports `--template-path`, `--template-version`, and `--offline`.
- Records the selected template version and digest in the generated project.

A compiler release must prove that its embedded template bundle is byte-
identical to either:

- An existing compatible immutable `templates/v*` release, or
- A template release in the same committed release declaration and sealed plan.

In the second case, the template release is finalized before the compiler
release. A compiler release cannot point to a bundle version/digest that is
neither already immutable nor being released in the same plan.

Template content may evolve under one not-yet-released bundle version. Once an
immutable release exists for that version, any content or digest change
requires a new version. Release-candidate validation compares the committed
bundle version/digest with existing immutable releases and seals the intended
digest in `.release/release.toml`.

Maintenance branches such as `templates-v1` may produce compatible fixes for
old tracks while development proceeds on a breaking new track in the main
branch.

Existing `cargo-miden` binaries that hardcode external repository tags cannot be
retrofitted. Their referenced repositories and tags remain readable.

Template validation includes:

- Rendering every template.
- Building generated projects against the prepared package closure.
- Testing the oldest supported client in the compatibility track.
- Testing the in-tree current compiler/SDK stack.
- For template-only releases, testing the configured matrix of already-
  published compatible compiler, SDK, and `cargo-miden` versions rather than
  assuming a same-run crate publication.
- Verifying deterministic bundle contents and digest.
- Rejecting unexpected archive entries, absolute or parent-traversal paths,
  disallowed symlinks, duplicate paths, excessive expanded size, and excessive
  file counts before extraction.

## 13. Binary Artifacts and Attestations

The compiler release produces six executable artifacts:

| Executable | macOS target | Linux target |
| --- | --- | --- |
| `midenc` | `aarch64-apple-darwin` | `x86_64-unknown-linux-gnu` |
| `cargo-miden` | `aarch64-apple-darwin` | `x86_64-unknown-linux-gnu` |
| `miden-objtool` | `aarch64-apple-darwin` | `x86_64-unknown-linux-gnu` |

Builds use:

- Explicit native runner labels rather than `*-latest`.
- A pinned Rust toolchain.
- `cargo build --release --locked --target <target>`.
- Incremental compilation disabled.
- Path remapping and reproducibility settings where practical.

Each executable is:

- Checked for the expected executable format and architecture.
- Smoke-tested with `--version` and a tool-specific meaningful operation.
- Packaged into a deterministic one-binary archive with fixed entry order,
  timestamps, ownership, modes, and compressor settings.
- Listed in `SHA256SUMS` and `release-manifest.json`.

Both the raw executable and exact uploaded archive receive build provenance
attestations. The raw executable attestation allows verification after archive
extraction; the archive attestation verifies the downloaded release asset.
The deterministic template archive also receives build provenance when a
template release is selected.

The same bytes flow from the build job through workflow artifacts to the draft
release. The finalizer never rebuilds or repackages them.

Bit-for-bit deterministic archives are a hard requirement. Bit-for-bit
deterministic compiler output is a goal and scheduled diagnostic; a documented
compiler/linker nondeterminism does not invalidate provenance for the exact
released bytes.

## 14. GitHub Actions and Security

### 14.1 Workflow structure

`release-ci.yml`:

- Runs on `pull_request` and `merge_group`.
- Has no write permissions, secrets, release environment, or production OIDC
  path.
- Calls `release-verify.yml` from the pull request commit.

`release-verify.yml`:

- Is reusable through `workflow_call`.
- Runs release lint, planning, SemVer, package-closure verification, template
  tests, and binary build/package/smoke tests.
- Transfers artifacts between jobs and verifies digests.
- Can optionally attest artifacts only when called from an approved hosted
  rehearsal or production caller with the necessary permissions.

`release-rehearsal.yml`:

- Is manually dispatched and may also run on a schedule from the default
  branch.
- Prefers `staging.crates.io` publication when configured and healthy.
- In the preferred path, also creates, uploads, verifies, and cleans up
  disposable GitHub drafts.
- Supports the same GitHub draft rehearsal with registry publication disabled
  as its fallback.
- Cannot enter the production crates.io environment.

`release.yml`:

- Is the only workflow configured as a production crates.io Trusted Publisher.
- Is manually dispatched from protected `main` with one protected
  release-subject tag input.
- Loads release scope exclusively from `.release/release.toml` at the subject
  commit and rejects workflow attempts to alter it.
- Records the executor and subject commits independently and checks out subject
  source separately from executor code.
- Calls the same `release-verify.yml`.
- Contains only orchestration and privileged production phases.

### 14.2 Permissions

Use job-level least privilege:

| Job class | Permissions |
| --- | --- |
| Plan/validate/package | `contents: read` |
| Build/package | `contents: read` |
| Attest verified bytes | `contents: read`, `id-token: write`, `attestations: write` |
| crates.io publish | `contents: read`, `id-token: write` |
| Draft/final release management | `contents: write` |

The top-level default is `permissions: {}` or `contents: read`.

All third-party Actions are pinned to full commit SHAs. `GITHUB_TOKEN` is used
for GitHub operations; no PAT is used by the release workflow. Checkouts use
`persist-credentials: false`.

Build/package jobs have no OIDC or registry credentials. Attestation jobs
download and digest-check workflow artifacts but do not check out or execute
release-subject code. This keeps subject build scripts out of jobs capable of
requesting an OIDC token.

The production workflow does not use `pull_request_target` or `workflow_run`.
Privileged publication is not triggered by a chained event.

### 14.3 Trusted Publishing

Every publishable crate is configured independently with:

- Organization/repository.
- Top-level production workflow filename.
- `release-production` environment.

The publish job enters the protected environment, obtains a temporary token
using the official crates.io authentication action immediately before the
relevant publication stage, and allows the action's post-step to revoke it.
The token is scoped to the individual Cargo command rather than exported for
later steps. Publication uses the pinned Cargo version with
`cargo publish --no-verify --locked --registry crates-io`; all build and
package verification has already happened in credential-free jobs.

SDK and compiler publication may use separate jobs/token acquisitions so a
long build or SDK publish does not consume the compiler token lifetime.

An auth-only production mode may obtain and revoke the temporary token without
calling Cargo. It verifies the real workflow/repository/environment OIDC path,
but cannot prove per-crate publication authorization.

### 14.4 Immutable GitHub releases

For every selected unit:

1. The protected tag identifies the reviewed release commit.
2. A draft GitHub release is created for that tag.
3. The complete payload asset set and write-once sealed plan are uploaded;
   recovery journal segments are attached only to the coordination draft.
4. Assets are downloaded and verified while the release is still mutable.
5. Crates and/or templates are published and verified.
6. Fresh consumer resolution/install, generated-template, and downloaded-
   artifact smoke tests pass.
7. Recovery journal segments are removed, the final manifest is added, and all
   public assets, digests, and attestations are reverified.
8. The draft is published and becomes immutable.
9. GitHub automatically creates the immutable release attestation.
10. `gh release verify`, `gh release verify-asset`, and
   `gh attestation verify` run as postconditions.

SDK and template releases set `make_latest=false`. Stable compiler releases may
become the repository's latest release; compiler prereleases set
`make_latest=false`.

Finalization is the last operation. If correction is needed after finalization,
a new version is released.

## 15. Testing and Pull-Request Validation

### 15.1 Required release gate

One always-running `release / gate` check is required for pull requests and
merge-queue commits. It does not use top-level path filters.

The workflow computes change impact internally. Its terminal aggregation job
runs with `always()` and fails if a required child job failed or was
unexpectedly skipped.

Every pull request runs:

- Release-tool formatting, Clippy, unit, property, and fixture tests.
- Configuration and schema validation.
- Complete package classification and publishability checks.
- Intent generation twice with byte-identical canonical output.
- Version, dependency requirement, and lockfile consistency checks.
- Changed-package SemVer checks.
- Changelog structural checks appropriate to the change.
- `actionlint`.
- `zizmor`.
- Repository-specific workflow policy lint.

The workflow-policy lint asserts that:

- Only `release.yml` can enter the production environment.
- Production dispatch accepts only a subject tag, not release scope or
  versions.
- Subject-code build/package jobs have neither OIDC nor registry credentials.
- The crates.io token is passed only to a pinned
  `cargo publish --no-verify --locked --registry crates-io` invocation.
- No long-lived crates.io secret is referenced.

Planner determinism tests capture one canonical external-state snapshot and
feed that same snapshot to both runs. Live GitHub/crates.io state is re-read and
revalidated separately before mutation; changing external state is not folded
into a determinism assertion.

CI fetches the release-subject commit and required baseline tags explicitly
rather than relying on a shallow checkout. Base-SHA selection is defined
separately for `pull_request` and `merge_group` events.

Changes to any release-sensitive path force full validation:

```text
tools/release/**
.release/**
Cargo.toml
Cargo.lock
**/Cargo.toml
Makefile.toml
.github/workflows/**
.github/actions/**
tools/cargo-miden/templates/**
binary packaging logic
the change-impact classifier
```

Full validation adds:

- Packaging the complete selected workspace closure.
- Static local-registry resolution/build checks using exact `.crate` files.
- All release-tool failure-injection and mock-protocol tests.
- All SemVer checks.
- The full six-artifact native runner matrix.
- Deterministic archive/checksum/manifest checks.
- A checked-in fixture inventory covering every supported manifest dependency
  form, release-unit edge, registry-index translation, recovery state, workflow
  phase, and plan-schema compatibility version.

Nightly/default-branch validation repeats the full package closure, all SemVer
checks, clean double-build diagnostics, and hosted no-production rehearsal.
This detects dependency, runner-image, toolchain, and external-service drift
that no pull-request diff can predict.

### 15.2 Package-closure verification

For every selected crate:

1. Run a pinned Cargo version with `cargo package --locked --no-verify`.
2. Inspect the normalized manifest and package file list.
3. Compute the `.crate` digest.
4. Seed a temporary static registry with the exact packages and index entries.
5. Resolve third-party dependencies from crates.io.
6. Build/check libraries and install/smoke-test binaries exclusively through
   registry packages rather than workspace paths.

`cargo publish --dry-run --locked` remains useful where the dependency closure
already exists on crates.io, but it is not the primary coordinated-release
test.

The local-index generator follows Cargo's registry schema, including renamed
dependencies and same-registry versus crates.io source translation. Schema
fixtures are checked independently of the planner so the package test does not
merely reproduce the planner's own metadata error.

### 15.3 Failure injection

Scripted registry and GitHub backends test:

- Immediate and delayed index visibility.
- Upload accepted followed by response loss or polling timeout.
- Existing expected and conflicting checksums.
- Existing yanked or otherwise non-resolvable versions; these are conflicts
  even when their stored checksum matches.
- Authentication expiration.
- Rate limiting and transient server errors.
- Permanent rejection within an SDK stage.
- Partial draft assets and conflicting release tags.
- Missing attestations.
- Concurrent executors.
- Interruption after every state transition.

Properties require that:

- Preflight failures make no irreversible calls.
- Reruns never publish a crate twice.
- Partial SDK failure cannot begin compiler publication.
- Conflicting state always stops.
- Finalization cannot precede every prerequisite.

### 15.4 Staging validation

The preferred hosted registry rehearsal is `staging.crates.io`.

An implementation spike must first verify:

- Account and authentication availability.
- Whether staging Trusted Publishers can mirror the intended workflow identity.
- Staging crate bootstrap/ownership requirements.
- Whether external dependencies are available or must be explicitly sourced
  from production crates.io.
- Acceptable version and cleanup conventions.
- Expected persistence and service availability.

If viable, staging uses a temporary checkout with unique prerelease versions,
for example `X.Y.Z-staging.<run-id>`, and rewrites all first-party dependency
requirements consistently. SemVer validation runs on the untransformed
candidate. The tool then derives and validates a rehearsal projection before
publishing. Staging exercises the real publication order, Cargo upload
behavior, registry responses, index polling, checksum reconciliation, and
consumer installation; it does not prove production `.crate` bytes or
production permissions.

The preferred staging rehearsal also creates disposable rehearsal-namespace
tags and GitHub draft releases, uploads and redownloads the planned artifacts,
verifies build attestations, and cleans up only still-draft objects. Rehearsal
tag names cannot match production tag rules or workflow triggers.

Staging is a manual or scheduled acceptance check, not a normal required PR
check, because it is an external non-production service without a documented
availability guarantee.

### 15.5 GitHub draft fallback

If staging is unavailable or unsuitable, the rehearsal:

- Uses the same production intent and rehearsal checks with registry
  publication disabled.
- Runs the complete unprivileged package closure.
- Builds and attests the six real artifacts on GitHub runners.
- Creates uniquely named draft releases.
- Uploads, redownloads, and verifies the complete asset set.
- Verifies build attestations.
- Leaves all crates unpublished.
- Deletes only still-draft rehearsal releases and their disposable tags.

This does not test immutable finalization. That limitation is accepted because
finalization occurs only after crate, artifact, and attestation verification
has succeeded and has little repository-specific logic.

### 15.6 Irreducible production-only behavior

Without a real production publish, the system cannot prove:

- Every real crate's individual production Trusted Publisher mapping.
- crates.io ownership and server-side acceptance for the exact package.
- Production rate limits and index/CDN propagation at release time.
- Recovery after a genuinely partial production publication.
- External settings have not changed between audit and execution.

The first production releases remain supervised integration milestones.

## 16. Release Processes

This section is the operational source of truth for the three supported flows.
Each step identifies whether it is performed by a maintainer or automation.

### 16.1 Production release

#### Phase A: Prepare the release candidate

1. **Maintainer:** Select `sdk`, `compiler`, `templates`, `sdk+compiler`, or
   `all`, and choose intended versions.
2. **Automation:** Create or update `.release/release.toml` with the exact unit
   scope, versions, expected tags, template track/digest, and release-note
   targets.
3. **Maintainer:** Run `set-workspace-version` and/or `set-crate-version`.
4. **Automation:** Update manifests, local requirements, and `Cargo.lock`; emit
   the affected package and dependent-crate report.
5. **Maintainer:** Review version, dependency, and committed scope changes.
6. **Maintainer:** Run `changelog-prompt` for selected units/crates, optionally
   pass the prompt to an agent, and edit/review the result.
7. **Automation:** Run local release lint, SemVer checks, candidate/intent
   planning, and changelog structural checks.
8. **Maintainer:** Open a release-candidate pull request containing
   `.release/release.toml` and all version, lockfile, changelog, template, and
   release-note changes.
9. **Automation:** Run the required release gate and full validation implied by
   release-sensitive changes.
10. **Maintainer/reviewer:** Review and merge the exact candidate scope only
    after all required checks pass.

No production tag, GitHub release, crates.io credential, or crate publication
is created in Phase A.

#### Phase B: Establish the release commit and plan

1. **Maintainer:** Create the primary protected tag(s) declared in
   `.release/release.toml` at the exact merged
   release commit:
   - Compiler: `vX.Y.Z`
   - SDK: `sdk/vX.Y.Z`
   - Templates: `templates/vX.Y.Z`
2. **Maintainer:** Dispatch `release.yml` from protected `main` with one
   coordination subject tag: compiler when selected, otherwise SDK, otherwise
   templates.
3. **Automation:** Acquire the repository-wide production mutation lock with
   cancellation disabled.
4. **Automation:** Load the committed candidate declaration from the subject,
   derive unit scope only from it, and validate that every declared primary tag
   points to the subject commit and matches manifest/template versions.
5. **Automation:** Record the subject and executor commits, capture one external
   state snapshot, reject overlapping active plans, and re-run release lint,
   SemVer checks, package closure, template tests, and release-intent
   generation.
6. **Automation:** Present the intent digest, package stages, expected
   drafts/assets, executor/subject commits, and warnings.
7. **Maintainer:** Review the intent summary. Cancel before approval if scope,
   version, tag, or artifact information is unexpected.

#### Phase C: Prepare GitHub drafts and artifacts

1. **Automation:** Generate each selected crate package and build each template
   bundle and executable exactly once; smoke-test, archive, hash, and attest
   selected artifacts.
2. **Automation:** Seal the canonical release plan with all package and asset
   digests.
3. **Automation:** Create one draft GitHub release for every selected unit.
4. **Automation:** Upload the exact payload inventory, checksums, and write-once
   `release-plan-<digest>.json` to each draft; upload uniquely named journal
   segments only to the coordination draft.
5. **Automation:** Redownload drafts and verify names, sizes, SHA-256 digests,
   and build attestations.
6. **Automation:** Repeat package generation under the pinned Cargo/toolchain
   and require the expected byte-stability result before approval.
7. **Automation:** Append a journal segment and emit a final pre-publication
   summary.

Failure in Phase B or C is fully reversible: no crate has been published and
all GitHub releases remain drafts.

#### Phase D: Approve and publish crates

1. **Maintainer:** Approve the protected `release-production` environment
   after reviewing the plan and prepared drafts. A template-only release uses
   the same approval boundary but does not request a crates.io token.
2. **Automation:** Revalidate tag/commit identity, draft asset digests, plan
   digest, candidate-declaration digest, executor compatibility, and live
   registry state.
3. **Automation:** Obtain a temporary Trusted Publishing token for the SDK
   stage.
4. **Automation:** For each SDK dependency stage:
   - Query every expected crate/version.
   - Treat yanked or non-resolvable versions as conflicts.
   - Skip only a matching, resolvable existing checksum.
   - Publish absent versions one at a time with the pinned Cargo version and
     explicit `--no-verify --locked --registry crates-io`.
   - Poll until each version/checksum is visible.
   - Stop the stage on any conflicting or ambiguous state.
5. **Automation:** Verify the entire selected SDK publication.
6. **Automation:** If compiler is selected, obtain a fresh temporary token and
   publish compiler stages using the same reconciliation rules.
7. **Automation:** Verify the entire selected compiler publication.
8. **Automation:** If templates are selected with crates, test the final bundle
   against the now-published required versions. For template-only releases,
   test the configured matrix of compatible already-published versions.

Once the first crate has been published, rollback is impossible. A failure
means pause, diagnose, and resume from observed registry state.

#### Phase E: Finalize selected releases

1. **Automation:** Reverify every selected crate, template bundle, asset,
   checksum, and build attestation.
2. **Automation:** With a fresh `CARGO_HOME`, resolve/install representative
   consumers from crates.io, build generated templates, and run smoke tests
   against assets redownloaded from the still-draft releases.
3. **Automation:** Create protected per-crate SDK baseline tags
   `crate/<name>/vX.Y.Z` at the release commit for every successfully published
   changed crate. The tag permission path is preflighted before publication. If
   creation still fails, pause and require the maintainer to run the generated
   tag command before finalization.
4. **Automation:** Remove recovery journal segments from the coordination
   draft, upload the final self-excluding manifest, and confirm that every
   draft contains exactly the planned public assets and still targets the
   reviewed tag/commit.
5. **Automation:** Publish selected drafts in dependency order:
   - SDK
   - Templates, when selected
   - Compiler
   SDK/templates and compiler prereleases use `make_latest=false`; only stable
   compiler releases may become latest.
6. **Automation:** Verify immutable release attestations, release assets, and
   build provenance.
7. **Automation:** Optionally repeat consumer smoke checks as post-release
   monitoring; they are not the first consumer gate.
8. **Automation:** Emit the final release report and final journal through the
   workflow record without attempting to mutate immutable releases.
9. **Maintainer:** Review the report and handle announcements or downstream
   coordination outside the release tool.

### 16.2 Staging or draft rehearsal

This flow tests a candidate or release-tool change without publishing
production crates.

#### Preferred path: `staging.crates.io`

1. **Maintainer:** Dispatch `release-rehearsal.yml` for a reviewed candidate or
   a named checked-in full rehearsal profile. Freeform unit selection is not
   accepted.
2. **Automation:** Run SemVer/policy checks on the untransformed source.
3. **Automation:** Derive a rehearsal projection, assert that it preserves the
   package graph, stages, unit ordering, binary inputs, and template checks,
   then create an isolated temporary checkout with unique staging prerelease
   versions.
4. **Automation:** Rewrite first-party requirements consistently without
   modifying the source checkout.
5. **Automation:** Run the same package closure, template tests, and artifact
   matrix used by production.
6. **Automation:** Obtain staging credentials through the staging environment.
7. **Automation:** Publish SDK and compiler fixture/candidate packages to
   staging in the production dependency order.
8. **Automation:** Poll and reconcile staging registry state, including
   timeouts and resumptions.
9. **Automation:** Install/build representative consumers from staging.
10. **Automation:** Build and attest GitHub artifacts when compiler is selected.
11. **Automation:** Under a rehearsal-only tag namespace, create disposable
    GitHub drafts, upload/redownload assets, verify checksums and attestations,
    and delete only still-draft objects after success.
12. **Automation:** Produce a rehearsal report comparing the projection with
    the production intent and explicitly noting that versions, package bytes,
    permissions, and registry are not production-identical.
13. **Maintainer:** Review the report. Staging success is evidence for the
    implementation, not authorization for a production release.

No production tag or immutable release is created.

#### Fallback path: GitHub draft rehearsal

1. **Maintainer:** Select the draft fallback when staging is unavailable,
   unconfigured, or unsuitable.
2. **Automation:** Run the same rehearsal projection and checks with registry
   publication disabled.
3. **Automation:** Build, archive, hash, and attest selected artifacts.
4. **Automation:** Create disposable draft releases and upload the planned
   assets.
5. **Automation:** Redownload and verify all assets and attestations.
6. **Automation:** Produce a report explicitly marking registry upload/index,
   production package identity, and immutable finalization as untested.
7. **Automation:** Delete only disposable still-draft releases and tags after
   successful verification, or retain them temporarily for diagnosis after
   failure.
8. **Maintainer:** Review and acknowledge the rehearsal limitations.

### 16.3 Changes to the release tooling

This flow applies to changes under `tools/release`, `.release`,
`Makefile.toml`, release workflows/actions, package classification, template
packaging, or artifact packaging.

#### Phase A: Local development

1. **Implementer:** Update the design first when changing release invariants,
   security boundaries, unit structure, or irreversible-operation ordering.
2. **Implementer:** Add or update unit, property, fixture, snapshot, and
   protocol tests with the implementation. Plan-schema changes include backward
   compatibility or explicit migration fixtures for every supported active
   schema.
3. **Implementer:** Run release-tool format, Clippy, tests, release lint, plan
   determinism, and relevant real-workspace temporary-copy checks.
4. **Automation:** Validate the checked-in fixture inventory and required
   planner, manifest-form, registry-schema, workflow-phase, recovery-state, and
   plan-compatibility coverage. Coverage expectations are concrete inventory
   entries rather than an unenforceable claim that every code change maps to a
   fixture.

#### Phase B: Pull-request validation

1. **Implementer:** Open a pull request describing changed invariants,
   production-only deltas, new failure modes, and rehearsal evidence.
2. **Automation:** Force the complete release validation suite; release-tool
   changes may not use incremental path selection.
3. **Automation:** Execute the same reusable verification workflow proposed by
   the pull request with read-only permissions.
4. **Automation:** Run actionlint, zizmor, custom workflow-policy checks,
   package closure verification, all failure-injection tests, and the six-
   artifact matrix.
5. **Release CODEOWNER:** Review security permissions, phase ordering,
   production/rehearsal parity, and test coverage.

#### Phase C: Hosted acceptance

1. **Maintainer:** For a material executor, registry, or workflow change,
   trigger a staging rehearsal from the reviewed commit.
2. **Automation:** Run the preferred `staging.crates.io` flow.
3. **Maintainer:** If staging is unavailable, authorize the GitHub draft
   fallback and record that staging behavior remains unverified.
4. **Automation:** Attach the rehearsal report to the pull request or workflow
   run.
5. **Maintainer/reviewer:** Merge only after required CI and the applicable
   hosted acceptance check succeed or an explicit staging-availability waiver
   is recorded.

#### Phase D: Post-merge readiness

1. **Automation:** Run the complete default-branch/nightly release validation.
2. **Maintainer:** After changes to the production workflow identity,
   environment, or Trusted Publishing integration, run the protected auth-only
   production check.
3. **Automation:** Obtain and revoke the production temporary token without
   invoking `cargo publish`.
4. **Maintainer:** Supervise the next real production release; release-tool
   changes are not considered proven against production until that release
   completes.

## 17. Failure Handling

### 17.1 Before first crate publication

- Stop immediately.
- Keep or delete drafts according to diagnostic needs.
- Fix subject code/configuration in a new reviewed release candidate and
  generate a new plan if any planned source or byte changes.
- For an executor-only defect, fix protected `main` and resume the unchanged
  subject/plan only after plan-schema compatibility tests pass.

### 17.2 After partial crate publication

- Do not move or delete release tags.
- Do not delete draft releases containing the recovery plan.
- Fix an executor defect on protected `main`; do not move the subject tag.
- Resume only with an executor that declares and proves compatibility with the
  sealed plan schema.
- Do not republish an existing version blindly.
- Query crates.io for every planned package/version/checksum.
- Resume only missing packages whose prerequisites are verified.
- Treat a conflicting checksum as an incident requiring maintainer action.
- Keep all GitHub releases as drafts until the selected unit is complete.

### 17.3 After immutable finalization

- Never attempt to replace an asset or reuse the tag.
- Release a corrected version.
- Document the superseded release and consider yanking affected crate versions
  only through an explicit maintainer decision.

## 18. Implementation Milestones

### Milestone 1: Inventory and policy

Acceptance criteria:

- Every workspace package is explicitly classified.
- Workspace membership is explicit and `cargo metadata` is stable.
- Publishability and dependency closure lint cleanly.
- The release configuration schema is documented and validated.

### Milestone 2: Planner, version, and changelog tooling

Acceptance criteria:

- Canonical intents are byte-identical for identical captured inputs, and
  sealed plans are byte-identical when supplied the same prepared artifacts.
- Committed candidate declarations unambiguously determine production scope.
- Subject and executor commits are modeled and validated separately.
- Version/property/fixture tests pass.
- Real-workspace temporary-copy tests change only expected manifests and lockfile.
- SemVer and changelog prompt tasks meet their specified interfaces.

### Milestone 3: Package closure

Acceptance criteria:

- Every selected crate produces an inspected `.crate`.
- The temporary static registry builds the full prepared closure.
- No package depends on an unpublished version outside the plan.
- Package digests and normalized metadata are recorded.
- Registry-index translation fixtures cover renamed and cross-registry
  dependencies, and yanked/non-resolvable states are rejected.

### Milestone 4: Executor and recovery

Acceptance criteria:

- Scripted registry/GitHub failure matrices pass.
- Random interruption/resume tests never duplicate publication.
- SDK/compiler ordering and finalization invariants hold under every tested
  failure.
- Journal reconciliation handles matching, missing, and conflicting state.
- Write-once plans, append-only journal segments, and final manifests obey
  their control-asset lifecycle.
- A reviewed newer executor successfully resumes a compatible older sealed
  plan in tests.
- Concurrent or overlapping production plans cannot mutate external state.

### Milestone 5: Binary and template artifacts

Acceptance criteria:

- All six artifacts build on native target runners.
- Architecture and smoke tests pass.
- Archives, checksum files, release manifests, and template bundles are
  deterministic.
- Missing, extra, and corrupted asset tests fail as intended.
- Template identity coupling, compatibility matrices, and safe-extraction tests
  pass.

### Milestone 6: PR workflow and security

Acceptance criteria:

- Pull requests execute the shared reusable verification workflow.
- The always-running release gate is required.
- actionlint, zizmor, workflow policy, CODEOWNERS, and SHA-pin rules pass.
- PR jobs have no production mutation or credential path.

### Milestone 7: Hosted rehearsal

Acceptance criteria:

- The staging feasibility spike has a documented outcome.
- If viable, a complete staging publication and consumer test succeeds.
- The preferred staging flow and GitHub draft fallback both build, attest,
  upload, redownload, and verify all planned GitHub artifacts.
- The rehearsal projection proves graph/stage/input parity with its production
  intent.
- Rehearsal limitations are explicit in the report.

### Milestone 8: Production readiness

Acceptance criteria:

- Trusted Publisher mappings are manually verified for every selected crate.
- GitHub settings/rules/environment audit passes.
- The protected auth-only check succeeds.
- Existing SDK baseline tags are seeded.
- Legacy publication triggers are disabled and the long-lived crates.io token
  is removed before the first pilot.
- A real templates-only immutable release succeeds before the first crate
  release when scheduling permits.

### Milestone 9: Supervised production adoption

Acceptance criteria:

- A supervised SDK release succeeds.
- A supervised compiler release succeeds.
- A combined SDK/compiler path succeeds.
- `release-plz`, the legacy workflows, and obsolete documentation are removed.
- The external template repositories are archived/read-only after compatible
  resolver adoption.

## 19. Decisions and Open Questions

### 19.1 Decisions

- The release tool is compiler-repository-specific.
- The repository has three release units: compiler, SDK, and templates.
- SDK crates publish before compiler crates in combined releases.
- Production uses Trusted Publishing and Trusted-Publishing-only mode.
- GitHub drafts are fully populated and verified before immutable publication.
- The planner/executor split and canonical plan are required.
- Production scope is committed and reviewed rather than selected by workflow
  inputs.
- Release-subject source is distinct from the protected-main executor so a
  compatible newer executor can resume an older sealed plan.
- Production mutation is serialized globally with cancellation disabled.
- Sealed plans, journal segments, and final public manifests have distinct
  lifecycles.
- PRs run the same substantive reusable verification workflow as production.
- `staging.crates.io` is the preferred registry rehearsal.
- Preferred staging rehearsal also exercises disposable GitHub drafts; the same
  draft flow without registry publication is the fallback.
- Immutable finalization is not duplicated in the fallback rehearsal.
- LLM changelog generation remains a manual, reviewed preparation activity.

### 19.2 Open questions for implementation

- Is `staging.crates.io` sufficiently available and compatible for the full
  compiler/SDK dependency graph?
- Should public package manifests explicitly allow a named staging registry, or
  should staging permissions be introduced only in a temporary checkout?
- Which currently publishable internal packages should become private?
- What exact SDK crate should act as the aggregate release anchor if `miden`
  becomes unsuitable?
- How many prior plan-schema versions must a current executor support before an
  explicit migration is required?
- What degree of binary bit reproducibility is achievable on the selected macOS
  and Linux runners?
- Which tool-specific artifact smoke operations provide meaningful coverage
  without making release preparation excessively slow?

These questions must be resolved before the corresponding implementation
milestone is considered complete.

## 20. References

- [Cargo publish](https://doc.rust-lang.org/cargo/commands/cargo-publish.html)
- [Cargo package](https://doc.rust-lang.org/cargo/commands/cargo-package.html)
- [Cargo registries](https://doc.rust-lang.org/cargo/reference/registries.html)
- [Running a Cargo registry](https://doc.rust-lang.org/cargo/reference/running-a-registry.html)
- [Cargo registry web API](https://doc.rust-lang.org/cargo/reference/registry-web-api.html)
- [crates.io Trusted Publishing](https://crates.io/docs/trusted-publishing)
- [crates.io authentication Action](https://github.com/rust-lang/crates-io-auth-action)
- [GitHub reusable workflows](https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows)
- [GitHub artifact attestations](https://docs.github.com/en/actions/how-tos/secure-your-work/use-artifact-attestations/use-artifact-attestations)
- [GitHub immutable releases](https://docs.github.com/en/code-security/concepts/supply-chain-security/immutable-releases)
- [GitHub environments](https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments)
- [GitHub secure-use guidance](https://docs.github.com/en/actions/reference/security/secure-use)
- [`cargo-semver-checks`](https://github.com/obi1kenobi/cargo-semver-checks)
- [`actionlint`](https://github.com/rhysd/actionlint)
- [`zizmor`](https://docs.zizmor.sh/)

---

## Schema v2: configurable release units

Recorded after the fact. The sections above describe the design as it was first
built, with `compiler`, `sdk`, and `templates` as a closed Rust enum. They are
left as written; this section records what changed and why.

The motivation was not this repository. It was that the same five-phase process —
reversible work first, one approval gate, irreversible work after — is worth
having in other Cargo repositories, and a tool that names its units in its own
source code cannot be given to one. Everything below follows from making the
units data instead of code. `.release/config.toml` gains `schema-version = 2`,
and the production code no longer contains a single hardcoded unit name.

### Two axes, not one

The original enum conflated two questions: *does this unit publish crates* and
*does this unit have a tag, a changelog, and a GitHub release*. For the three
units this repository has, they never came apart — `compiler` and `sdk` answered
yes/yes, `templates` answered no/yes, and private crates were a separate concept
entirely. Treating them as one axis was invisible here and wrong in general.

Splitting them gives four kinds:

| kind | publishes crates | is released |
| --- | --- | --- |
| `crates` | yes | yes |
| `library` | yes | no |
| `artifact` | no | yes |
| `private` | no | no |

`publishes_crates()` and `is_releasable()` are the two predicates, and every
policy decision in the tool now asks one of them rather than matching on a unit
name. That is what removed the names: publish ordering, changelog paths, tag
templates, version domains, asset routing, and lint's own checks were all
name-driven and are now kind-driven and configuration-driven.

### `library`: the cell that had no name

The fourth kind is the one this repository does not use, and the one that
justifies the split. A crate that several releasable units depend on, that must
be on the registry for any of them to resolve, and that nobody wants to tag has
no home under a one-axis model:

- `private` is wrong: consumers could not resolve it.
- A `crates` unit of its own is wrong: it acquires a tag namespace, a changelog
  nobody writes, and a third version to argue about.
- Folding it into one of the units that uses it is wrong: the other units then
  depend on a crate owned by a unit they are not released with, and
  `verify-closure`'s self-containment check reports the scope as not
  self-contained — correctly.

`library` is exactly that cell. Its crates join the publish stage of every
releasable unit that depends on them, transitively, deduplicated across stages so
a shared crate is claimed by the first stage rather than named twice in an intent
a human has to review. `intent::release_scope` walks the same closure, because
"what will this release publish" and "what does each stage publish" must agree.

A `library` unit may share the workspace version domain, since it is never
released on its own and simply rides the version of whatever publishes it.

### Three relations, and two of them point in opposite directions

| Relation | Declared on | Means |
| --- | --- | --- |
| `after` | the unit that waits | publish order |
| `release-when` | the unit that gets dragged in | explicit co-release |
| `tracks` | the tracking unit | this unit's sources embed a requirement on another unit's packages |

`after` and `release-when` are deliberately declared from opposite ends.
Ordering is a property of the unit that must wait, so it names what it waits for;
co-release is a property of the unit that gets dragged in, so it names what drags
it. Declaring both from the same end would read more uniformly and would put the
knowledge in the wrong place — a unit does not know who depends on it, and a unit
that must be released alongside another does know why.

`tracks` implies `after`, because a unit embedding a requirement on another
cannot resolve until that other unit is on the registry. `after_all()` is
`after` plus the tracked units, and it is what the topological sort walks.

#### Co-release is decided from content, not structure

`tracks` deliberately does not imply co-release. The requirement `set-version`
writes is `major.minor` for a stable version and the exact version for a
prerelease, so the tracked unit moving does not necessarily change anything:

| tracked unit moves | requirement | tracking unit's release |
| --- | --- | --- |
| `0.14.0` → `0.14.1` | `"0.14"` → `"0.14"` | not forced |
| `0.14.0` → `0.15.0` | `"0.14"` → `"0.15"` | forced |
| `0.14.0` → `0.15.0-rc.1` | `"0.14"` → `"0.15.0-rc.1"` | forced |

A structural rule — "tracking implies co-release" — forces a release on every
patch bump of the tracked unit, which means a version bump with nothing to
describe and a changelog section that says nothing. `candidate::validate` instead
reads the requirement the tracking unit's manifest currently declares, computes
what releasing the tracked unit's declared version would require, and demands
co-release only when the two differ. Structure decides *ordering*; content
decides *scope*.

`release-when` remains for co-release that has no version-requirement basis at
all. This repository declares none.

### `Stage.latest` is sealed, not derived

Which release claims the repository's "latest" slot was previously decided at
finalization, by asking whether the unit was the compiler and the version was
stable. With units in configuration, deriving it at finalization would mean
reading `.release/config.toml` after the approval gate — and configuration can
move between the review and the publication.

`latest` is therefore computed at intent time, from `unit.latest && !prerelease`,
and carried through `seal` into the plan. `finalize` acts on the sealed boolean
and reads no configuration. The reviewed plan is the authority, which is the same
principle that makes the intent deterministic in the first place.

Both schemas go to 2: `intent::SCHEMA_VERSION` and the plan's, since a plan
embeds an intent. In-flight intents and plans do not survive the change. That is
acceptable because Phase B has no side effects — an in-flight release is
re-planned from its candidate, which is cheap and reversible.

At most one unit may declare `latest`, checked at load.

### Asset routing is glob-based, and every ambiguity is an error

Staging previously routed by unit name. It now routes by each unit's `assets`
globs, matched against the file name — `--artifacts` is a flat directory, so
there are no path separators to reason about and a hand-written `*`/`?` matcher
is in proportion to the tool's dependency list.

Three ways routing can be ambiguous, all errors rather than defaults:

- **Unmatched.** A file matching no unit's globs. Defaulting it anywhere is
  guessing; dropping it silently loses an asset from a release nobody can add it
  to afterwards.
- **Ambiguous.** A file matching more than one unit. Globs must not overlap.
- **Out of plan.** A file matching a unit that this release does not include. It
  would be uploaded nowhere. The error names both remedies: stop building it for
  this release, or add the unit to the candidate.

`required-assets` is the fourth case, in the other direction: a glob that must
match at least one staged file when its unit is in the plan, so a matrix job that
did not run fails the release rather than producing a release missing a binary.

### Two releasable units may not share the workspace version domain

`version-source` is `"workspace"` or `"own"`, and it names a domain rather than a
per-unit version: `set-version --unit X` moves every package in every unit naming
X's domain. Two *releasable* units both naming `"workspace"` is refused at load,
because bumping one would silently move the other's crates to a version that is
never published — and two releasable units that share a version domain are one
unit. The message says so, and offers the two real fixes: give one `"own"`, or
merge them. A `library` unit is exempt, for the reason given above.

The corollary is a known limit: with only two domain names, a repository with
three releasable crate units cannot give each an independent version, and two
releasable units both naming `"own"` are not currently refused the way two naming
`"workspace"` are. A repository that needs three independent domains needs
`version-source` to become a named domain rather than a two-valued enum.

### Validation is front-loaded, deliberately

Every command loads the configuration, and loading validates all of it: kind/field
applicability, per-kind required fields, artifact source shape, package
classification, relation targets, tracked-package membership, and acyclicity of
the publish order. A unit naming a peer that does not exist, or a cycle in
`after`, is a mistake that would otherwise surface phases later — possibly after
something irreversible has happened. The cost is a few milliseconds on every
invocation; the alternative is discovering a typo after the approval gate.
