# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Documentation

- README: full section on **unread catch-up / wakeup sync** (triggers, config keys, throttling, lock, blog dedupe interaction) and **blog dedupe** options.
- README: clearer **flow**, **at a glance** table, friendlier section titles, and tighter copy (same technical content).

## [0.3.0] - 2026-04-27

### Changed (breaking for installs on 0.2.0)

- **Independent maintainer scope:** `npm` name **`@hdanyal-ts/paperclip-plugin-agentmail`**, GitHub **`hdanyal-ts/paperclip-plugin-agentmail`**, manifest / plugin id **`hdanyal-ts.paperclip-plugin-agentmail`**. **Reinstall** the plugin and re-register the AgentMail webhook after upgrade. No **employer org** in package metadata; fork as you need.

[0.3.0]: https://github.com/hdanyal-ts/paperclip-plugin-agentmail/releases/tag/v0.3.0

## [0.2.0] - 2026-04-27

### Changed (breaking for earlier installs)

- **Third-party distribution:** not a `@paperclipai/*` or upstream first-party product; not listed in monorepos **Bundled** plugin examples.
- Prior naming iteration used a sample org scope; superseded by **0.3.0** (personal maintainer `hdanyal-ts`).

[0.2.0]: https://github.com/hdanyal-ts/paperclip-plugin-agentmail/releases/tag/v0.2.0

## [0.1.0]

Prior internal iteration under `@paperclipai/plugin-agentmail-paperclip` / `paperclipai.agentmail-paperclip`.
