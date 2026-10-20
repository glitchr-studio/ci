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

What it does not do: a version matrix. A package declares broad constraints and is tested on what they resolve to today, on PHP 8.4; the application's lock decides what runs.
