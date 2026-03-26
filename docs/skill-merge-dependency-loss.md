# Skill Branch Merge による依存パッケージ消失問題

## 概要

Feature skill を `git merge` で適用する際、`package.json` / `package-lock.json` のマージ競合を `--theirs` で解決すると、既にインストール済みの別スキルの依存パッケージが消失し、サービスが起動不能になる。

## 発生した事象

### 経緯

1. `skill/discord` が適用済みで `discord.js` が `package.json` に含まれていた
2. `/add-gmail` を実行し、`gmail` リモートから `git merge gmail/main` を実行
3. `package.json` と `package-lock.json` でマージ競合が発生
4. `git checkout --theirs package.json package-lock.json` で競合を解決
5. gmail リモートの `package.json` には `discord.js` が含まれていなかったため、依存定義が消失
6. `npm install` 後もキャッシュに残っていた `discord.js` が `node_modules` にあったためビルドは通った
7. サービス再起動時、`systemctl --user restart nanoclaw` で `nanoclaw` がクラッシュループに入った
8. 直接実行で `ERR_MODULE_NOT_FOUND: Cannot find package 'discord.js'` を確認
9. `npm install discord.js` で復旧

### 根本原因

各 skill ブランチは `main` から分岐しており、他のスキルの依存を含まない。`--theirs` でマージ競合を解決すると、ローカルにのみ存在していた依存（他スキルで追加されたもの）が上書きされて消える。

## 影響範囲

この問題は `package.json` に限らず、複数スキルが同じファイルを変更するケース全般で発生しうる：

| ファイル | リスク |
|---------|--------|
| `package.json` | 他スキルの依存が消える |
| `package-lock.json` | 同上（ロックファイル） |
| `src/channels/index.ts` | 他チャンネルの import が消える |
| `src/container-runner.ts` | 他スキルのマウント設定が消える |
| `container/agent-runner/src/index.ts` | 他スキルの MCP サーバー設定が消える |

## 対策

### マージ競合解決時のルール

`--theirs` や `--ours` による一括解決を避け、競合ファイルを手動でマージする：

```bash
# NG: 一方を丸ごと採用
git checkout --theirs package.json

# OK: 競合箇所を個別に確認・解決
# 1. 競合ファイルを開いて両方の変更を確認
# 2. 両方の依存を含む形で解決
# 3. git add で解決済みとしてマーク
```

### SKILL.md への反映（推奨）

各スキルの SKILL.md のマージ手順を以下のように修正する：

```bash
git fetch <remote> main
git merge <remote>/main || {
  # package-lock.json は theirs で OK（npm install で再生成される）
  git checkout --theirs package-lock.json
  git add package-lock.json
  # package.json は手動マージ — 両方の依存を残す
  # その他の競合ファイルも内容を確認して解決
  git merge --continue
}
```

### マージ後の検証チェックリスト

マージ完了後、以下を確認する：

1. **依存の差分確認**: `git diff HEAD~1 -- package.json` で意図しない依存の削除がないか確認
2. **ビルド**: `npm install && npm run build` でエラーがないか確認
3. **テスト**: `npx vitest run` で既存テストが通るか確認
4. **起動テスト**: `node dist/index.js` で起動できるか確認（即座に Ctrl+C で停止）

### 長期的な改善案

- skill ブランチの SKILL.md に「マージ競合時は `package.json` を手動マージすること」という注意書きを追加する
- skill ブランチを定期的に `main` にマージフォワードし、他のスキルとの互換性を維持する（CI で自動化済み — `docs/skills-as-branches.md` 参照）
- 独立したリモート（`gmail`, `discord` 等）ではなく `upstream` の `skill/*` ブランチからマージすることで、共通の `main` をベースにした競合を減らす

## 参考

- [docs/skills-as-branches.md](skills-as-branches.md) — スキルブランチの設計と運用
- [CONTRIBUTING.md](../CONTRIBUTING.md) — スキルの種類とガイドライン
