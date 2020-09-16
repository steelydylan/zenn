---
title: "種類別、GitHub Actionsの書き方まとめ"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: true
---

GitHub Actionsの使い方でハマることが多いので備忘録にまとめます


## 環境変数をセットしたい

```yml
echo ::set-env name=NAME::name
```

現在の環境変数をstep共通で使いまわしたい場合

```yml
echo ::set-env name=NAME::$(echo NAME)
```

## masterブランチにpushされた時に実行したい

github.refにpushされたbranch名が入っている

```yml
- name update production
  if: github.ref == 'refs/heads/master' && github.event_name == 'push'
  run: ...
```

## releaseという文字列が含まれるPRに対して実行したい

containsを利用することで特定の文字がbranchに含まれるか判定できる
以下のケースでは`release/1.0.0`などのbranchをmasterなどにマージしたい時に利用する

```yml
- name release
  if: contains(github.head_ref, 'release') && github.event_name == 'pull_request'
  run: ...
```

## AWS CLIのセット

```yml
- name: Configure AWS credentials from bot account
  uses: aws-actions/configure-aws-credentials@v1
  with:
    aws-access-key-id: ${{ secrets.ACCESS_KEY }}
    aws-secret-access-key: ${{ secrets.SECRET_ACCESS_KEY }}
    aws-region: ap-northeast-1
```

## package.jsonをもとに自動タグ付け

```yml
- name: package-version-to-git-tag
  uses: pkgdeps/action-package-version-to-git-tag@v1
  with:
    github_token: ${{ secrets.GITHUB_TOKEN }}
    github_repo: ${{ github.repository }}
    git_commit_sha: ${{ github.sha }}
    git_tag_prefix: ""
```