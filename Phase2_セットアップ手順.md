# Phase 2 セットアップ手順（Claude Code on web で週次自動化）

前提: Phase 1 の `profile/` `recommend/` `.claude/` を dashboard リポジトリに配置し、GitHub に push 済みであること。

## 0. 事前確認
- Claude Code on web は **Pro / Max / Team**（または premium seat の Enterprise）で利用可。research preview。
- dashboard リポジトリが **GitHub 上にある**こと（Pages 公開元）。

## 1. リポジトリを Claude Code on web に紐付け
1. `claude.ai/code` を開く。
2. GitHub アカウントを接続し、dashboard リポジトリへのアクセスを許可（clone / push 権限）。
3. 環境のネットワークポリシーは、WebSearch と GitHub への push ができる設定にする。

## 2. サイト側の描画を一度だけ組み込み（人力・初回のみ）
- サイト本体は自動化の編集対象外なので、ここは手動で。
- `phase2/recommend_widget.html` を参考に:
  - 単体ページとして置く場合: `recommend.html` としてサイトに配置。
  - 既存ページに埋め込む場合: `<div id="anime-recommend">` と `<script>…</script>` をコピー。
- `RECOMMEND_URL` を実際のパスに調整（ルート直下配置なら `recommend/recommend.json`）。
- 動作確認: ページを開き、現行15作が表示されればOK。

## 3. 週次スケジュールタスクを登録
- Claude Code on web の該当リポジトリで、**スケジュールタスク（週1回）**を作成。
- プロンプト欄に `phase2/週次タスク_プロンプト.md` の「=== プロンプト本文 ===」以下を貼り付け。
- 実行間隔の例: 毎週月曜 07:03 頃（丁度の00分は避ける）。
- 出力ブランチ: recommend.json は**公開ブランチ（例: main）**に反映される必要がある。
  - main への直接 push を許可するか、Pages が別ブランチ配信ならそのブランチを push 先にする。
  - main にブランチ保護がある場合は、保護を緩めるか、自動マージ運用にする。

## 4. 権限の担保（再確認）
- 一次担保: 週次プロンプトが **`git add recommend/` のみ**をコミット。サイト本体は反映されない。
- 二次担保: `.claude/settings.json` が `profile/` `recommend/` 以外の編集を抑止。
  - `deny` リストに、あなたのサイトの実ファイル/フォルダ（index.html・CSS/JS/画像・_config.yml 等）を追記して調整。

## 5. 初回の動作確認（E2E）
1. スケジュールを待たず、タスクを**手動で1回起動**。
2. `recommend/recommend.json` が更新され、`history.json` に追記、`news/<日付>.md` が生成されることを確認。
3. `recommend/` 以外に差分が無いこと（`git status`）を確認。
4. push 後、公開サイトのおすすめ表示が更新されることを確認。

## 運用まとめ
| 頻度 | 作業 | 実行場所 |
|---|---|---|
| 数ヶ月に1回（人力） | dアニメ履歴 再取得 → `profile/` 更新 | Mac Studio + Claude-in-Chrome |
| 週1回（自動） | WebSearch → おすすめ更新 → push | Claude Code on web |
| 常時 | サイトで最新おすすめ閲覧 | GitHub Pages |

## 注意
- dアニメの自動取得（人力トラック）は利用規約の確認を推奨。
- Claude Code on web はプランの Claude Code 使用量・レート制限を消費（週1回の軽負荷なら影響小）。
