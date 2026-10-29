# 移植・統合仕様書 (Integration Guide)

このドキュメントは、`discord-permission-auditor` の権限監査・自動修復ロジックを、総合Bot（Fort9, BotOS等）にモジュール・Cogとして移植・統合するための技術仕様書です。

---

## 1. 必要な前提条件

### Discord Bot 権限 (Intents & Permissions)

移植先Botで以下の **特権インテント** および **サーバー内権限** を有効化してください。

- **Gateway Intents**:
  - `Intents.guilds = True`
  - `Intents.members = True` (サーバーメンバー一覧、ロール保持者、オーナー判定に必須)
- **Bot Permissions**:
  - `View Audit Log` (監査ログの閲覧)
  - `Manage Roles` (ロールの編集・危険権限の解除)
  - `Manage Channels` (チャンネル権限の確認・上書き)
  - `Manage Webhooks` (不要・孤立Webhookの検出と削除)
  - `Manage Server` (無期限招待リンクの検出・削除)

---

## 2. モジュール構成と責務

```text
discord-permission-auditor/
├── auditor.py       # コア監査ロジックおよび各修復関数 (fix_*)
├── findings.py      # Finding データモデル、重要度 (Severity)、Embed 生成
├── risks.py         # Raid / Nuke / 外部攻撃リスクの総合スコア算出
├── config.py        # 監査ルールの有効/無効、無視対象ロール・チャンネル設定
└── cogs/
    ├── audit.py     # /audit, /audit-channel, /audit-mention スラッシュコマンド
    ├── fix.py       # /fix 自動修復UI (SelectMenu + Modal確認)
    └── help.py      # ヘルプ表示 Cog
```

---

## 3. 総合Botへの組み込み手順

### パターンA: Cogとしてそのまま読み込む場合

1. 移植先Botのプロジェクト配下に本リポジトリのソース一式（`auditor.py`, `findings.py`, `risks.py`, `config.py`, `cogs/`）をコピー。
2. Bot起動時のセットアップフックで Cog をロード:

```python
await bot.load_extension("cogs.audit")
await bot.load_extension("cogs.fix")
await bot.load_extension("cogs.help")
```

### パターンB: 単一機能関数として内部から直接呼び出す場合

定期実行タスクや管理コマンドから非同期で直接監査を実行する場合:

```python
import config
from auditor import run_audit
from risks import calculate_risks
from findings import build_summary_embed, build_text_report

async def perform_server_security_check(guild: discord.Guild):
    cfg = config.load_config()
    findings, check_count = await run_audit(guild, cfg)
    risks = calculate_risks(findings)
    
    # Embed形式でレポートを取得
    embed = build_summary_embed(guild.name, findings, check_count, risks)
    return embed, findings
```

---

## 4. 監査ルール一覧 (`check_id`)

`run_audit(guild, cfg)` で実行されるチェック一覧:

| check_id | 重要度 | 内容 | 自動修復 |
|---|---|---|---|
| `bot_perm_selfcheck` | INFO | Bot自身の監査用権限不足チェック | 不可 |
| `everyone_excess` | CRITICAL〜MEDIUM | `@everyone` に付与された危険権限（管理者、BAN、Kick等） | **可能** |
| `server_misconfig` | HIGH〜LOW | 2FA必須化、認証レベル、メディアフィルター、招待作成、Webhook管理権限 | **可能** (一部) |
| `role_inheritance` | LOW | 上位ロールと重複する下位ロールの冗長権限 | 不可 (警告のみ) |
| `everyone_visible` | INFO | `@everyone` が閲覧可能なチャンネル一覧 | 不可 (手動推奨) |
| `mention_everyone` | HIGH〜MEDIUM | 一般メンバーに付与されている全員メンション権限 | **可能** |
| `external_bot_usable` | MEDIUM〜LOW | 外部アプリ・ユーザーインストールBot実行権限の開放状態 | **可能** |
| `stale_invites` | LOW | 無期限・有効期限なしの招待リンク | **可能** |
| `owner_admin_roles` | HIGH | オーナー以外の一般メンバーに管理者ロールが付与されている状態 | 不可 |
| `integration_webhooks`| MEDIUM | 作成者が退去済みで放置された孤立Webhook | **可能** |

---

## 5. 自動修復 (`auto_fixable`) の安全設計方針

- **二重確認モーダル**: `/fix` コマンドは SelectMenu で選択後、影響範囲を明示した `FixConfirmModal` を経由して実行されます。
- **管理者権限の再検証**: 実行直前および確認時にも `interaction.user.guild_permissions.administrator` を検証し、権限昇格を防ぎます。
- **破壊的変更の防止**:
  - 公開チャンネル（`everyone_visible`）の一括非表示化は安全のため `auto_fixable=False` に固定。
  - ロール権限の削除は、確実に同一ロール・同一権限名（`perm_name`）が一致した場合のみ安全に解除されます。
