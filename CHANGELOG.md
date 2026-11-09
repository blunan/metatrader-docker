# Changelog

## [1.1.4](https://github.com/blunan/metatrader-docker/compare/v1.1.3...v1.1.4) (2026-11-09)


### Bug Fixes

* **deps:** update ghcr.io/linuxserver/baseimage-selkies:debiantrixie docker digest to d5e900e ([#32](https://github.com/blunan/metatrader-docker/issues/32)) ([c8648ef](https://github.com/blunan/metatrader-docker/commit/c8648efdb653b2974cc52aa9d72f79beb73da2c5))

## [1.1.3](https://github.com/blunan/metatrader-docker/compare/v1.1.2...v1.1.3) (2026-11-05)


### Bug Fixes

* **deps:** update ghcr.io/linuxserver/baseimage-selkies:debiantrixie docker digest to ad55311 ([#30](https://github.com/blunan/metatrader-docker/issues/30)) ([05b1dbe](https://github.com/blunan/metatrader-docker/commit/05b1dbe52db3133d886f6f3fc5b20d3eb35bdb76))

## [1.1.2](https://github.com/blunan/metatrader-docker/compare/v1.1.1...v1.1.2) (2026-11-02)

Updates WineHQ staging from 11.19 to 11.21 ([#26](https://github.com/blunan/metatrader-docker/pull/26)). Its image is listed in the [v1.1.2 release](https://github.com/blunan/metatrader-docker/releases/tag/v1.1.2).

## [1.1.1](https://github.com/blunan/metatrader-docker/compare/v1.1.0...v1.1.1) (2026-11-02)

Moves to a newer build of linuxserver.io's Selkies base image (`debiantrixie`, digest `7fa5ffe`) ([#24](https://github.com/blunan/metatrader-docker/pull/24)). Its image is listed in the [v1.1.1 release](https://github.com/blunan/metatrader-docker/releases/tag/v1.1.1).

## [1.1.0](https://github.com/blunan/metatrader-docker/compare/v1.0.0...v1.1.0) (2026-10-27)

Adds optional Python: with `MT_PYTHON=true`, the next start installs Windows Python 3.14 and MetaQuotes' `MetaTrader5` package (with `numpy`) into `/config`, after checking the installer's signature and the packages' hashes. It's off by default ([#21](https://github.com/blunan/metatrader-docker/pull/21)). Its image is listed in the [v1.1.0 release](https://github.com/blunan/metatrader-docker/releases/tag/v1.1.0).

## [1.0.0](https://github.com/blunan/metatrader-docker/releases/tag/v1.0.0) (2026-10-26)

The first image published as `ghcr.io/blunan/metatrader-docker` ([#19](https://github.com/blunan/metatrader-docker/pull/19)). Earlier builds were published as `ghcr.io/blunan/mt-browser` and then `ghcr.io/blunan/meta-trade-browser`, which get no more updates. Its image is listed in the [v1.0.0 release](https://github.com/blunan/metatrader-docker/releases/tag/v1.0.0).
