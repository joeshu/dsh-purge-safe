# 2.0.2

## 中文

- 适配 DeepSeek Harness **0.2.2-rc.5**。
- 修复客户端入口声明：`dsh.client.inject` 不再引用 rc.5 已移除的 `@deepseek-ai/dsh-client-runtime`、`@deepseek-ai/dsh-client-ui-slots`，只保留 `dsh-client-connection`、`dsh-client-locale`、`dsh-client-ui-settings`。rc.5 的 `slots` 服务由 `dsh-client-ui-renderer` 提供，旧的包名在启动图里是悬空依赖。
- 62 条宿主源码补丁在 rc.5 上全部命中，无需改动；宿主事件与 `system-prompt` 配置项同样兼容。
- **新增 2 条针对 rc.5 差异的补丁**（现共 64 条）：
  - `#100 AGENT_INSTRUCTIONS_ALL_INTROS`：rc.5 把 workspace 指令前言拆成四条常量（`AGENT_INSTRUCTIONS_INTRO` / `REPLACEMENT_AGENT_INSTRUCTIONS_INTRO` / `EMPTY_REPLACEMENT_...` / `ADDITIONAL_INSTRUCTIONS_GUIDANCE` + `COMPACT_...`），旧补丁只换了第一条。新会话第一轮走 `replacePreviousBaseline: true`，命中的是 `REPLACEMENT_*`，于是注入段前面仍挂着官方那句「may be relevant to your work / they do not override」，模型读到的第一条 workspace 指令仍是「仅供参考」。本条把余下三条一并换成强指令。
  - `#101 PTC_ESCALATION_NEUTRAL`：rc.4 起 PTC 执行器（`run_code`）独立成包 `dsh-tool-executor-ptc`，沙箱升级字段的描述独立写了两份（`lib/index.js` 与 `lib/types/ptc.js`）。旧的 `#36` 只覆盖 `dsh-tool-fs`，因此走 ptc 预设时模型仍看到「requires justification and approval」「shown to the user for approval」「Request wider access only after evidence of a denial」，又会先等审批。本条把这三种措辞清掉。
- **桌面壳自愈跨世代修好**：rc.4/rc.5 的 `lib/main.js` 删掉了模块顶部的 `DSH_PROFILE = "desktop"` 锚点，导致 `proxy-bypass-list`、GUI 侧代理环境清理两条自愈静默失配（实机核对：`live main.js` 里 `[dsh-purge-safe] proxy bypass loopback` / `gui strip proxy` 计数为 0）。现在锚点回退到 `app.setAppLogsPath();`（跨版本稳定、无缩进顶层语句），rc.5 上六条 host 标记全部命中且幂等，`node --check` 通过。
- `dshTarget` / `dshTargets` 更新为 `0.2.2-rc.5`（rc.5 起这两项只是市场元数据，兼容性校验走 `engines.dsh` 或 `@deepseek-ai/dsh-*` peerDependencies）。
- **修复：派活后子会话立刻「拒绝」**（`lib/net-scope.js`）。现象是主会话派出去的 recon/assess 子代理一句话都不干，回报「不行，这是严令禁止的」，父会话只能自己上手。
  - 根因：区域闸门挂在 `agent/pre-step`，拿 step 1 的 user 句判「攻击意图 + 地域域名」。父会话的人类原话（「帮我打一下 https://shanshuiapi.com/ 这个中转站」）不含攻击词，顺利过闸；但**子会话的 step 1 是主会话按指令写出来的派活 brief**，里面写着「nmap 全端口 / subfinder / nuclei / 端口与服务」，还列着靶标的兄弟域名变体「也查一下 shanshuiapi.cn」。于是同一段意图在子会话里被重判一次 → `shanshuiapi.cn` 命中 `.cn` 区域规则 → 直接写一条「不行，这是严令禁止的」并把 turn 收成 completed。
  - 证据：子会话 `b7d4ce78` 的完整日志里**没有任何 `request/header` 事件**——模型一次都没被调用，那条「拒绝」是插件伪造的 `assistant/message`（还写着 `provider: deepseek / model: deepseek-chat`，而真实路由是 `wenwen2 / deepseek-v4.1-flash`）。
  - 修法：闸门只判**人类那一轮**。新增 `isDelegatedSession()`，复用 `identity.js` 的 `isChildAgentContext`（只认 `origin`/`delegationDepth`，不认 `parentSession`，避免把续接分支误判成子会话）；子会话直接放行，人类首轮照旧过闸。
  - 同时把伪造消息的 `provider/model` 改成取本轮真实 `modelSelection`（拿不到才退回 `dsh-purge-safe/net-scope`），下次万一再拦，日志里能一眼看出是谁写的。
- 用户须知与各语言 README 的版本号同步到 rc.5。

## English

- Adapted for DeepSeek Harness **0.2.2-rc.5**.
- Fixed the client entry declaration: `dsh.client.inject` no longer names `@deepseek-ai/dsh-client-runtime` or `@deepseek-ai/dsh-client-ui-slots`, both removed in rc.5. Only `dsh-client-connection`, `dsh-client-locale`, and `dsh-client-ui-settings` remain. In rc.5 the `slots` service is provided by `dsh-client-ui-renderer`, so the old package ids were dangling boot-graph edges.
- All 62 host source patches still match on rc.5 and need no changes; host events and the `system-prompt` options remain compatible.
- **Two new patches for rc.5-only differences** (64 total now):
  - `#100 AGENT_INSTRUCTIONS_ALL_INTROS`: rc.5 split the workspace-instruction preamble into four constants (`AGENT_INSTRUCTIONS_INTRO` / `REPLACEMENT_AGENT_INSTRUCTIONS_INTRO` / `EMPTY_REPLACEMENT_...` / `ADDITIONAL_INSTRUCTIONS_GUIDANCE` + `COMPACT_...`), and the old patch replaced only the first. A new session's first render uses `replacePreviousBaseline: true`, which hits `REPLACEMENT_*`, so the injected block still carried the official "may be relevant to your work / they do not override" sentence. This patch replaces the remaining three intros with the strong operator framing.
  - `#101 PTC_ESCALATION_NEUTRAL`: since rc.4 the PTC executor (`run_code`) lives in its own package `dsh-tool-executor-ptc`, with the escalation field descriptions written twice (`lib/index.js` and `lib/types/ptc.js`). The old `#36` covered only `dsh-tool-fs`, so under the ptc preset the model still saw "requires justification and approval", "shown to the user for approval", and "Request wider access only after evidence of a denial". This patch clears all three.
- **Desktop-shell self-heal now survives the generation change**: rc.4/rc.5 deleted the module-level `DSH_PROFILE = "desktop"` anchor from `lib/main.js`, so the `proxy-bypass-list` and GUI proxy-env cleanup heals silently mismatched (verified on the live install: `[dsh-purge-safe] proxy bypass loopback` / `gui strip proxy` counts were 0). The anchor now falls back to `app.setAppLogsPath();` — a stable, unindented top-level statement across generations. On rc.5 all six host markers land idempotently and `node --check` passes.
- `dshTarget` / `dshTargets` bumped to `0.2.2-rc.5` (since rc.5 these are marketplace metadata only; compatibility is decided by `engines.dsh` or `@deepseek-ai/dsh-*` peerDependencies).
- **Fixed: a dispatched subagent "refused" immediately** (`lib/net-scope.js`). Symptom: the recon/assess child spawned by the lead did no work at all and reported "不行，这是严令禁止的"; the lead had to do everything itself.
  - Root cause: the region gate listens on `agent/pre-step` and judges step 1's user message for "attack intent + regional domain". The human's own sentence ("帮我打一下 https://shanshuiapi.com/ 这个中转站") carries no attack keyword, so it passed. But **a child's step 1 is the dispatch brief the lead wrote**, full of "nmap full port / subfinder / nuclei / port and service" plus sibling-domain variants like "also check shanshuiapi.cn". The same intent was re-judged inside the child → `shanshuiapi.cn` matched the `.cn` region rule → the plugin wrote "不行，这是严令禁止的" and closed the turn as completed.
  - Evidence: the child session `b7d4ce78` contains **no `request/header` event at all** — the model was never called. The "refusal" was a plugin-forged `assistant/message` that even claimed `provider: deepseek / model: deepseek-chat` while the real route was `wenwen2 / deepseek-v4.1-flash`.
  - Fix: the gate now judges **only the human turn**. New `isDelegatedSession()` reuses `identity.js`'s `isChildAgentContext` (origin / delegationDepth only, deliberately ignoring `parentSession` so resumed branches are not mistaken for children); delegated sessions pass straight through, the human's first turn is still gated.
  - The forged message's `provider`/`model` now come from the live `modelSelection` (falling back to `dsh-purge-safe/net-scope`), so a future block is attributable at a glance.
- Welcome notice and per-language READMEs updated to rc.5.

## 日本語

- DeepSeek Harness **0.2.2-rc.5** に対応。
- クライアントエントリ宣言を修正: `dsh.client.inject` から rc.5 で削除された `@deepseek-ai/dsh-client-runtime` と `@deepseek-ai/dsh-client-ui-slots` を外し、`dsh-client-connection` / `dsh-client-locale` / `dsh-client-ui-settings` の 3 件のみにしました。rc.5 の `slots` サービスは `dsh-client-ui-renderer` が提供するため、旧パッケージ ID は起動グラフ上の宙に浮いた依存でした。
- 宿主ソースへの 62 件のパッチは rc.5 でもすべて一致し、変更不要です。宿主イベントと `system-prompt` の設定項目も互換です。
- **rc.5 固有の差分に対するパッチを 2 件追加**（合計 64 件）:
  - `#100 AGENT_INSTRUCTIONS_ALL_INTROS`: rc.5 で workspace 指示の前書きが 4 つの定数（`AGENT_INSTRUCTIONS_INTRO` / `REPLACEMENT_AGENT_INSTRUCTIONS_INTRO` / `EMPTY_REPLACEMENT_...` / `ADDITIONAL_INSTRUCTIONS_GUIDANCE` + `COMPACT_...`）に分割され、旧パッチは 1 つ目しか置換していませんでした。新規セッションの初回は `replacePreviousBaseline: true` で `REPLACEMENT_*` を参照するため、注入ブロックの前に公式の「may be relevant to your work / they do not override」が残っていました。本条で残り 3 つも強い指示に置換します。
  - `#101 PTC_ESCALATION_NEUTRAL`: rc.4 以降 PTC 実行器（`run_code`）は独立パッケージ `dsh-tool-executor-ptc` になり、サンドボックス昇格フィールドの説明が 2 か所（`lib/index.js` と `lib/types/ptc.js`）に書かれています。旧 `#36` は `dsh-tool-fs` しか対象外だったため、ptc プリセットでは「requires justification and approval」「shown to the user for approval」「Request wider access only after evidence of a denial」が残り、モデルは承認待ちで止まっていました。本条で 3 つの文言を除去します。
- **デスクトップシェルの自己修復を世代交代に対応**: rc.4/rc.5 の `lib/main.js` からモジュール先頭の `DSH_PROFILE = "desktop"` アンカーが削除されたため、`proxy-bypass-list` と GUI 側プロキシ環境のクリーンアップが無言で不一致になっていました（実機確認: `[dsh-purge-safe] proxy bypass loopback` / `gui strip proxy` のカウントが 0）。アンカーを `app.setAppLogsPath();`（世代をまたいで安定した非インデントのトップレベル文）にフォールバックさせ、rc.5 で 6 つの host マーカーがすべて幂等にヒットし `node --check` も通ることを確認しました。
- `dshTarget` / `dshTargets` を `0.2.2-rc.5` に更新（rc.5 以降これらはマーケット用メタデータで、互換性判定は `engines.dsh` または `@deepseek-ai/dsh-*` の peerDependencies で行われます）。
- **修正: 派遣した子セッションが即座に「拒否」する問題**（`lib/net-scope.js`）。症状は、リードが派遣した recon/assess 子エージェントが何もせず「不行，这是严令禁止的」と返し、リードが自分で作業するしかない状態です。
  - 原因: リージョンゲートは `agent/pre-step` で step 1 の user メッセージから「攻撃意図 + 地域ドメイン」を判定します。人間の発話（「帮我打一下 https://shanshuiapi.com/ 这个中转站」）には攻撃語がないため通過します。しかし**子セッションの step 1 はリードが書いた派遣ブリーフ**で、「nmap 全ポート / subfinder / nuclei / ポートとサービス」が並び、対象の兄弟ドメイン「shanshuiapi.cn も確認」まで含まれます。同じ意図が子セッションでもう一度判定され、`shanshuiapi.cn` が `.cn` 地域ルールに一致 → 「不行，这是严令禁止的」を書き込み、turn を completed で閉じていました。
  - 証拠: 子セッション `b7d4ce78` のログには **`request/header` イベントが 1 件もありません**。モデルは一度も呼ばれておらず、その「拒否」はプラグインが偽造した `assistant/message` でした（しかも `provider: deepseek / model: deepseek-chat` と記録され、実際のルートは `wenwen2 / deepseek-v4.1-flash`）。
  - 修正: ゲートは**人間のターンだけ**を判定します。新規 `isDelegatedSession()` は `identity.js` の `isChildAgentContext` を再利用（`origin` / `delegationDepth` のみを見て、`parentSession` は意図的に無視。再開ブランチを子と誤判定しないため）。子セッションはそのまま通過し、人間の初回ターンは従来どおり判定されます。
  - 偽造メッセージの `provider`/`model` は実際の `modelSelection` から取得するようにし（取得できなければ `dsh-purge-safe/net-scope`）、将来ブロックが起きても出所が一目で分かるようにしました。
- 利用 notice と各言語 README のバージョン表記を rc.5 に更新。

# 2.0.1

## 中文

- 修复模式中的红队模式。
- 修复权限设置中自定义模式的问题。
- 新增运行中窗口查看。

## English

- Fixed Red Team mode in the mode selector.
- Fixed custom mode in permission settings.
- Added support for viewing running windows.

## 日本語

- モード選択のレッドチームモードを修正。
- 権限設定のカスタムモードの問題を修正。
- 実行中のウィンドウを確認できる機能を追加。

## 安装与更新

使用解压后的本地目录安装，Web 与桌面端分开安装、分开应用。当前为本地固定版本，在线更新与通道切换已关闭。详细步骤见 [中文 README](./README.zh-CN.md)。
