# Changelog

Each release is a tag of this repository, and `v1` moves to the newest v1 release. Use `@v1` to get fixes as they
ship, or a release tag or its commit SHA to stay on one. Up to v1.1 the action was published as
`valystrum/sbom-action`.

## v1.2 (2026-10-29)

For Valystrum v0.21.0 and later.

- Published as `valystrum/valystrum-sbom`: change `uses: valystrum/sbom-action@v1` to
  `uses: valystrum/valystrum-sbom@v1`.
- New `complete` input: set it to `true` when the step's SBOM is the whole of the source, and the components it no
  longer lists are removed in Valystrum, with their CVE and licence findings resolved. New `removed` output, also
  counted in the job summary.
- Each CycloneDX generator is pinned to an exact release and checked before it runs: `@cyclonedx/cyclonedx-npm` 6.0.1
  and `cyclonedx-bom` 7.6.0 with their whole dependency trees locked by hash (in `locks/`, which has to stay beside
  `action.yml`), `cyclonedx-gomod` 1.12.0 (needs Go 1.26 or later on the runner) and `cyclonedx-maven-plugin` 2.9.3
  with `--strict-checksums`. In v1.1 they floated to their latest releases.
- Python projects work again: v1.1 passed the Python generator an option its current release refuses, so any Python
  manifest failed the step. The generator now runs in a virtual environment, so the runner needs `python3` with
  `venv` instead of `pipx`.

## v1.1 (2026-09-28)

- The stamped signature, `X-CS-Signature: t=<unix seconds>,v1=<HMAC-SHA256 over "<t>.<body>">`, signed again on
  every retry. Valystrum has refused the earlier `sha256=` form since 2026-09-28, so a release before v1.1 fails.
- Licensed under the Apache License 2.0.

## v1

The first release: a CycloneDX SBOM from npm, Python, Go and Maven projects, uploaded to your Valystrum software
inventory.
