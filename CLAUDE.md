# CLAUDE.md — manabibashi/workflows

## 組織共通ルール(REPO_STANDARD 準拠)
- このリポジトリは manabibashi の REPO_STANDARD.md に準拠する。手元に無ければ
  gh api repos/manabibashi/.github/contents/REPO_STANDARD.md -H "Accept: application/vnd.github.raw+json" で取得
- ブランチは main のみ。作業は短命ブランチ → PR → squash。force push・ブランチ削除・
  リポジトリ設定変更などの破壊的操作はユーザー確認必須
- 例外【A-4】: マージ済み**ローカル**ブランチは、① `git fetch --prune` 後に upstream が gone
  ② 対応 PR がマージ済み(`gh pr list --state merged --head <branch>`)
  ③ ブランチ先端 SHA が当該 PR の headRefOid と一致、または PR のコミット一覧
  (`gh pr view <番号> --json commits --jq '.commits[].oid'`)に含まれる(未 push コミット無し。
  PR の「Update branch」で main を取り込むと headRefOid はローカルに無いマージコミットになる)
  の 3 条件を検証できた場合のみ、確認なしで `git branch -D` で削除してよい
  (squash 運用のため `-d` は失敗する)。リモートブランチの削除は対象外(確認必須のまま)
- push / マージの最終実行はユーザー判断。Claude Code は PR 作成まで
- タスク・ステータスは GitHub Issues で管理する。backlog.md 等の独自ファイルを作らない
- プラン名や時限的な外部仕様をドキュメントに書かない(書く場合は日付を添える)
- GitHub の参照・操作は GitHub MCP(接続済みなら)または gh CLI を使う
- GitHub Actions を書く際は各 action の最新メジャーを確認してから使う(GitHub MCP または gh api)。
  Node 20 世代(actions/checkout@v4 等)を新規に書かない
- 下の「リポジトリ固有」節に、dependabot 構成・automerge level・デプロイ方式・
  環境名の由来(ブランチ整理で消してはいけない名前)を記録する

## リポジトリ固有

このリポジトリは組織共通ワークフローの**実体**の置き場(REPO_STANDARD §5・E-2)。
自分自身のアプリケーションコードは持たない。

### 構成
- `.github/workflows/ci.yml` — 共通 CI の実体(`workflow_call`)
- `.github/workflows/dependabot-automerge.yml` — 共通 automerge の実体(`workflow_call`)
- `templates/caller-*.yml` — 各リポジトリの `.github/workflows/` へコピーする薄い呼び出し(付録 C)

### dependabot 構成
- `github-actions` のみ(directory `/`)。マニフェストが無いため他のエコシステムは無し。

### automerge level
- **このリポジトリ自身は共通 automerge の caller を置いていない**(Dependabot PR は人間がマージする)。
  自分自身を `@v1` で呼ぶと、v1 タグ付け替え時の挙動が読みにくくなるため。

### 更新の伝播
- 実体を修正 → main へマージ → `git tag -f v1 && git push -f origin v1` で v1 を付け替える。
  参照側(`@v1`)は次回実行から新実装を使う。
- `uses:` は full-length SHA でピン留めする(`pinact run`、D-3)。templates/ の `manabibashi/workflows@v1`
  参照は猶予期間中の許容例外(REPO_STANDARD §5)。

### Actions のアクセス設定
- Settings → Actions → General → Access = **Accessible from repositories in the organization**。
  これが private のままだと全リポジトリの caller が失敗する。
