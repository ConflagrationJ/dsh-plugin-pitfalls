English | [中文](DSH-PLUGIN-PITFALLS.zh-CN.md)

# DeepSeek Harness Plugin Development Pitfall Handbook

> Version `v0.1.2` · Applies to DSH Desktop `44.7.0` (profile `desktop`) · Unofficial document
> The official documentation always takes precedence: this document only fills in the places where the docs don't connect the dots but which really block you.

## 0. How to use it

- Skim §1–§2 before you start; follow §3–§5 when writing the Host/Client halves; follow §6 for the config UI; follow §7 for verification; follow §8 for release.
- Every entry has the format: **symptom / root cause / what to do / verification**.
- Appendix A at the end is "symptom → what to check first"; Appendix B is the minimum acceptance checklist.

## 1. Choose the form first

| What you want | Form |
|---|---|
| A new tool, a new service, an interceptor, a lifecycle | Host plugin |
| A set of installable/distributable plugins | bundle (`dsh.bundle.patch` in `package.json`) |
| A settings page, a plugin page, an interface inside the conversation | Client plugin + slot |
| Just a set of workflows/rules/prompts | skill (directory + `SKILL.md`) |
| "Default injection / default execution" behaviour | plugin + `agent/created`; don't pretend inside a skill |

**P1｜Choosing a slot after writing the code**

- Symptom: once the UI is written you find there is nowhere to put it, or once placed it fights the host layout.
- Root cause: DSH's UI is "slots are declared first, plugins register later"; the slot decides the available space, props, and theme context.
- What to do: first use `Slots.listSubTree` from `cordis_inspect_query` to choose the root, read its `catalog` (registration options, ownerProps, standardProps), and only then write the component.
- Verification: after registration, your id or key appears in `Slots.listSubTree`'s `occupants`.

## 2. Package structure and manifest

The minimum structure of an installable bundle:

```text
package.json      name/version/type/exports/files/dsh.bundle{dsh.client}
cordis.patch.yml  insert your Host row
index.js          Host half
client.js         Client half (optional)
locale/*.json     meta.title / meta.description
icon.svg          original or redistributable
```

**P2｜A missing file in `files` cannot be caught locally**

- Symptom: installing from a local directory works fine, but after packing into a tarball files/locale/icon are missing.
- Root cause: directory installation is a **link**, so `files` does not apply; only a tarball is filtered by `files`.
- What to do: list everything in `files`; unpack the tarball and check the file list before every release.
- Verification: after `npm pack`, inspect the files inside the package; or unpack the release asset and check.

**P3｜Forgetting locale meta or the icon**

- Symptom: the plugin page shows the package name/default image, or shows inconsistent values.
- Root cause: display metadata comes from the `meta` in the exported `locale/*.json` and from `./icon`; when missing, it falls back to the package name and the default image.
- What to do: write `meta.title` and `meta.description` in both languages; export the icon via `./icon`; size ≤ 256 KiB.
- Verification: after installation, check the title/description/icon on the plugin page.

**P4｜`private` and publishing**

- Symptom: an internal package name is accidentally published to the public registry, or the release asset lacks dependencies.
- What to do: keep the bundle `private: true` and distribute it via tarball/asset; decide separately if it must go to a registry.

## 3. Installation and dependencies (the most time-consuming part)

`plugin_manager` is the only correct entry point; do not hand-write the profile's `package.json` or `cordis.patch.yml`.

**P5｜Directory installation does not install your npm dependencies**

- Symptom: the Host row says `failed to import: Cannot find package '<dep>'`, even though `dependencies` is written in `package.json`.
- Root cause: local directory installation is `link:`; pnpm will not install dependencies for linked packages; the dependencies exist only in the profile's node_modules, not on the resolution chain of the real path being linked.
- What to do: pack the bundle into a tarball (`npm pack`) and use the tarball as the `install_bundle` target; install release assets from tarballs too.
- Verification: the install log shows `+ @scope/pkg x.y.z` (a real package, not `link:`); the plugin row is `fiberPhase: active`.

**P6｜Replacing an installed package: refreshing the page is not enough**

- Symptom: the code changed and the version number was bumped, but the page still shows the old UI; or the Host row's behaviour is unchanged.
- Root cause: replacing an installed package requires a restart to load the new generation of JS modules; the same holds for the Client half.
- What to do: change the package → `remove_bundle` → `install_bundle` with the new tarball → **restart DSH**.
- Verification: the schema changes in `Config.listConfigs`, the new registrations in the slot `occupants`, and the new description on the plugin page must all appear after the restart.

**P7｜remove/reinstall under a link installation carries real risk**

- Symptom: running remove/reinstall while a directory link installation is in place wiped the package directory in the workspace (reproduced once on this machine; the cause was not fully identified).
- What to do: switch to tarball installation early in the project; to change code, repack and install again; do not rely on delete/reinstall while in link state.
- Verification: before `remove_bundle`, confirm the workspace has a git commit or a copy.

**P8｜The prefix problem for entry ids and slot keys**

- Symptom: the config page/row config is registered but the interface does not appear.
- Root cause: the id from `Config.listConfigs` is `include:<patch id>`, while the id in the profile patch is `<patch id>`; different UI layers take different ids.
- What to do: use the patch id (e.g. `self-organizing-agent-team`) as the namespace for `settings.describe`/the config form; for the `plugins.row.config` key use `<package name>#<row id>`, with the row id as declared in the bundle patch; when unsure, register both keys:
  - `<pkg>#<patch id>`
  - `<pkg>#include:<patch id>`
- Verification: the corresponding key appears in `Slots.listSubTree`'s `occupants`.

## 4. The Host half

**P9｜A throw inside `apply` = the whole row is failed**

- Symptom: the plugin row is red failed, and the log shows only the import/throw.
- Root cause: `apply` (including `Config` parsing, reading files, registering services) runs at activation, and a throw means failure.
- What to do: `apply` should only register; put fallible I/O in `try` and give a readable error; put all registrations in `ctx.effect`/`ctx.on` and return disposers.
- Verification: `fiberPhase: active` in `plugin_manager list_plugins`.

**P10｜Editable Config fields must be `.volatile()`**

- Symptom: `Config` is declared, but the plugin page has no config form.
- Root cause: settings projects only **volatile fields** as editable forms; ordinary fields are merely part of the schema.
- What to do:

  ```js
  export const Config = z.object({
    autoLoad: z.boolean().default(true).volatile(),
  })
  ```

  At runtime it is a reference; read the value with `config.autoLoad.get()` (do not compare it directly as a boolean).
- Verification: `x-cordis.volatile: true` appears in the schema from `Config.listConfigs`; after changing the value, the row's override appears in the profile patch.

**P11｜"Default injection" uses `agent/created` + an agent-scoped section**

- What to do: `ctx.on('agent/created', ({ agent }) => agent.ctx.effect(() => agent.ctx.systemPrompt.section({ name, order, text }), 'label'))`.
- Notes:
  - registration happens on the agent ctx, so it is cleaned up automatically when the agent is destroyed;
  - resume also fires `agent/created`, and does not stack duplicates;
  - every new agent (including teammate/subagent) will carry it, so the cost must be counted in the context budget;
  - do not register when the switch is off; config changes take effect for "agents created afterwards".
- Do not modify messages inside `agent/request`, and do not re-inject large blocks of text on every request.

**P12｜`ctx.skills.register` conflicts with filesystem skills**

- Symptom: the plugin registered a skill, but the `skill` tool loads the old version from `~/.dsh/skills`.
- Root cause: the filesystem provider sits at the preset scope layer, which is nearer than the plugin's runtime registration at the global layer, so it wins on a name collision.
- What to do: keep only one source; when a bundle registers a skill, do not also install the "filesystem fallback"; state "pick one" clearly in the docs.
- Verification: where the `Base directory` returned by the `skill` tool points.

## 5. The Client half

**P13｜Client plugin format**

- It must be `window.__ModuleLoader__.load({ id: '<package name>', factory(require) {...} })`; `id` must equal the package name.
- `dsh.client` in `package.json`: `platform: 'web'`, `immediately`, `inject` (only for activation order, not for module imports).
- Do not `require` Harness Client packages such as `@deepseek-ai/dsh-client-ui-primitives`; copy the controls you need and switch to `--dsw-alias-*` theme tokens.

**P14｜Whether a slot registration carries `locale` decides whether the component has `t`**

- Symptom: the settings page is blank; the console shows errors such as `t is not a function`.
- Root cause: a component's `t` comes from the registration option `locale: '<namespace>'`; without it there is no `t`.
- What to do: after `ctx.locale.register(NS, { en, zh })`, pass `locale: NS` at registration; translation keys must exist on both sides.
- Verification: the page renders; the text follows when switching language.

**P15｜Assets and cleanup**

- Put all styles, listeners, and timers in `ctx.effect` and return a disposer; do not write to `document.body`, do not use iframes, and do not read other plugins' DOM.
- Verification: no residue after uninstall/refresh.

## 6. Config UI (the section with the most real lessons)

DSH's plugin page **does not render Config automatically**. There are three ways to register a config UI:

| slot | key | Purpose |
|---|---|---|
| `plugins.row.config` | `<package name>#<row id>` | The **Configure** page on the row |
| `plugins.bundle.config` | package name | The config area on the bundle page |
| `settings.section` / `settings.general.item` | custom id | A page/row in Settings |

**P16｜Declaring Config alone is useless**

- You must register one of the slots above; otherwise the plugin page has only the card and the row, with no form.

**P17｜Non-volatile fields do not appear in the form**

- See P10; this is the most common reason for "the form doesn't exist".

**P18｜Row key prefixes need compatibility**

- See P8; the docs say "the row id follows the patch declaration", but in practice the composition layer may produce an `include:` prefix, so registering both is the most robust.

**P19｜Reading and saving form values**

- Two sources:
  - the `form.state` passed in by the page owner + `form.mutate(operations, expectedRevision)`;
  - get the form yourself with `ctx.configForms.get(ns)` and hand it to the component as the hook of the `inject` face.
- The save path is the profile's `cordis.patch.yml`; after changing values, inspect the file, then create a new agent to verify the behaviour.
- Verifying only that "the switch can be clicked" is not enough: **turn it off once and on once, and check both persistence and actual behaviour in both directions**.

## 7. Verification and troubleshooting

When there is no browser control, use inspect for "structural-level verification":

```text
cordis_inspect_list
cordis_inspect_query(host, Tool, listTools)
cordis_inspect_query(host, Service, listService[service])
cordis_inspect_query(host, Config, listConfigs{name}|{entry})
cordis_inspect_query(client, Slots, listSubTree{root})
```

- `Slots.listSubTree`'s `occupants` is hard evidence that "registration succeeded"; `keyDomain`'s "already taken" shows whether a key is occupied.
- `Config.listConfigs`'s `status: schema` + `x-cordis.volatile` tells whether the form will be projected.
- Do not fabricate screenshots; without browser control, report "visuals not verified" (a requirement of the official `verification.md`).

**P20｜Restart vs refresh**

| Change | Required |
|---|---|
| Installing a new bundle | HMR may pick it up; restart if necessary |
| Replacing an installed package (Host/Client code changed) | **Restart** |
| Only changing config values | Usually takes effect immediately, but only affects agents created afterwards |
| Only changing docs/README | Not required |

**P24｜Team messages: a successful tool call is not delivery**

- Symptom: `send_message` returns `{sent:true}` and the teammate replies "what do you want". `list_agents` looks fine.
- Root cause: `send_message` reports success when the steer is accepted (target inbox), and nothing after that: no later receipt confirms a durable flush or that the model read it. `spawn_teammate` is different: it flushes and checks the initial prompt before the member becomes `active`, and a prompt-checkpoint failure raises `TEAM_PROVISIONING_CONFLICT` instead of creating a member without its body.
- What to do: do not use `list_agents` as delivery evidence (it only proves the tool exists). Use a nonce round-trip: spawn a probe teammate whose prompt embeds a random token, require it to write the token to a workspace file and echo it back, then verify both. To check `send_message`, repeat with a second token. Spawned names are permanent, so do this once per host build, not per task.
- Verification: the file content equals the token after trimming an optional BOM or surrounding whitespace, the closing message echoes it, and timestamps are after the probe started; a thrown `send_message` means the message was never accepted.

## 8. Release engineering

**P21｜Released archives are not reproducible**

- Symptom: the zip hashes produced locally and in CI differ.
- Root cause: ordering is affected by platform case sensitivity, inconsistent timestamps/permissions/`create_system`, text CRLF differences, and zlib version differences.
- What to do: use stored (no compression) for ZIP, fixed timestamps, `create_system=3`, fixed permissions, POSIX path ordering, and LF text; gzip tarballs are very hard to reproduce across zlib versions, so **take the `SHA256SUMS` shipped with the release as authoritative**.
- Verification: packing locally with two Python versions gives identical results; CI assets match local.

**P22｜Line endings make content hashes drift**

- Symptom: snapshot verification fails in CI but passes locally.
- Root cause: Windows checks out CRLF, Linux checks out LF.
- What to do: put `* text=auto eol=lf` in `.gitattributes`; recompute snapshot-type pins after LF normalisation.

**P23｜The release process needs a gate**

- Use `workflow_dispatch` + a `dry_run` input for GitHub Actions; put the actual release in the `release` environment with Required reviewers configured.
- The repository Actions settings need Write permission, otherwise `GITHUB_TOKEN` cannot create a Release.
- Release assets must include at least: the installable package, checksums, and release notes (compat pin, commit, verification record).
- Do not write PATs into the repository, scripts, or logs.

## 9. Coexistence with other plugins/skills

- One package name + one bundle patch; keep the row id stable, and do not use the row id as a version number.
- Keyed registrations in the same slot: a later registration replaces the earlier one; multiple bundles competing for the same key overwrite each other.
- When coexisting with skills, keep only one source (P12).
- Be careful when writing `engines.dsh`/`peerDependencies` for a DSH version: unmet requirements block installation; experimental packages may change schema across versions.
- `@deepseek-ai/dsh-experimental-agent-team` ships `lib/types/mailbox.js` (durable queue, delivered acknowledgement, recovery), but `TeamService` does not import it on DSH 44.7.0. Do not use that file to argue runtime delivery semantics (version pin: `0.2.1-rc.2`).

## 10. Documentation map

| Want to look up | Where to look |
|---|---|
| bundle manifest, display metadata, icon, installation observations | `cordis-plugin-development/references/host-plugin.md` |
| Client module format, slot, theme, locale | `references/ui-plugin.md` + `references/practices.md` |
| config page slot, row key, form.state/mutate | `@deepseek-ai/dsh-client-ui-plugin-manager` README |
| volatile, settings form projection | `@deepseek-ai/dsh-settings` README |
| skill registration, discovery roots, invocation strategy | `@deepseek-ai/dsh-skill` / `dsh-skill-filesystem` / `dsh-tool-skill` README |
| provider composition, patch syntax | `cordis-composition-reference` skill |
| boundaries of browserless verification | `references/verification.md` |
| the same operation for UI and agent tools | `references/user-actions.md` |

## Appendix A: Symptom → what to check first

| Symptom | Check first |
|---|---|
| No form/switch on the plugin page | Is `plugins.row.config`/`plugins.bundle.config` registered; are the fields `.volatile()` |
| Blank settings page | Does the slot registration include `locale`; is the component crashing in the console |
| Row failed to import | Were dependencies installed with the tarball; was it a link installation |
| Behaviour unchanged after install/change | Does replacing the package require a restart; is the filesystem skill installed as well |
| Config can be clicked but not saved | Are the fields volatile; is the profile `cordis.patch.yml` written |
| Skill loads an old version | The base directory returned by `skill`; is there a second source |
| Message sent but the teammate does nothing | Don't stop at `list_agents`; use the nonce round-trip probe (P24) |
| Asset hashes differ | Ordering, line endings, timestamps, `create_system`, zlib |

## Appendix B: Minimum acceptance checklist

1. `install_bundle` → `application: applied`, row `fiberPhase: active`.
2. Your slot occupant is visible in `Slots.listSubTree`.
3. The `Config.listConfigs` schema contains editable fields (and they are volatile).
4. Change the config → the profile patch has an override → new agent behaviour changes (verify once with it off and once with it on).
5. The skill directory / the source path returned by the skill tool is correct.
6. Release assets: `SHA256SUMS` all match; the tarball installs; the Release notes include compat/commit.

## Version record

- `v0.1.0` (2026-10-18): first version, based on DSH Desktop 44.7.0 and one real bundle development (`self-organizing-agent-team`).
- `v0.1.1` (2026-10-18): added the English edition (`DSH-PLUGIN-PITFALLS.en.md`) and bilingual READMEs; content is equivalent to v0.1.0.
- `v0.1.2` (2026-10-18): added P24 (Team message delivery semantics and the nonce round-trip probe), the matching Appendix A symptom row, and the §9 note that `mailbox.js` is not wired.
