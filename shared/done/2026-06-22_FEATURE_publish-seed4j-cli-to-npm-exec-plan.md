# Publish seed4j-cli to npm

This ExecPlan is a living document. Keep `Progress`, `Decisions`, `Risks`, and `Lessons Learned` up to date as work advances.

Safety boundary: This task is limited to authorized, defensive maintenance of this repository. Do not provide offensive guidance or policy-bypassing instructions.

## Purpose / Big Picture

Publish `seed4j-cli` as an npm package so users and automation can install the existing CLI with `npm install -g seed4j-cli` while still running the Spring Boot JAR through Java. The installed command remains `seed4j`, and the first npm distribution is a Node.js wrapper around the Maven-built executable JAR. Users can observe the result by installing or packing the npm package and running `seed4j --help` or `seed4j --version`.

## Scope

In scope: npm metadata for public publishing, a Node wrapper command, a package preparation script that copies the built JAR into `dist/`, Node behavior tests for wrapper behavior, release automation for tag-based npm publishing, GitHub Release publication from a Release Drafter draft, release-note label guidance, and README documentation for npm install and Java 25 requirements.

Out of scope: bundling a JRE, using jDeploy, jlink, jpackage, GraalVM native images, changing CLI Java behavior, changing Maven coordinates beyond release-version automation, or adding MCP-specific publishing behavior.

## Definitions

`seed4j-cli` is the npm package name. It is intentionally unscoped.

`seed4j` is the executable command exposed by the npm package through `package.json` `bin`.

`dist/seed4j-cli.jar` is the JAR included in the npm package. It is copied from Maven `target/` by the package preparation script and is not source-controlled.

Trusted Publishing is npm's OIDC-based publishing flow from GitHub Actions. The workflow must use `npm publish --provenance` and package maintainers must configure the matching npm package publisher in npm before the first trusted publish can work.

Release Drafter is the GitHub Action that keeps a draft GitHub Release up to date on `main` from merged pull requests and their labels. The tag release workflow publishes that draft after npm publishing succeeds.

## Existing Context

The repository currently has a Java/Spring Boot CLI packaged by Maven and a Node toolchain used for formatting and development tasks. `package.json` is private, has version `0.0.0`, and does not expose a `bin` command. `.gitignore` ignores `/bin/`, which conflicts with committing an npm wrapper under `bin/`. README currently documents manual installation by copying the Maven-built JAR and a shell script into `/usr/local/bin`.

## Desired End State

`package.json` is ready for public npm publication as unscoped `seed4j-cli`, with `bin.seed4j` pointing to `./bin/seed4j.js`, package `files` restricted to publishable assets, and scripts for wrapper tests and package preparation. `bin/seed4j.js` launches `java -jar dist/seed4j-cli.jar` with forwarded arguments, preserves the Java process exit code or signal-derived failure, and prints a clear Java 25 requirement message when Java is missing. Release automation builds the JAR, prepares the npm package, smoke-tests the packed artifact, publishes on `v*` tags using npm Trusted Publishing, publishes the matching GitHub Release from the maintained draft, and attaches the versioned JAR from `target/`.

## Milestones

### Milestone 1 - Establish the Living Plan

#### Goal

Create this ExecPlan before major code changes and record the accepted decisions from the prior issue discussion.

#### Changes

- [x] Create `/home/renanfranca/projects/seed4j-cli/_temporary/ai_agent/seed4j-cli-ai-context/shared/2026-06-22_FEATURE_publish-seed4j-cli-to-npm-exec-plan.md`.
- [x] Record package naming, wrapper strategy, Java 25 requirement, and validation strategy.

#### Validation

- [x] Command: `test -f /home/renanfranca/projects/seed4j-cli/_temporary/ai_agent/seed4j-cli-ai-context/shared/2026-06-22_FEATURE_publish-seed4j-cli-to-npm-exec-plan.md`
- [x] Expected result: file exists.

#### Acceptance Criteria

- [x] The plan is self-contained and can guide implementation from a fresh context.

### Milestone 2 - Specify npm Wrapper Behavior

#### Goal

Add behavior tests proving the npm-installed command launches the packaged JAR correctly.

#### Changes

- [x] Add Node test coverage under `test/npm/` for argument forwarding, exit-code propagation, Java-missing diagnostics, and JAR path resolution.
- [x] Add an npm script to run the wrapper behavior tests with Node's built-in test runner.

#### Validation

- [x] Command: `npm run test:npm-package`
- [x] Expected result before implementation: failed for the expected missing wrapper behavior.
- [x] Expected result after implementation: passed.

#### Acceptance Criteria

- [x] Tests observe the public wrapper command behavior without depending on private helper functions.

### Milestone 3 - Implement npm Package Runtime Files

#### Goal

Make the npm package installable and runnable while keeping Maven as the source of the executable JAR.

#### Changes

- [x] Update `package.json` and `package-lock.json` for public package metadata, `bin`, `files`, and scripts.
- [x] Add `bin/seed4j.js` as an executable Node wrapper.
- [x] Add `scripts/prepare-npm-package.js` to copy the Maven-built JAR to `dist/seed4j-cli.jar`.
- [x] Update `.gitignore` so committed npm wrapper and scripts are allowed while generated `dist/` remains ignored.

#### Validation

- [x] Command: `npm run test:npm-package`
- [x] Expected result: wrapper tests passed.
- [x] Command: `npm run package:prepare`
- [x] Expected result after Maven package exists: `dist/seed4j-cli.jar` was created.

#### Acceptance Criteria

- [x] `npm pack --dry-run` includes only intended publishable package files and the prepared JAR when present.

### Milestone 4 - Add Release Automation and Documentation

#### Goal

Document and automate the release path for npm publication.

#### Changes

- [x] Add `.github/workflows/release.yml` for `v*` tags: derive version, set npm and Maven versions, build/test, prepare package, smoke-test, and publish with `npm publish --provenance`.
- [x] Update README to show `npm install -g seed4j-cli`, `seed4j --help`, `npx --package seed4j-cli seed4j --help`, Java 25 runtime requirement, Trusted Publishing setup, and first-publish fallback notes.

#### Validation

- [x] Command: `npm run prettier:check`
- [x] Expected result: formatting passed.

#### Acceptance Criteria

- [x] Users can follow README instructions for npm installation, and maintainers can follow release notes for first publish and tag releases.

### Milestone 5 - Focused Verification

#### Goal

Run focused local validation without triggering the full `clean verify` gate automatically.

#### Changes

- [x] Run focused Node, formatting, Maven test, package, and npm pack checks.
- [x] Update this ExecPlan with validation outcomes, risks, and lessons.

#### Validation

- [x] Command: `npm run test:npm-package`
- [x] Command: `npm run prettier:check`
- [x] Command: `./mvnw test`
- [x] Command: `./mvnw --batch-mode -ntp clean package`
- [x] Command: `npm run package:prepare`
- [x] Command: `npm pack --dry-run`
- [x] Command: packed tarball smoke install into a temp prefix and run `seed4j --version`.

#### Acceptance Criteria

- [x] Focused checks pass; sandbox escalations for npm cache and Husky git config access are documented.
- [x] User is asked to run `./mvnw clean verify` as the final local validation gate.

### Milestone 6 - Publish GitHub Release from Draft

#### Goal

Keep release notes drafted on `main` and publish the matching GitHub Release only after the npm package is successfully published from a `v<semver>` tag.

#### Changes

- [x] Add `.github/release-drafter.yml` based on the `seed4j` changelog categories, excluding frontend client labels.
- [x] Add `.github/workflows/release-drafter.yml` to update the draft on `main` and manual dispatch for `seed4j/seed4j-cli`.
- [x] Update `.github/workflows/release.yml` so `contents` permission is `write`, `npm publish --provenance` remains before GitHub Release publication, Release Drafter publishes the release for `v${VERSION}`, and `target/seed4j-cli-${VERSION}.jar` is uploaded as a Java archive.
- [x] Update README release documentation with draft behavior, tag release order, labels, and attached JAR guidance.
- [x] Synchronize required GitHub labels while leaving existing labels intact and not adding frontend client labels.

#### Validation

- [x] Command: `npm run test:npm-package`
- [x] Command: `npm run prettier:check`
- [x] Command: `./mvnw --batch-mode -ntp clean package`
- [x] Command: `npm run package:prepare`
- [x] Command: `npm pack --json --dry-run`
- [x] Command: inspect `.github/workflows/release.yml`
- [x] Command: `gh label list --repo seed4j/seed4j-cli`

#### Acceptance Criteria

- [x] `main` maintains a draft GitHub Release using `seed4j`-style release notes without frontend categories.
- [x] A tag workflow publishes npm first, then publishes the GitHub Release and uploads the versioned JAR.
- [x] README tells maintainers how labels affect release categories and that npm remains the primary installation channel.

## Progress

- [x] Milestone 1 started
- [x] Milestone 1 completed
- [x] Milestone 2 started
- [x] Milestone 2 completed
- [x] Milestone 3 started
- [x] Milestone 3 completed
- [x] Milestone 4 started
- [x] Milestone 4 completed
- [x] Milestone 5 started
- [x] Milestone 5 completed
- [x] Milestone 6 started
- [x] Milestone 6 completed

## Decisions

- Decision: Set the first npm package version to `0.0.1`.
  Rationale: The user explicitly selected `0.0.1` as the initial release version; package metadata, lockfile, and README release example must agree.
  Date/Author: 2026-06-23 / Codex

- Decision: Publish the npm package as unscoped `seed4j-cli`.
  Rationale: The latest user decision supersedes older issue acceptance criteria that mentioned a scoped package.
  Date/Author: 2026-06-22 / Codex

- Decision: Keep the installed command name as `seed4j`.
  Rationale: Existing CLI documentation and user habits use `seed4j`, and npm `bin` can expose that command from the `seed4j-cli` package.
  Date/Author: 2026-06-22 / Codex

- Decision: Use a Node wrapper around the existing Spring Boot JAR and require Java 25.
  Rationale: This is the smallest publishable npm path and avoids runtime bundling choices that are explicitly out of scope.
  Date/Author: 2026-06-22 / Codex

- Decision: Publish the GitHub Release only after `npm publish --provenance` succeeds.
  Rationale: The GitHub Release should reflect an npm version that is actually available; attaching the versioned JAR is convenience, not the primary distribution channel.
  Date/Author: 2026-06-23 / Codex

- Decision: Use the `seed4j` Release Drafter category model without frontend client categories.
  Rationale: The CLI repository should share the parent project's release-note style while avoiding labels that do not apply to this package.
  Date/Author: 2026-06-23 / Codex

- Decision: Rename the remove category title from `Remove modules` to `Remove features` in `seed4j-cli`.
  Rationale: The CLI release notes should not imply that every removal is a Seed4J module removal.
  Date/Author: 2026-06-23 / Codex

## Risks and Mitigations

- Risk: `dist/seed4j-cli.jar` can be stale relative to source code.
  Mitigation: The release workflow runs Maven package before `npm run package:prepare`, and local docs instruct maintainers to rebuild before preparing the npm package.

- Risk: npm Trusted Publishing requires out-of-repository package configuration and may fail on the first release if not configured.
  Mitigation: README release notes will call out the npm package Trusted Publisher setup and a first-publish fallback.

- Risk: Release Drafter depends on PR labels for useful categorization.
  Mitigation: README documents the label dependency, and required labels are synchronized in GitHub before relying on the draft.

- Risk: Publishing a GitHub Release before npm succeeds would advertise a version that users cannot install through the primary channel.
  Mitigation: Keep Release Drafter publish and JAR upload steps after `npm publish --provenance` in `.github/workflows/release.yml`.

- Risk: The npm package is only a Java launcher, so users without Java 25 will see runtime failure.
  Mitigation: Wrapper tests cover a clear Java-missing message, and README documents Java 25 as a runtime prerequisite.

- Risk: `/bin/` is currently ignored.
  Mitigation: Update `.gitignore` to allow the committed wrapper while continuing to ignore generated binary output elsewhere.

## Validation Strategy

1. Add and run focused Node wrapper tests with `npm run test:npm-package`.
2. Run formatting with `npm run prettier:check` after file edits.
3. Run Java unit tests with `./mvnw test`.
4. Build the JAR with `./mvnw --batch-mode -ntp clean package`.
5. Prepare and inspect the npm package with `npm run package:prepare` and `npm pack --dry-run`.
6. Smoke-install the packed tarball into a temporary npm prefix and run `seed4j --version`.
7. Inspect GitHub workflow ordering and release asset path.
8. Check GitHub labels with `gh label list --repo seed4j/seed4j-cli`.
9. Ask the user to run `./mvnw clean verify` locally for the final full gate.

## Rollout and Recovery

Release by creating a `vX.Y.Z` tag. The release workflow derives `X.Y.Z`, sets npm and Maven versions, builds and tests the CLI, prepares the npm package, smoke-tests the packed tarball, publishes with provenance, publishes the matching GitHub Release from the draft, and uploads `target/seed4j-cli-X.Y.Z.jar`. If a release fails before npm publishing, fix the repository or npm Trusted Publisher settings and rerun the workflow for the tag. If npm succeeds but GitHub Release publication or asset upload fails, rerun the workflow after fixing the release automation or upload conflict. If a bad npm version is published, publish a corrected patch version and deprecate the bad version with an explanatory npm deprecation message.

## Lessons Learned

- Validation update on 2026-06-23: `npm run test:npm-package`, `npm run prettier:check`, and `npm pack --json --dry-run` passed with package version `0.0.1`.
- Validation passed: `npm run test:npm-package`, `npm run prettier:check`, `./mvnw test`, `./mvnw --batch-mode -ntp clean package`, `npm run package:prepare`, `npm pack --dry-run`, and a temp-prefix install of the packed tarball followed by `seed4j --version`.
- `npm pack --dry-run` needed elevated filesystem access in this sandbox because npm wanted to write to the user npm cache and Husky attempted git config access through the worktree indirection.
- The temp-prefix smoke test reported the local machine active runtime mode as extension because it used the current user Seed4J CLI configuration; this still validates the npm wrapper and packaged JAR path, while GitHub release runners should start with a clean home.
- The release workflow now edits the first Maven project version directly instead of resolving the Maven Versions Plugin implicitly.
- The repository's Node package existed only for development tooling before this work, so npm publication requires changing both metadata and ignored paths.
- Release Drafter validation on 2026-06-23: `npm run test:npm-package`, `npm run prettier:check`, `./mvnw --batch-mode -ntp clean package`, `npm run package:prepare`, and `npm pack --json --dry-run` passed. `npm pack --json --dry-run` required elevated filesystem access for the npm cache and Husky git config access in this sandbox.
- GitHub label validation on 2026-06-23: `gh label list --repo seed4j/seed4j-cli` showed all configured release-note labels are present. The user created the missing labels `server: spring boot` and `area: spam`; frontend labels are not required by `.github/release-drafter.yml`.
