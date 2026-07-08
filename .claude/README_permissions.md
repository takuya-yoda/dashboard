# 編集権限の担保について

目的: Cowork/自動化に **`profile/` と `recommend/` 以外を編集させない**。

## 二重の担保
1. **一次担保（確実）= スコープ付きコミット**
   週次自動タスクは変更を `recommend/` のみステージングして push する:
   ```
   git add recommend/
   git commit -m "chore: weekly recommend update"
   git push -u origin <branch>
   ```
   たとえ他ファイルを誤編集しても、ステージされないため公開サイトへは反映されない。これが最も確実な防御。

2. **二次担保（best-effort）= `.claude/settings.json`**
   `deny` が最優先。`allow` は `profile/**` `recommend/**` を許可。
   注意: `deny(**)` のような全体拒否は許可フォルダも巻き込むため使わない。代わりに **サイト本体の実ファイル/ディレクトリを `deny` に明示列挙**する方式にしている。
   → リポジトリの実構成に合わせ、`deny` にサイトのファイル/フォルダ（例: 実際の `index.html`, CSS/JS/画像ディレクトリ, `_config.yml` 等）を追記して調整すること。
