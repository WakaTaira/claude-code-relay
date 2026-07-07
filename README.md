# claude-code-relay

Skills and subagent definitions that turn a Claude Code main session into a design / orchestration / audit layer, delegating implementation to seven fixed `relay-*` subagents routed by difficulty (opus → haiku) — with an optional `/goal`-driven unattended loop (`RELAY-STATUS` protocol).

Meant to run **after the design is settled** — e.g. via plan mode or an interview skill such as [grilling / grill-with-docs](https://github.com/mattpocock/skills/tree/main/skills/engineering/grill-with-docs). relay takes the agreement as given and drives the implementation phase: task decomposition → dispatch → audit → verification, stopping only where human judgment (design changes, hands-on testing) is required.

- `en/` — English version, `ja/` — Japanese version (same workflow; `ja/` keeps a few environment-specific model notes)
- Install: copy `skills/*` into `~/.claude/skills/` and `agents/*.md` into `~/.claude/agents/` from the language of your choice
- `relay` = fable-orchestrated main session; `relay-opus` = Opus-orchestrated variant
- Unattended loop: arm `/goal RELAY-STATUS: DONE or BLOCKED has been declared`, then invoke relay

---

Claude Code のメインセッションを設計・統括・監査レイヤーに専念させ、実装を難度別の固定サブエージェント `relay-*` 7 体（opus → haiku）へ振り分けるスキル＋エージェント定義。`/goal` 併用の無人ループ（`RELAY-STATUS` プロトコル）に対応。

**設計が固まってから**使う想定（plan mode、または [grilling / grill-with-docs](https://github.com/mattpocock/skills/tree/main/skills/engineering/grill-with-docs) のような設計詰めスキルの後段）。合意済み設計を所与として、タスク分解 → ディスパッチ → 監査 → 検証を自走させ、人間の判断（設計変更・実機テスト）が要る所でだけ止まる。

- `en/` が英語版、`ja/` が日本語版（ワークフローは同一。`ja/` には環境固有のモデル運用メモが残る）
- 導入: 好きな言語の `skills/*` を `~/.claude/skills/` へ、`agents/*.md` を `~/.claude/agents/` へコピー
- 無人ループ: `/goal RELAY-STATUS: DONE または BLOCKED が宣言されている` を張ってから relay を発動

License: MIT
