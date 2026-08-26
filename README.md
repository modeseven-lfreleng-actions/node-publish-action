<!--
# SPDX-License-Identifier: Apache-2.0
# SPDX-FileCopyrightText: 2026 The Linux Foundation
-->

# 🚀 Node.js Package Publish Action

<!-- prettier-ignore-start -->
<!-- markdownlint-disable-next-line MD013 -->
[![Linux Foundation](https://img.shields.io/badge/Linux-Foundation-blue)](https://linuxfoundation.org/) [![Source Code](https://img.shields.io/badge/GitHub-100000?logo=github&logoColor=white&color=blue)](https://github.com/lfreleng-actions/node-publish-action) [![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0) [![pre-commit.ci status badge]][pre-commit.ci results page] [![OpenSSF Scorecard](https://api.scorecard.dev/projects/github.com/lfreleng-actions/node-publish-action/badge)](https://scorecard.dev/viewer/?uri=github.com/lfreleng-actions/node-publish-action)
<!-- prettier-ignore-end -->

Stamps a version into `package.json` and publishes the package to an
npm registry, such as a Linux Foundation Nexus instance.

## node-publish-action

The action wraps the production-proven Linux Foundation publish flow:

```text
npm version <X> --no-git-tag-version && npm publish
```

It composes
[node-create-npmrc-action](https://github.com/lfreleng-actions/node-create-npmrc-action)
for registry authentication, keeping publish `run:` logic out of
calling workflows. Versions get stamped at publish time, matching the
merge-driven release model where committed metadata (such as
`version.properties`), not the committed `package.json`, provides the
version.

## Usage Example

<!-- markdownlint-disable MD046 -->

```yaml
steps:
  - name: "Publish npm package"
    id: publish
    uses: lfreleng-actions/node-publish-action@main
    with:
      publish_version: '1.2.3-SNAPSHOT'
      registry_url: 'https://nexus3.example.org/repository/npm.snapshot/'
      load_credential: 'true'
      vault_mapping_json: ${{ secrets.VAULT_MAPPING_JSON }}
      op_service_account_token: ${{ secrets.OP_SERVICE_ACCOUNT_TOKEN }}
```

<!-- markdownlint-enable MD046 -->

## Requirements

The action needs `jq`, `realpath` (GNU coreutils, including `-m`
support), `mktemp` and `tr` on the runner. OIDC trusted publishing
also needs `head` and GNU `sort` with `-V`, for the toolchain floor
comparison. The action checks that second set when `oidc` is
`'true'`, leaving Basic and token publishes untouched. GitHub-hosted
Ubuntu runners
include these tools; minimal self-hosted or non-Linux runners must
provide them. The action checks for them up front and fails with a
clear error naming any missing tool. It installs Node.js and npm via
the pinned `actions/setup-node` action, without dependency caching.
Real publishes need egress to the target registry; dry-run mode
publishes nothing, writes no credential, and withholds the OIDC
token endpoint from npm. That last point holds for every dry run,
including one requesting `provenance`: npm attaches no provenance to
a run that publishes nothing, so there is no identity to supply.

## Inputs

<!-- markdownlint-disable MD013 -->

| Name                     | Required | Default  | Description                                                                                  |
| ------------------------ | -------- | -------- | -------------------------------------------------------------------------------------------- |
| publish_version          | True     |          | Version to stamp and publish, such as `1.2.3` or `1.2.3-SNAPSHOT`                            |
| registry_url             | False    | `''`     | Target npm registry URL ending with `/`; required unless `dry_run` is `'true'`               |
| dry_run                  | False    | `false`  | Pack and verify without publishing (npm may still read the registry); skips credential setup |
| path_prefix              | False    | `.`      | Project directory; must resolve within the workspace                                         |
| node_version             | False    | `22`     | Node.js version to set up, such as `22`, `22.x` or `lts/*`                                   |
| node_version_file        | False    | `''`     | File containing the Node.js version, such as `.nvmrc`; overrides `node_version`              |
| tag                      | False    | `latest` | npm distribution tag                                                                         |
| access                   | False    | `''`     | Package access: `public`, `restricted` or unset                                              |
| provenance               | False    | `false`  | Generate registry-native provenance; needs registry and OIDC support                         |
| nexus_user               | False    | `''`     | Basic auth username (default: calling repository name); ignored by token/OIDC modes          |
| nexus_password           | False    | `''`     | Registry password for Basic auth; required for real publishes unless another mode            |
| auth_token               | False    | `''`     | Bearer token written as `_authToken`; for registries rejecting Basic auth                    |
| oidc                     | False    | `false`  | Publish via npm OIDC trusted publishing; stores no credential                                |
| load_credential          | False    | `false`  | Fetch the password from 1Password via credential-load-action                                 |
| vault_mapping_json       | False    | `''`     | JSON mapping repository owner to 1Password vault (with `load_credential`)                    |
| op_service_account_token | False    | `''`     | 1Password service account token (with `load_credential`)                                     |
| scope                    | False    | `''`     | npm scope for the auth entry (for example `@onap`)                                           |

<!-- markdownlint-enable MD013 -->

### Input Character Allowlists

The action checks free-text inputs against strict whole-string
character allowlists before use, rejecting shell metacharacters and
embedded newlines:

<!-- markdownlint-disable MD013 -->

| Input           | Allowed                | Structure                                                         |
| --------------- | ---------------------- | ----------------------------------------------------------------- |
| publish_version | `0-9 A-Za-z . -`       | All-digit `MAJOR.MINOR.PATCH` segments plus an optional `-`suffix |
| registry_url    | `A-Za-z 0-9 . : / _ -` | `https://` scheme, non-empty host, trailing `/`, no userinfo      |
| tag             | `A-Za-z 0-9 . _ -`     | Non-empty                                                         |
| node_version    | `A-Za-z 0-9 . * / _ -` | Follows setup-node version syntax                                 |

<!-- markdownlint-enable MD013 -->

The `nexus_user`, `scope` and credential inputs pass through to
`node-create-npmrc-action`, which applies its own validation.

## Outputs

<!-- markdownlint-disable MD013 -->

| Name              | Description                                        |
| ----------------- | -------------------------------------------------- |
| published_version | Version stamped into `package.json` and published  |
| package_name      | Package name from the publish metadata             |
| tarball_name      | Tarball filename from the publish metadata         |

<!-- markdownlint-enable MD013 -->

## Behaviour

1. **Check inputs**: tests every input against its allowlist; real
   publishes need `registry_url` and a credential source as well,
   failing fast with a clear error otherwise. With
   `load_credential: 'true'`, real publishes need non-empty
   `vault_mapping_json` and `op_service_account_token` values too
2. **Authenticate** (real publishes using Basic or token auth):
   `node-create-npmrc-action` writes an authenticated `.npmrc` into
   the project directory. OIDC publishes skip this step entirely:
   npm exchanges a GitHub OIDC token at publish time, so no
   credential is ever written to disk and none needs scrubbing
3. **Stamp**: `npm version <X> --no-git-tag-version
   --allow-same-version --ignore-scripts` updates `package.json`,
   with the result read back and verified. A `::notice::` names any
   `preversion`/`version`/`postversion` scripts the project defines,
   since `--ignore-scripts` means they do not run
4. **Publish**: `npm publish --json` with the configured tag, access
   and provenance flags; the action parses the JSON metadata and
   verifies the published version matches the request
5. **Verify** (real publishes): `npm view <name>@<version>` against
   the target registry confirms availability; an unreadable package
   downgrades to a warning (registries may restrict anonymous reads
   or index asynchronously), while a readable package with the wrong
   version fails the action

## Dry-Run Mode

With `dry_run: 'true'` the action runs `npm publish --dry-run`, which
packs the package and reports the metadata without publishing. npm
may still consult the registry while doing so. Dry-run mode skips the
`.npmrc`/credential steps entirely, so it needs no `registry_url` and
no credential inputs.

CI holds no registry credentials and completes no publish, so the
testing workflow uses dry-run mode wherever the path under test
allows it. Two cases cannot, because dry-run mode skips the step
they exercise: the missing-`id-token` test, which input
validation rejects before any registry interaction, and the token
pass-through test, which inspects the generated `.npmrc` and then
fails against a registry host that does not exist.

## Authentication Modes

Three mutually exclusive modes. Supplying more than one fails the
action rather than picking a winner, since the effective credential
would otherwise be ambiguous.

That exclusivity governs the *inputs*. One case can still leave npm
free to choose: a Basic or token publish requesting `provenance`
keeps the OIDC token endpoint available, because npm signs with the
workflow identity, and npm may then authenticate through a trusted
publisher if the registry has one registered for the workflow. See
[Provenance](#provenance). Every other combination clears the
endpoint, so npm cannot substitute a credential.

### Basic auth (Nexus)

Pass `nexus_password` directly, or set `load_credential: 'true'` with
`vault_mapping_json` and `op_service_account_token` to fetch it from
1Password. `node-create-npmrc-action` writes an `_auth` entry holding
`base64(username:password)`.

### Bearer token

Pass `auth_token`. Written as an `_authToken` entry. Registries that
reject Basic auth for publishing, notably `registry.npmjs.org`,
require this form.

`nexus_user` plays no part in token or OIDC publishing, since neither
carries a username. Setting it alongside either mode is inert rather
than an error — callers that compute it unconditionally, such as a
matrix publishing to both Nexus and npmjs.org, would otherwise have
to strip it per target. The action emits a `::notice::` naming it as
ignored.

### OIDC trusted publishing

Set `oidc: 'true'`. The action stores no credential anywhere: npm
exchanges a GitHub OIDC token for a short-lived publish token, and
attaches provenance itself where it supports doing so.

Provenance is not universal under OIDC. npm generates it for a
**public package published from a public repository**, and not
otherwise, so a private repository, or a package published with
`access: restricted`, still authenticates and publishes but arrives
without an attestation. Trusted publishing still buys you the
absence of a stored credential in that case, which is the larger
prize for most callers.

One further condition is easy to trip over: npm attaches provenance
automatically while the `provenance` setting keeps its **default**,
and not once anything sets it. An explicit `provenance=false` in
`.npmrc` or in `npm_config_provenance` makes it non-default, and the
attestation then never appears. The action detects that and emits a
`::notice::`, treating it as a legitimate opt-out rather than an
error.

`publishConfig.provenance` behaves differently, and the difference
catches people out: npm flattens `publishConfig` into the publish
options rather than into its configuration, so it leaves the setting
default and trusted publishing still attaches provenance to a public
package. The action notices that case too, and points at `.npmrc` as
the way to opt out. An explicit `true` from either source fails the
run instead, because npm refuses explicit provenance for a
restricted package.

The action preflights two of the requirements, failing before any
registry interaction:

- `id-token: write` on the calling job, detected via the token
  endpoint. With `workflow_call`, npm needs the grant on **both** the
  caller and the called workflow
- npm >= 11.5.2 **and** Node >= 22.14.0. Meeting the Node floor does
  not meet the npm one — Node 22 ships npm 10.x, and even Node 24.0.0
  ships npm 11.3.0. The simplest remedy is to raise the Node version
  this action selects until its bundled npm clears the floor; current
  Node 24 releases do. Raise it through `node_version`, or through
  the file named by `node_version_file` when you select Node that
  way, since that input overrides `node_version` entirely. The action
  reports this rather than upgrading npm behind your back

  To keep a Node version whose bundled npm is too old, upgrade npm
  against **that same** version before calling this action:

  ```yaml
  - uses: actions/setup-node@820762786026740c76f36085b0efc47a31fe5020 # v7.0.0
    with:
      node-version: '22.14.0'   # the version passed below
  - run: npm install -g npm@^11
  - uses: lfreleng-actions/node-publish-action@<sha>
    with:
      publish_version: '1.2.3'
      registry_url: 'https://registry.npmjs.org/'
      node_version: '22.14.0'
      oidc: 'true'
  ```

  Running `npm install -g npm@^11` without first selecting the same
  Node version has no effect: this action runs `setup-node` itself,
  which switches to that version's own bundled npm and strands the
  upgrade on the earlier installation

npm enforces the rest at publish time, and the action cannot check
them in advance:

- GitHub-hosted runners; npm does not support self-hosted runners
  for trusted publishing
- The package's `repository.url` must match the GitHub repository
- npm validates the **calling** workflow's filename, so register the
  trusted publisher against the caller (for example
  `gerrit-merge.yaml`), not the reusable workflow

Provenance is not generated for packages published from private
repositories, even when the package itself is public.

## Credential Handling

`node-create-npmrc-action` masks the credential material, writes the
`.npmrc` with restrictive permissions and registers a guaranteed
post-job step that scrubs the file again — including when later steps
fail. This action adds no duplicate cleanup logic.

OIDC skips that machinery entirely: the action writes nothing, so no
file needs scrubbing.

## Provenance

The `provenance` input passes `--provenance` to npm, generating
registry-native Sigstore provenance. This works against registries
with provenance support (npmjs.org) and requires an OIDC token
(`id-token: write` permission). Nexus has no provenance support, so
leave the input at `false` for Nexus targets; generate GitHub
artifact attestations for the packed tarball instead.

With `oidc: 'true'` the action rejects the flag rather than passing
it through: trusted publishing does not take `--provenance`, and npm
attaches provenance by itself where it supports it. Note the
qualification above — a restricted package, or one from a private
repository, gets no provenance either way, so the rejection reflects
an unsupported flag rather than a promise that an attestation
follows.

> [!NOTE]
> `provenance: 'true'` alongside Basic or token auth keeps the OIDC
> token endpoint available, because npm signs with the workflow
> identity. On a registry that also has a trusted publisher
> registered for this workflow, npm may then authenticate through
> that exchange rather than the credential you supplied. The publish
> still succeeds and still carries provenance, but the credential
> used is npm's choice, not this action's. Prefer `oidc: 'true'` for
> npmjs.org, which makes that the explicit intent and needs no
> stored credential at all.

## Path Constraints

Relative values for `path_prefix` resolve against `GITHUB_WORKSPACE`,
not the current working directory, so behaviour stays deterministic
when a calling workflow sets a custom working directory. The project
directory must resolve within `GITHUB_WORKSPACE` and contain a
`package.json`; the boundary check re-runs in every later step that
touches the path. Paths that escape the workspace fail the action.

## Notes

- The action performs no dependency caching, in line with the
  organisation's cache-poisoning stance
- `--allow-same-version` keeps re-stamping idempotent when the
  committed `package.json` version already matches the request
- Stamping uses `--ignore-scripts`, so `preversion`, `version` and
  `postversion` do not run. No release lane installs dependencies
  before stamping, so a hook invoking the test or build script could
  not succeed; suppressing them keeps the merge-driven and tag-driven
  lanes producing the same tree. The action emits a `::notice::`
  naming the skipped scripts, because a dependency-free `version`
  hook (writing the version into a source constant, say) would
  otherwise go missing from the published package with no signal
- Publishing runs `prepublishOnly`/`prepack`/`prepare` scripts when
  the project defines them; run builds beforehand (for example via
  [node-build-action](https://github.com/lfreleng-actions/node-build-action))
  so the packed content is complete

[pre-commit.ci results page]: https://results.pre-commit.ci/latest/github/lfreleng-actions/node-publish-action/main
[pre-commit.ci status badge]: https://results.pre-commit.ci/badge/github/lfreleng-actions/node-publish-action/main.svg
