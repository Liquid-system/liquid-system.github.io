---
title: "hugo導入メモ"
date: 2023-02-27T22:22:32+09:00
tags: "tech"
---

このサイトは[hugo](https://github.com/gohugoio/hugo)と GitHub Pages を使って運用している。

### 詰まった点

- workflows のパーミッション設定
  - gh-pages ブランチが作られるはずが write 権限がオフになっていた為アクセスできなかった。参考 [Permission denied to github-actions[bot]. The requested URL returned error: 403](https://stackoverflow.com/questions/73687176/permission-denied-to-github-actionsbot-the-requested-url-returned-error-403)
- icon が変更できない。
  - プレビュー段階では初期アイコンのままらしい。デプロイしたら変更できた。


