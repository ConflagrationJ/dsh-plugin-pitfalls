[English](DSH-PLUGIN-PITFALLS.en.md) | 中文

# DeepSeek Harness 插件开发避坑手册

> 版本 `v0.1.2` · 适用 DSH Desktop `44.7.0`（profile `desktop`）· 非官方文档
> 官方文档永远优先：本文只补充“文档没有串起来、但会真正卡住”的地方。

## 0. 怎么用

- 动手前扫 §1–§2；写 Host/Client 时按 §3–§5；做配置 UI 按 §6；验证按 §7；发布按 §8。
- 每条格式：**现象 / 根因 / 做法 / 验证**。
- 文末附录 A 是“症状 → 先查什么”，附录 B 是最小验收清单。

## 1. 先选形态

| 你要的东西 | 形态 |
|---|---|
| 新工具、新服务、拦截器、生命周期 | Host 插件 |
| 可安装/可分发的一组插件 | bundle（`package.json` 的 `dsh.bundle.patch`） |
| 设置页、插件页、对话里的界面 | Client 插件 + slot |
| 只是一套流程/规范/提示词 | skill（目录 + `SKILL.md`） |
| 需要“默认注入/默认执行” | 插件 + `agent/created`，不要写 skill 里假装 |

**P1｜先写代码再选 slot**

- 现象：界面写完发现没地方放，或者放进去和宿主布局打架。
- 根因：DSH 的 UI 是“slot 先声明、插件后注册”，slot 决定了可用空间、props、主题上下文。
- 做法：先用 `cordis_inspect_query` 的 `Slots.listSubTree` 选定 root，读它的 `catalog`（registration options、ownerProps、standardProps），再写组件。
- 验证：注册后 `Slots.listSubTree` 的 `occupants` 里能看到你的 id 或 key。

## 2. 包结构与清单

一个可安装 bundle 的最小结构：

```text
package.json      name/version/type/exports/files/dsh.bundle{dsh.client}
cordis.patch.yml  insert 你的 Host 行
index.js          Host 半
client.js         Client 半（可选）
locale/*.json     meta.title / meta.description
icon.svg          原创或可再分发
```

**P2｜`files` 漏文件，本地测不出来**

- 现象：本地目录安装一切正常，打成 tarball 后缺文件/缺 locale/缺 icon。
- 根因：目录安装是 **link**，`files` 不生效；只有打 tarball 才按 `files` 过滤。
- 做法：`files` 列全；每轮发布前解压 tarball 检查文件列表。
- 验证：`npm pack` 后看包内文件；或解压发布资产核对。

**P3｜忘记 locale meta 或 icon**

- 现象：插件页显示包名/默认图，或显示不一致。
- 根因：显示元数据来自导出的 `locale/*.json` 的 `meta` 与 `./icon`；缺失时回退包名和默认图。
- 做法：`meta.title`、`meta.description` 两种语言都写；icon 用 `./icon` 导出；大小 ≤ 256 KiB。
- 验证：安装后在插件页看标题/描述/图标。

**P4｜`private` 与发布**

- 现象：误把内部包名发到公共 registry，或发布资产缺少依赖。
- 做法：bundle 保持 `private: true`，用 tarball/资产分发；需要上 registry 时再单独决定。

## 3. 安装与依赖（最花时间的一段）

`plugin_manager` 是唯一正确入口；不要手写 profile 的 `package.json` 或 `cordis.patch.yml`。

**P5｜目录安装不会安装你的 npm 依赖**

- 现象：Host 行 `failed to import: Cannot find package '<dep>'`，明明 `package.json` 里写了 `dependencies`。
- 根因：本地目录安装是 `link:`，pnpm 不会为 linked 包安装依赖；依赖只在 profile 的 node_modules 里，不在被 link 的真实路径的解析链上。
- 做法：把 bundle 打成 tarball（`npm pack`），用 tarball 作为 `install_bundle` 的 target；发布资产也用 tarball 安装。
- 验证：安装日志里出现 `+ @scope/pkg x.y.z`（真实包，不是 `link:`）；插件行 `fiberPhase: active`。

**P6｜替换已安装包，刷新页面不够**

- 现象：代码改了、版本号也改了，页面里还是旧 UI；或 Host 行行为没变。
- 根因：替换已安装包需要重启才会加载新的 JS 模块代际；Client 半也一样。
- 做法：改包 → `remove_bundle` → 用新 tarball `install_bundle` → **重启 DSH**。
- 验证：`Config.listConfigs` 的 schema 变化、slot `occupants` 里的新注册、插件页新描述，都要在重启后出现。

**P7｜link 安装下 remove/reinstall 有真实风险**

- 现象：在目录 link 安装状态下执行 remove/reinstall，工作区里的包目录被清空（本机复现过一次，原因未完全定位）。
- 做法：项目早期就切到 tarball 安装；需要改代码就重新打包再装，不要依赖 link 状态下的删除/重装。
- 验证：`remove_bundle` 前先确认工作区有 git 提交或副本。

**P8｜entry id 与 slot key 的前缀问题**

- 现象：配置页/行配置注册了但界面不出现。
- 根因：`Config.listConfigs` 的 id 是 `include:<patch id>`，而 profile patch 里的 id 是 `<patch id>`；不同 UI 层取不同的 id。
- 做法：`settings.describe`/配置表单的 namespace 用 patch id（如 `self-organizing-agent-team`）；`plugins.row.config` 的 key 用 `<包名>#<行 id>`，行 id 以 bundle patch 声明为准；不确定时两个 key 都注册：
  - `<pkg>#<patch id>`
  - `<pkg>#include:<patch id>`
- 验证：`Slots.listSubTree` 的 `occupants` 里出现对应 key。

## 4. Host 半

**P9｜`apply` 里抛错 = 整行 failed**

- 现象：插件行红色 failed，日志里只有 import/throw。
- 根因：`apply`（含 `Config` 解析、读文件、注册服务）在激活时执行，抛错即失败。
- 做法：`apply` 只做注册；把可能失败的 I/O 放进 `try` 并给出可读错误；所有注册都放 `ctx.effect`/`ctx.on`，返回清理器。
- 验证：`plugin_manager list_plugins` 的 `fiberPhase: active`。

**P10｜可编辑 Config 字段必须 `.volatile()`**

- 现象：声明了 `Config`，插件页却没有配置表单。
- 根因：settings 只把 **volatile 字段**投影成可编辑表单；普通字段只是 schema 的一部分。
- 做法：

  ```js
  export const Config = z.object({
    autoLoad: z.boolean().default(true).volatile(),
  })
  ```

  运行时是引用，读值用 `config.autoLoad.get()`（不要直接当 boolean 比较）。
- 验证：`Config.listConfigs` 的 schema 中出现 `x-cordis.volatile: true`；改值后 profile patch 出现该行的 override。

**P11｜“默认注入”用 `agent/created` + agent 作用域 section**

- 做法：`ctx.on('agent/created', ({ agent }) => agent.ctx.effect(() => agent.ctx.systemPrompt.section({ name, order, text }), 'label'))`。
- 注意：
  - 注册在 agent ctx 上，agent 销毁自动清理；
  - resume 也会触发 `agent/created`，不会重复叠加；
  - 每个新 agent（含 teammate/subagent）都会带上，成本要算进上下文预算；
  - 关闭开关时不要再注册；配置变化对“之后创建的 agent”生效。
- 不要在 `agent/request` 里改消息，也不要在每个请求里重复注入大段文本。

**P12｜`ctx.skills.register` 与文件系统 skill 会打架**

- 现象：插件注册了技能，但 `skill` 工具加载到的却是 `~/.dsh/skills` 里的旧版本。
- 根因：文件系统 provider 在 preset 作用域层，比插件在全局层的 runtime 注册更近，同名时它赢。
- 做法：只保留一个来源；bundle 注册技能时不要把“文件系统 fallback”也装上；文档里写清“二选一”。
- 验证：`skill` 工具返回的 `Base directory` 指向哪里。

## 5. Client 半

**P13｜Client 插件格式**

- 必须是 `window.__ModuleLoader__.load({ id: '<包名>', factory(require) {...} })`；`id` 必须等于包名。
- `package.json` 的 `dsh.client`：`platform: 'web'`、`immediately`、`inject`（只用于激活顺序，不是模块导入）。
- 不要把 `@deepseek-ai/dsh-client-ui-primitives` 之类的 Harness Client 包 `require` 进来；复制需要的控件并改用 `--dsw-alias-*` 主题 token。

**P14｜slot 注册带不带 `locale`，决定组件有没有 `t`**

- 现象：设置页是空白的；console 里是 `t is not a function` 一类错误。
- 根因：组件的 `t` 来自注册选项的 `locale: '<命名空间>'`；没写就没有 `t`。
- 做法：`ctx.locale.register(NS, { en, zh })` 后，注册时带 `locale: NS`；翻译键两边都要有。
- 验证：页面能渲染；切语言后文案跟着变。

**P15｜资源与清理**

- 所有样式、监听、计时器放 `ctx.effect` 并返回 disposer；不要写 `document.body`、不要 iframe、不要读别的插件 DOM。
- 验证：卸载/刷新后没有残留。

## 6. 配置 UI（真实教训最多的一段）

DSH 的插件页**不会自动渲染 Config**。配置 UI 有三种注册方式：

| slot | key | 用途 |
|---|---|---|
| `plugins.row.config` | `<包名>#<行 id>` | 行上的 **Configure** 页 |
| `plugins.bundle.config` | 包名 | bundle 页里的配置区 |
| `settings.section` / `settings.general.item` | 自定义 id | Settings 里的页面/行 |

**P16｜只声明 Config 没有用**

- 必须注册上面的某个 slot；否则插件页只有卡片和行，没有表单。

**P17｜非 volatile 字段不会出现在表单里**

- 见 P10；这是“表单不存在”的最常见原因。

**P18｜row key 的前缀需要兼容**

- 见 P8；文档说“行 id 以 patch 声明为准”，但实际组合层可能出现 `include:` 前缀，双注册最稳。

**P19｜表单值的读取与保存**

- 两种来源：
  - 页面 owner 传入的 `form.state` + `form.mutate(operations, expectedRevision)`；
  - 自己 `ctx.configForms.get(ns)` 拿 form，作为 `inject` face 的 hook 交给组件。
- 保存路径是 profile 的 `cordis.patch.yml`；改完检查文件，再新建 agent 验证行为。
- 只验证“开关能点”不够：**关一次、开一次，两个方向都看落盘和实际行为**。

## 7. 验证与排障

没有浏览器控制时，用 inspect 做到“结构级验证”：

```text
cordis_inspect_list
cordis_inspect_query(host, Tool, listTools)
cordis_inspect_query(host, Service, listService[service])
cordis_inspect_query(host, Config, listConfigs{name}|{entry})
cordis_inspect_query(client, Slots, listSubTree{root})
```

- `Slots.listSubTree` 的 `occupants` 是“注册成功”的硬证据；`keyDomain` 的 “already taken” 能看出 key 是否被占用。
- `Config.listConfigs` 的 `status: schema` + `x-cordis.volatile` 能判断表单是否会被投影。
- 不要伪造截图；没有浏览器控制就报告“视觉未验证”（官方 `verification.md` 的要求）。

**P20｜重启 vs 刷新**

| 变化 | 需要 |
|---|---|
| 新装一个 bundle | 可能 HMR 生效，必要时重启 |
| 替换已安装包（Host/Client 代码变了） | **重启** |
| 只改配置值 | 通常即时生效，但只影响之后创建的 agent |
| 只改文档/README | 无需 |

**P24｜团队消息：工具成功 ≠ 正文送达**

- 现象：`send_message` 返回 `{sent:true}`，队友却回一句"你想做什么"。`list_agents` 一切正常。
- 根因：`send_message` 只在 steer 被接收时报告成功（目标 inbox 接收），之后没有回执——不确认已完成持久 flush，也不确认模型读了它。`spawn_teammate` 相反：初始 prompt 会在成员转 `active` 前 flush 并核对，核对失败抛 `TEAM_PROVISIONING_CONFLICT`，不会留下"成功但没正文"的成员。
- 做法：不要用 `list_agents` 证明送达（它只证明 tool 在）。用 nonce 往返：spawn 时要求队友把随机串写进 workspace 文件并原样回传，核对文件与结束消息；要验 `send_message` 就再发一个随机串做第二轮。spawn 的名字永久占用，按宿主构建做一次，不要每个任务做。
- 验证：文件内容在去掉可选 BOM 或首尾空白后等于随机串、结束消息原样回传，且时间戳晚于探针开始；`send_message` 抛错则说明消息根本没被接收。

## 8. 发布工程

**P21｜发布的压缩包不可复现**

- 现象：本地和 CI 打出的 zip hash 不同。
- 根因：排序受平台大小写影响、时间戳/权限/`create_system` 不一致、文本 CRLF 差异、zlib 版本差异。
- 做法：ZIP 用 stored（不压缩）、固定时间戳、`create_system=3`、固定权限、按 POSIX 路径排序、文本 LF；tarball 的 gzip 很难跨 zlib 复现，**以发布附带的 `SHA256SUMS` 为准**。
- 验证：两个 Python 版本本地打包结果一致；CI 资产与本地一致。

**P22｜行尾会让内容 hash 漂移**

- 现象：CI 上的快照校验失败，本地正常。
- 根因：Windows 检出 CRLF、Linux 检出 LF。
- 做法：`.gitattributes` 写 `* text=auto eol=lf`；快照类 pin 在 LF 规范化后重算。

**P23｜发布流程要有闸门**

- GitHub Actions 用 `workflow_dispatch` + `dry_run` 输入；正式发布放 `release` environment，配 Required reviewers。
- 仓库 Actions 设置需要 Write 权限，否则 `GITHUB_TOKEN` 无法建 Release。
- Release 资产至少包含：可安装包、校验和、发布说明（compat pin、commit、验证记录）。
- 不要把 PAT 写进仓库、脚本或日志。

## 9. 与其他插件/ skill 共存

- 一个包名 + 一个 bundle patch；行 id 稳定，不要把行 id 当版本号。
- 同一 slot 的 keyed 注册：后注册会替换前者；多个 bundle 抢同一个 key 会互相覆盖。
- 和 skill 共存时只保留一个来源（P12）。
- 依赖 DSH 版本时谨慎写 `engines.dsh`/`peerDependencies`：不满足会阻止安装；实验性包可能跨版本变 schema。
- `@deepseek-ai/dsh-experimental-agent-team` 随包带了 `lib/types/mailbox.js`（持久队列 + delivered 确认 + 恢复重试），但 DSH 44.7.0 的 `TeamService` 没有 import 它；不要用这个文件论证运行时送达语义（版本 pin：`0.2.1-rc.2`）。

## 10. 文档地图

| 想查 | 看哪里 |
|---|---|
| bundle manifest、display metadata、icon、安装观察 | `cordis-plugin-development/references/host-plugin.md` |
| Client 模块格式、slot、主题、locale | `references/ui-plugin.md` + `references/practices.md` |
| 配置页 slot、行 key、form.state/mutate | `@deepseek-ai/dsh-client-ui-plugin-manager` README |
| volatile、settings 表单投影 | `@deepseek-ai/dsh-settings` README |
| skill 注册、发现根、invocation 策略 | `@deepseek-ai/dsh-skill` / `dsh-skill-filesystem` / `dsh-tool-skill` README |
| provider 组合、patch 语法 | `cordis-composition-reference` skill |
| 无浏览器验证边界 | `references/verification.md` |
| 同一操作给 UI 和 agent 工具 | `references/user-actions.md` |

## 附录 A：症状 → 先查什么

| 症状 | 先查 |
|---|---|
| 插件页没有表单/开关 | 是否注册 `plugins.row.config`/`plugins.bundle.config`；字段是否 `.volatile()` |
| 设置页空白 | slot 注册是否带 `locale`；console 是否组件崩溃 |
| 行 failed to import | 依赖是否随 tarball 安装；是否 link 安装 |
| 装完/改完行为没变 | 是否替换包需要重启；是否双装了文件系统 skill |
| 配置能点但不保存 | 字段是否 volatile；profile `cordis.patch.yml` 是否写入 |
| 技能加载到旧版本 | `skill` 返回的 base directory；是否有第二来源 |
| 消息发出但队友没反应 | 别只看 `list_agents`；用 nonce 往返探针（P24） |
| 资产 hash 不一致 | 排序、行尾、时间戳、`create_system`、zlib |

## 附录 B：最小验收清单

1. `install_bundle` → `application: applied`，行 `fiberPhase: active`。
2. `Slots.listSubTree` 里看到你的 slot occupant。
3. `Config.listConfigs` 的 schema 包含可编辑字段（且 volatile）。
4. 改写配置 → profile patch 有 override → 新建 agent 行为变化（关/开各验一次）。
5. 技能目录/技能工具返回的来源路径正确。
6. 发布资产：`SHA256SUMS` 全 match；tarball 能安装；Release 说明含 compat/commit。

## 版本记录

- `v0.1.0`（2026-10-18）：初版，基于 DSH Desktop 44.7.0 与一次真实 bundle 开发（`self-organizing-agent-team`）。
- `v0.1.1`（2026-10-18）：补英文版（`DSH-PLUGIN-PITFALLS.en.md`）与中英文 README，内容与 v0.1.0 等价。
- `v0.1.2`（2026-10-18）：新增 P24（Team 消息投递语义与 nonce 往返探针）、附录 A 对应症状行，以及 §9 的 `mailbox.js` 未接线提醒。
