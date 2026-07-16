# manabibashi/workflows — 共通ワークフロー置き場

組織全リポジトリの CI と Dependabot 自動マージの**実体**をここに集約する(REPO_STANDARD §5)。
各リポジトリには `templates/` の薄い呼び出しファイルだけを置く。
実体を修正 → `v1` タグを付け替えれば、全リポジトリに即反映される。

## ファイル構成

| このバンドルのファイル | 配置先 |
|---|---|
| `ci.yml` | このリポジトリの `.github/workflows/ci.yml` |
| `dependabot-automerge.yml` | このリポジトリの `.github/workflows/dependabot-automerge.yml` |
| `caller-ci.yml` | このリポジトリの `templates/caller-ci.yml`(→ 各リポジトリの `.github/workflows/ci.yml` へコピー) |
| `caller-dependabot-automerge.yml` | このリポジトリの `templates/caller-dependabot-automerge.yml`(→ 各リポジトリへコピー) |

## セットアップ手順(初回のみ)

1. **public** リポジトリ `manabibashi/workflows` を作成し、上記の配置でコミットする
   - **public であること**が必須【E-2 改】。private だと、組織内の public リポジトリから
     reusable workflow を参照できず、caller が startup_failure で落ちる
     (Actions のアクセス設定を「組織内から参照可」にしても解決しない)。
     共通ワークフローはシークレットを含まない CI 定義のみなので public でよい
   - private で運用する場合は Settings → Actions → General → Access を
     「Accessible from repositories in the organization」にする必要がある(public なら不要)
2. `pinact run` を実行して `uses:` のタグ参照を full-length SHA に固定 → コミット(D-3)
   - **caller の `manabibashi/workflows@v1` 参照は SHA 化しない**(v1 の付け替えで伝播させるため。
     REPO_STANDARD §5 の猶予期間中の許容例外)。pinact は caller も SHA 化してしまうので、
     実行後に `@v1` へ戻すこと
3. タグを打つ: `git tag v1 && git push origin v1`
   - 以後、実体を更新したら `git tag -f v1 && git push -f origin v1` で付け替える
   - 参照側(`@v1`)は自動で新実装を使う。挙動を固定したいリポジトリは SHA 参照に変えてもよい
4. dependabot.yml(github-actions エコシステム)をこのリポジトリ自身にも置く

## 各リポジトリへの展開手順(1 リポジトリ 15 分)

1. `templates/` の 2 ファイルをコピーし、`with:` を実態に合わせる
   - `runtime` / `test-command` / `working-directories` を設定
   - `operating-systems`: 既定は ubuntu のみ。Windows でも検証するリポジトリだけ
     `'["ubuntu-latest","windows-latest"]'` を渡す(os × working-directories の matrix になり、
     全レッグが `required-check` の集約対象に入る = Windows の失敗もマージを止める)
   - **automerge の `level`**: 実テストが回るなら 2、無ければ 3(REPO_STANDARD §4)
   - 既存の独自 automerge ワークフローは削除する
2. PR を作成し、Checks 欄に `ci / required-check` が出て成功することを確認 → マージ
3. リポジトリに custom property `ci=standard` を付与
4. (組織側・初回のみ)組織ルールセット `standard-ci` を作成:
   - 対象: custom property `ci=standard` のリポジトリ
   - ルール: Require status checks(`ci / required-check`)、Allowed merge methods: Squash のみ、
     Require linear history: ON
   - **順序厳守**: property の付与は必ず手順 2 のマージ後(先に付けると PR がマージ不能になる)
5. Dependabot を手動トリガー(Insights → Dependency graph → Dependabot → Check for updates)し、
   自動マージが期待通り動くことを確認

## 既知の制約

- **GITHUB_TOKEN によるマージは、push:main トリガーの Actions ワークフローを起動しない**(再帰防止仕様)。
  Actions で main へのデプロイを行うリポジトリでは、自動マージされたコミットのデプロイが走らない。
  - **Actions で main へデプロイするリポジトリが該当する**。ただし該当リポジトリでも、Dependabot の
    対象が github-actions のみでデプロイ成果物が変わらない場合は実害が無い。
  - **GitHub App 連携(Cloud Build 等)でデプロイするリポジトリは影響を受けない**。GITHUB_TOKEN の
    マージでもデプロイが発火するため。
  - どのリポジトリが該当するかは、private の REPO_STANDARD.md「GITHUB_TOKEN 制約の該当リポジトリ」を参照。
  - 将来必要になったら GitHub App トークン方式に切り替える
- グループ PR の update-type 判定は「グループ内で最も大きい更新種別」
- reusable workflow の権限は呼び出し側を超えられない。automerge の caller には
  `contents: write` / `pull-requests: write` の明示が必須
- actions のメジャーは Node 24 世代を使う(Node 20 ランナーは 2026-09-16 に完全削除され、
  @v4 世代は動かなくなる)。dependabot(github-actions)が major 更新を個別 PR で提案するので、
  放置せず取り込むこと
- 組織の「Require actions to be pinned to a full-length commit SHA」を ON にすると、`@v1` タグ参照も
  違反になる見込み(自組織参照が例外になるかは ON 前に要確認)。ON へ移行する際は各 caller を
  SHA 参照へ切り替える。以後の共通ワークフロー更新の伝播は「v1 タグ付け替えで即時」から
  「Dependabot の SHA 更新 PR(週次)」に変わる
