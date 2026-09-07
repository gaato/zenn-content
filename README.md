# zenn-content

Zenn の記事・本のソース。

## セットアップ

[mise](https://mise.jdx.dev/) で Node.js と zenn-cli を管理している。

```fish
mise trust
mise install
```

## 使い方

```fish
mise run preview               # http://localhost:8000 でプレビュー
mise run new:article           # 記事を新規作成
mise run new:article -- --slug my-article --title "タイトル"
mise run new:book              # 本を新規作成
mise run update                # zenn-cli を最新に更新
```
