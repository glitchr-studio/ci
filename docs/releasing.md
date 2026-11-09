# Releasing a provider

How a package of the omni families (a provider, a core, a bridge) publishes a version with this repository's
workflows. The process is the same for every package: a tag starts it, and nothing is uploaded by hand.

## Once per package

1. `composer.json` declares no `version` (the tag decides) and names its development branch's series in
   `extra.branch-alias`:

   ```json
   "extra": {"branch-alias": {"dev-1.x": "1.x-dev"}}
   ```

   `"dev-5.x": "5.0.x-dev"` works too: the series is then 5.0, and only 5.0.* tags pass.
2. `CHANGELOG.md` keeps its coming changes under `## Unreleased` (or `## [Unreleased]`).
3. `.github/workflows/ci.yml` can be called (`on: workflow_call:` beside `push` and `pull_request`), and
   `.github/workflows/release.yml` runs it on a tag before the release:

   ```yaml
   name: release
   on:
     push:
       tags: ['v*']
   permissions:
     contents: read
   jobs:
     verify:
       uses: ./.github/workflows/ci.yml          # the package's own CI, on the tagged commit
       permissions: {contents: read, packages: read}
     release:
       needs: verify
       permissions: {contents: write, id-token: write, attestations: write}
       uses: glitchr-studio/ci/.github/workflows/php-release.yml@<commit of this repository>
       secrets: inherit
   ```

   A provider generated from omni-provider-template has both files already, its `verify` job calling
   `php-package.yml` directly.
4. The package is on Packagist, submitted once from its GitHub address. Packagist's GitHub integration sees each
   new tag by itself. Without that integration, the repository's secrets `PACKAGIST_USERNAME` and
   `PACKAGIST_TOKEN` (Settings → Secrets and variables → Actions) let the workflow call `update-package` itself.
   With neither, the maintainer calls `update-package` after the release.

## Each release

1. On the development branch, move the `## Unreleased` entries under `## [X.Y.Z] - YYYY-MM-DD`, push, and wait
   for a green CI.
2. Tag that commit and push the tag:

   ```sh
   git tag -a v1.2.0 -m 1.2.0
   git push origin v1.2.0
   ```

3. `verify` runs the package's CI on the tag: `composer validate`, the package installed alone, PHPUnit,
   PHPStan against its baseline, a consumer install (and, for omnisense/omnibase, its harness).
4. `release` checks, then publishes:
   - the tag is `vMAJOR.MINOR.PATCH` (a suffix such as `-rc.1` allowed), and `composer.json` names no other
     version;
   - the tagged commit is on a branch that `extra.branch-alias` names, and the version is in that alias's
     series: `v1.2.0` on `1.x` (`1.x-dev`) passes, `v2.0.0` on `1.x` fails, and so does `v5.1.0` on `5.x`
     (`5.0.x-dev`);
   - `CHANGELOG.md` has the version's section, which becomes the release notes. A pre-release without its
     own section takes the section of `X.Y.Z`, else of `X.Y`, else `Unreleased`. A final release needs its own
     section;
   - the source archive of the tag (`git archive`, so `export-ignore` applies), its `SHA256SUMS`, and a build
     provenance attestation, then the GitHub release with both files attached, marked pre-release for a
     suffixed tag or a 0.x version.
5. Packagist lists the version within minutes. To check the archive's provenance:

   ```sh
   gh attestation verify name-1.2.0.zip --repo <owner>/<name> --signer-repo glitchr-studio/ci
   ```

A published tag is never moved or deleted, and the workflow refuses to replace a release that exists. A mistake
is fixed with a new version.

## Release candidates

`v1.0.0-rc.1`, `-rc.2`... go through the same checks and are published as pre-releases. Composer installs them
only for a project whose `minimum-stability` allows `RC`, or that asks for one explicitly
(`composer require vendor/name:1.0.0-rc.1`). Their notes come from `Unreleased` until the final's section is
written.
