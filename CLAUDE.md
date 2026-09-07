# CLAUDE.md — manabibashi/workflows

## 組織共通ルール(REPO_STANDARD 準拠)

- この節は REPO_STANDARD.md 付録 D のコピー。リポジトリ固有の注記はこの下の H2 節に書く
  (共通節は改訂のたびに機械的に差し替えるため、ここへの書き足しは失われる)
- manabibashi の REPO_STANDARD.md に準拠する。手元に無ければ
  gh api repos/manabibashi/.github/contents/REPO_STANDARD.md -H "Accept: application/vnd.github.raw+json" で取得
- ブランチは main のみ。短命ブランチ → PR → squash。Claude Code は push と PR 作成まで、マージはユーザー【A-6】。
  force push・ブランチやタグの削除・リポジトリ設定変更などの破壊的操作はユーザー確認必須
- ローカル同期・マージ済みブランチの掃除・push の範囲・投稿の帰属【A-4〜A-7】は組織プラグイン
  `manabibashi-standard` のフックが担保する。未導入なら REPO_STANDARD §2 / §8 の手順に従い、
  破壊的な git 操作の前に必ずユーザー確認を取る
- PR を作る前に `/code-review` と `/security-review` を実行する(未実施の PR 作成はフックが止める)
- テストを実行できるリポジトリのロジック変更は TDD(失敗するテスト → 実装 → リファクタ)で進める
- 複数行の本文(コミット・PR・Issue・コメント)は Write でファイルに書いて `-F` / `--body-file` で渡す。
  数行を超える処理はシェルに埋め込まず、Node スクリプトをファイルに書いて実行する
- タスク・ステータスは GitHub Issues で管理する。独自のタスクファイルを作らない
- プラン名や時限的な外部仕様をドキュメントに書かない(書くなら日付を添える)
- GitHub の参照・操作は GitHub MCP(接続済みなら)または gh CLI を使う
- GitHub Actions は各 action の最新メジャーを確認して書く。EOL 予定・EOL 済みのランナー世代を新規に書かない
- この共通節より下にリポジトリ固有の節を H2 見出しで置く(共通節配下の H3 に入れ子にしない)。
  そこに dependabot 構成・automerge level・デプロイ方式・環境名の由来を記録する

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
  自分自身を `@v1` で呼ぶと v1 タグ付け替え時の挙動が読みにくくなるうえ、GITHUB_TOKEN による
  自動マージは push:main トリガーを起動せず `retag-v1.yml` が走らないため。

### 更新の伝播
- 実体(`.github/workflows/` 配下)を修正 → main へマージ → `retag-v1.yml` が v1 を自動で付け替える【G-6】。
  参照側(`@v1`)は次回実行から新実装を使う。手動の `git tag -f v1 && git push -f origin v1` は
  最後の手段(force push のためユーザーが実行)。
- `retag-v1.yml` 自身も `.github/workflows/` 配下なので、その変更でも v1 が動く(実害なし)。
- `uses:` は full-length SHA でピン留めする(`pinact run`、D-3)。templates/ の `manabibashi/workflows@v1`
  参照は SHA 化しない(reusable workflow は組織設定「SHA ピン留め必須」の対象外。REPO_STANDARD §5)。

### Actions のアクセス設定
- Settings → Actions → General → Access = **Accessible from repositories in the organization**。
  これが private のままだと全リポジトリの caller が失敗する。
