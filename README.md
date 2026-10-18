# DSH Plugin Pitfalls

Unofficial, community-maintained **pitfall handbook for DeepSeek Harness (DSH) plugin development**.

**[中文手册：docs/DSH-PLUGIN-PITFALLS.zh-CN.md](docs/DSH-PLUGIN-PITFALLS.zh-CN.md)**

## What this is

A field-notes companion to the official DSH plugin documentation. It collects the failures that cost real time when building a DSH bundle end to end:

- packaging and installation (tarball vs linked directory, dependency resolution, restart semantics)
- Host/Client halves, slots, locale, theme tokens
- configuration UI: `plugins.row.config` / `plugins.bundle.config` / `settings.section`, `volatile()` fields, row-key prefixes
- verification with `cordis_inspect_query` when browser control is unavailable
- reproducible release assets, checksums, GitHub Actions with an approval gate
- coexistence with skills and other bundles

Each entry is written as **symptom → root cause → fix → verification**, so it can be used as a checklist instead of prose.

Scope: DSH Desktop `44.7.0`, profile `desktop`, 2026-10. DSH is moving quickly; treat this as a snapshot, and treat the official documentation as authoritative whenever they disagree.

## Why a separate repository

This handbook is intentionally independent from any single plugin. It should be reusable by other DSH plugin authors (and by other agents) regardless of what they build.

## Contributing

Add an entry only when you can state the symptom, the root cause you verified, and the check that proves the fix. Keep official behaviour and local workaround clearly separated.

## License

MIT. This is not an official DeepSeek or DSH document.
