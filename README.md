# dアニメ嗜好おすすめ ダッシュボード（dashboard）

視聴履歴の嗜好分析に基づき、毎週ウェブから最新作を探して「おすすめ作品リスト」を自動更新し、GitHub Pages で公開するプロジェクト。

- **手動（数ヶ月に1回）**: dアニメ視聴履歴を取得・分析し、嗜好モデル（傾向分析DB）を更新。
- **自動（週1回）**: 嗜好モデルに基づき WebSearch で新作を探索・採点し、おすすめを更新して `main` に push。
- **常時**: GitHub Pages 上で最新のおすすめを閲覧。

最終整備: 2026-07。運用基盤: Claude Code on web（ルーティン）＋ GitHub Pages。

---

## 1. 全体アーキテクチャ

```
 [Mac Studio + Claude-in-Chrome]        （人力・数ヶ月に1回）
        │ dアニメ視聴履歴を取得・分析
        ▼
 profile/  … 傾向分析DB（嗜好モデル・除外リスト）   ← 読取専用の“正”
        │
        ▼
 [Claude Code on web ルーティン]         （自動・週1回）
        │ profile を読む → WebSearch で候補収集 → 7軸採点 → 除外 → 上位選定
        │ recommend/ を更新 → git push origin HEAD:main
        ▼
 recommend/  … おすすめ出力（JSON＋履歴＋ニュース）
        │
        ▼
 GitHub Pages（サイト本体 + recommend_widget） → 所有者がブラウザで閲覧
```

三層の責務を分離しているのが要点：**嗜好モデル（安定・人力更新）**、**おすすめ出力（週次・自動更新）**、**サイト表示（静的）**。

---

## 2. リポジトリ構成

```
dashboard/
├── index.html など          … サイト本体（自動化は編集しない）
├── recommend_widget.html    … recommend.json を読み描画する最小スニペット（サイトに組込済）
│
├── profile/                 … 傾向分析DB（人力・数ヶ月に1回更新／週次は読取のみ）
│   ├── profile.json         … 嗜好モデル本体（7軸・採点式・判別ルール・クラスタ・アンカー）
│   ├── seen.csv             … 既視聴/既知の除外リスト（フランチャイズ単位）
│   ├── README.md            … profile のデータ辞書
│   └── archive/             … 元の詳細資料（master_works・レポート・視聴CSV・SQLite 等。来歴保存）
│
├── recommend/               … おすすめ出力（週次・自動更新）
│   ├── recommend.json        … 現行おすすめ（サイトが読む）
│   ├── history.json          … 既推薦タイトル（重複回避）
│   └── news/YYYY-MM-DD.md     … 週次ダイジェスト記録
│
└── .claude/
    ├── settings.json          … 編集許可を profile/・recommend/ に限定（補助的担保）
    └── README_permissions.md  … 権限の担保方針
```

---

## 3. 各コンポーネントの仕組み

### 3.1 profile/profile.json（嗜好モデル）
週次自動化が参照する“正”。主なフィールド：

| フィールド | 内容 |
|---|---|
| `axes` | 深層7軸の定義：A情念 / B関係 / C手触り / D喪失 / E思弁 / F最適化(負) / G知的密度 |
| `scale` | 各軸 0=低,1=中,2=高 |
| `scoring.formula` | `A + 1.5*B + C + D + E + G - 2*F`（適合スコア。高いほど高自認の“好き”に当たりやすい目安） |
| `rule` | 高自認の“好き”を予測する判別ルール文 |
| `clusters` | k-means 5クラスタの重心と説明（深層核/思弁/情念関係/雰囲気/最適化なろう） |
| `anchors` | 本人評価＋軸スコアが確定した較正用作品（好む/好まない） |
| `notes` | 二層構造（表層＝娯楽の主食／深層＝高自認の本命）等の運用注意 |

**情報源の区別（全体で厳守）**: 「元データ（dアニメ画面の事実）」「本人申告（user_rating）」「Claudeの解釈（軸スコア・クラスタ・推薦）」を明確に分ける。軸への当てはめは主観であり事実ではない。

### 3.2 profile/seen.csv（除外リスト）
- 列: `franchise`（シリーズ接尾辞を除いた正規化名）/ `status`（watched|interested）/ `example_title`。
- 週次おすすめが「既視聴を薦めない」ために参照。

### 3.3 recommend/（おすすめ出力）
- `recommend.json`: `{generated, basis, note, items:[{tier,title,media,cluster,fit,axes,why,facts,sources}]}`。
- `history.json`: `{recommended_titles:[...], updated}`。重複推薦を防ぐ。
- `news/<日付>.md`: 週次の要約＋出典。

### 3.4 週次ルーティン（Claude Code on web）
- 場所: `claude.ai/code/routines` の週次ルーティン。プロンプト本文は `週次タスク_プロンプト_v2.md`。
- 動作: profile 読取 → WebSearch で候補収集 → **2つ以上の独立出典で実在検証** → 7軸採点 → seen/history 除外 → 上位6〜8作 → `recommend/` 更新 → `git add recommend/` → `git push origin HEAD:main`。
- 必須設定:
  - **Permissions →「Allow unrestricted branch pushes」ON**（main へ直接反映するため。OFFだと `claude/*` ブランチ止まり）。
  - **環境の Network access = Full（または対象ドメインを Custom 許可）**（WebFetch がブロックされないように）。
  - **Repeats（Schedule）= 週1回・有効**。
- 権限の担保: 一次担保は **コミット範囲を `recommend/` に限定**（サイト本体を触っても push されない）。二次担保が `.claude/settings.json`。

### 3.5 recommend_widget.html（サイト表示）
- `RECOMMEND_URL`（既定 `recommend/recommend.json`）を fetch してカード描画。外部依存なし・ライト/ダーク対応。
- 単体ページ配置、または `<div id="anime-recommend">`＋`<script>` を既存ページに埋め込み。

---

## 4. 運用手順

### 4.1 週次（自動・無操作）
スケジュールが有効なら毎週自動実行。**実行一覧の緑は「異常終了しなかった」だけで成功保証ではない**ので、時々 run を開いて更新・push・選定内容を確認する。

### 4.2 数ヶ月に1回（人力・profile 更新）
1. Mac Studio ＋ Claude-in-Chrome（dアカウントログイン済み）で、視聴履歴/コンプリート/気になるを再取得。
2. `profile/archive/dアニメ_視聴データ.csv` を更新。
3. 新たな本人評価・高自認作を `profile/profile.json` の `anchors` に追加（軸スコア付き）。
4. `profile/seen.csv` を再生成（視聴CSVを正規化）。
5. 必要なら `scoring` の重みやクラスタを再最適化。
6. `profile.json` の `version` を更新して commit & push。
- 注意: dアニメ/dアカウントの自動操作は利用規約の確認を。

---

## 5. 修正・拡張の方法（どこを触るか）

| やりたいこと | 触る場所 | 方法 |
|---|---|---|
| おすすめ件数・媒体配分を変える | 週次プロンプト | 「上位6〜8作」「媒体を分散」の記述を調整 |
| 採点の重みを変える | `profile/profile.json` | `scoring.formula` を変更（例: B の係数）。`weights_note` に理由を残す |
| 新しい本人評価/較正作を追加 | `profile/profile.json` | `anchors` に `{t,r,A..G}` を追加 |
| 実在検証を厳しくする | 週次プロンプト | 「新作アニメは公式サイト必須／小説は出版社ページ必須」等を追記 |
| クラスタ定義を見直す | `profile/profile.json` + `archive/analysis34.py` | 母集団拡張で再計算し `clusters` 更新 |
| サイトの見た目 | `recommend_widget.html` | CSS/レイアウト調整（`RECOMMEND_URL` はパスに合わせる） |
| 実行時刻・頻度 | ルーティン | Schedule プリセット、または `/schedule update` で cron（最小1時間） |
| 毎週更新の通知 | ルーティン/連携 | 通知手段を追加（メール/プッシュ等） |

**原則**: 嗜好の“意味”に関わる変更は `profile/`、選び方・出力の運用に関わる変更は週次プロンプト、見た目は widget。役割の境界を守る。

---

## 6. トラブルシューティング（実際に遭遇した事例つき）

### recommend.json が更新されない
1. **どのブランチに入ったか**: 変更が `claude/*` ブランチ止まりで `main` に無い、が最頻。→ ルーティンの「Allow unrestricted branch pushes」ON＋プロンプトの push を `git push origin HEAD:main` に。
2. **候補ゼロで据え置き**: 実在検証を通る新規候補が無い週は意図的に据え置く設計。news に「新規なし」が記録される。
3. **収集がブロック**: 環境の Network access が Trusted だと WebFetch が `403 host_not_allowed`。→ Full に。
4. **見ている場所**: `main/recommend/recommend.json` を確認しているか。サイト表示は CDN キャッシュで数分遅れることあり。

### git push が 403（書き込み拒否）
- fetch は通るが push だけ 403 → **書き込み権限が無い**。
- 原因: **Claude GitHub App 未インストール**（`/web-setup` の gh トークン連携では App は入らない）、または org リポジトリの **SSO 未認可**。
- 対処: GitHub の Settings → Applications に **Claude** が無ければ未インストール。ルーティンの GitHub イベントトリガー追加時のプロンプト、または claude.ai/code の接続設定から **App を write 権限付きでインストール**し、対象リポジトリを Repository access に追加。

### コミットが GitHub で「Unverified」になる
- コミッタ identity のズレ。`git config user.email`/`user.name` を適切に設定し、必要なら `git commit --amend --reset-author`。

### 実在しない/怪しい作品が推薦される
- 生成が事実を創作している可能性。→ プロンプトの実在検証（2出典以上・未確認は除外）を強化。news の出典URLを実際に開いて確認。

### 実行は緑なのにタスクが未達
- 緑＝セッションが正常終了しただけ。**run のトランスクリプトを開いて**、push 成否・403・「候補ゼロ」等の実際の記述を確認する。

---

## 7. 設計上の重要事項・制約

- **権限の担保**: 「編集させない」の確実な担保は **コミット範囲を `recommend/` に限定**すること。`.claude/settings.json` は補助（`deny` が最優先のため全体拒否は許可フォルダも巻き込む＝サイトの実ファイルを個別に `deny` 列挙する方針）。
- **main 反映**: GitHub Pages の公開ブランチ（通常 main）に届いて初めて表示更新。ルーティンは既定で `claude/*` ブランチを作るため、直接反映には unrestricted branch pushes が必要。
- **情報源の区別**: 元データ／本人申告／Claudeの解釈を常に区別（`profile/README.md` 参照）。
- **利用規約**: dアニメ履歴の自動取得は ToS 未検証。取得は本人判断で。
- **research preview**: Claude Code on web / ルーティンは研究プレビュー。挙動・上限・API は変わり得る。使用量はプランの Claude Code 使用量＋1日実行回数上限を消費。

---

## 8. 参照

- Claude Code on the web: https://code.claude.com/docs/en/claude-code-on-the-web
- ルーティン（スケジュール実行）: https://code.claude.com/docs/en/routines
- クイックスタート（GitHub連携）: https://code.claude.com/docs/en/web-quickstart
- profile のデータ辞書: `profile/README.md`
- 権限の担保方針: `.claude/README_permissions.md`
- 週次プロンプト: `週次タスク_プロンプト_v2.md`
