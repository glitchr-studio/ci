# glitchr-studio/ci

The shared CI of the omni families' PHP packages (omniscient/omnibase, omnibridge/*, omniauth/*, omnisearch/*, glitchr/typesense-bundle).

- `image/php-8.4/Dockerfile`: **ghcr.io/glitchr-studio/php:8.4**, PHP 8.4 with Composer, PHPStan and every extension the suites need (intl, zip, gd, bcmath, ftp, xsl, exif, imagick, igbinary, git2; pdo_sqlite, curl, sodium from the official image). Built and published by `.github/workflows/image.yml` on a change, by hand and monthly. Tags: `8.4` and `8.4-<commit>`.
- `.github/workflows/php-package.yml`: the reusable workflow a package calls from its own `.github/workflows/ci.yml`. One job in that image: the package installed alone (no site, no `--ignore-platform-reqs`, dependencies resolved afresh), `composer validate`, PHPUnit, PHPStan against the package's `phpstan-baseline.neon` (new violations fail; without a baseline the run uploads one as the artifact `phpstan-baseline`), then the package installed into an empty project without its require-dev and with no Composer plugin allowed, and a class autoloaded from there.

```yaml
name: ci
on:
  push:
  pull_request:
  workflow_dispatch:
jobs:
  package:
    uses: glitchr-studio/ci/.github/workflows/php-package.yml@main
    with:
      smoke-class: Omnibridge\Runtime
      # typesense-image: typesense/typesense:30.0.rca41   # a Typesense server beside the job
      # phpunit-args: --testsuite unit
```

- `.github/workflows/php-release.yml`: the release, on a pushed tag `vX.Y.Z`, after `php-package.yml` passed on that tag: the tag checked as a version, the notes taken from the package's `CHANGELOG.md` section for it (a missing section fails), the tag's source archive with its SHA-256 and a build provenance attestation, the GitHub release (pre-release for a suffixed tag or a 0.x version), and Packagist's `update-package` when the repository holds `PACKAGIST_USERNAME`/`PACKAGIST_TOKEN` (otherwise Packagist's GitHub integration or the maintainer's call picks the tag up; the packages carry no Packagist webhook of their own).

```yaml
name: release
on:
  push:
    tags: ['v*']
jobs:
  verify:
    uses: glitchr-studio/ci/.github/workflows/php-package.yml@<sha>
    with:
      smoke-class: Omnimap\MapLibre\MapLibreProvider
  release:
    needs: verify
    permissions: {contents: write, id-token: write, attestations: write}
    uses: glitchr-studio/ci/.github/workflows/php-release.yml@<sha>
    secrets: inherit
```

Callers pin both workflows to a commit of this repository (the provider template does); `@main` follows every change.

What it does not do: a version matrix. A package declares broad constraints and is tested on what they resolve to today, on PHP 8.4; the application's lock decides what runs.
