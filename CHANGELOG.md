# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added

- Added a GitHub Pages deployment workflow that validates the project, builds the production site, and publishes `dist/` through the official Pages actions.

### Fixed

- Updated Markdown syntax manifest documentation references so `pnpm check:manifest` succeeds after the default demonstration posts are removed.
- Corrected the canonical site URL for the `0x5t4ckc47.github.io` deployment.
- Made the project and timeline resolver tests use explicit fixtures instead of assuming optional default content exists.
