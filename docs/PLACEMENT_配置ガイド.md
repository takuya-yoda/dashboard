# 配置ガイド（dashboard リポジトリへの配置）

このzipの中身を、GitHub Pages リポジトリ（`~/GitHubPages/dashboard`）の**ルート直下**に展開してください。

```
dashboard/
├── (既存のサイト本体 index.html 等はそのまま)
├── profile/            ← 追加：傾向分析DB（人力・数ヶ月に1回更新／週次は読取のみ）
│   ├── profile.json
│   ├── seen.csv
│   ├── README.md
│   └── archive/        ← 元の詳細資料（来歴保存）
├── recommend/          ← 追加：おすすめ出力（週次で自動更新）
│   ├── recommend.json
│   ├── history.json
│   └── news/2026-07-08.md
└── .claude/
    ├── settings.json          ← 編集権限をprofile/・recommend/に制限
    └── README_permissions.md  ← 権限の担保方針（重要）

※ 既存の temp/ フォルダは archive/ に集約済みなので、確認後に削除して構いません。
```

## 次の一手（Phase 2 の準備）
1. この構成を commit & push（初回のみ手動）。
2. Claude Code on web に dashboard リポジトリを紐付け（push権限トークン設定）。
3. 週次スケジュールタスクを設定（プロンプトは別途用意します）。
4. サイト本体に `recommend/recommend.json` を読んで描画する処理を一度だけ追加。

準備ができたら「Phase 2 に進めて」と言ってください。週次タスクのプロンプトと、サイト側の最小描画スニペットを用意します。
