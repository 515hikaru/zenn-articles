---
title: "GitHub の Container Registry に自作の Docker イメージを公開する"
emoji: "⛳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Docker", "GitHub Actions", "GitHub Packages"]
published: false
---

Docker 社が、DockerHub の機能である Automated Builds を有料プランのみに提供するように変更しました。投稿日の 1 年ほど前、2021 年の 6 月のことです。

[Changes to Docker Hub Autobuilds - Docker](https://www.docker.com/blog/changes-to-docker-hub-autobuilds/)

これを受けてずっと移行しないといけないなぁと思っていましたが、重い腰をあげたのがつい最近なので備忘録的に何をしたのかを書いておきます。

## 移行方針

前提として、僕はオープンソースで自分用のツール [pnovel](https://github.com/515hikaru/pnovel) を開発しており、このツールをインストールした Docker イメージを公開しています。目的は、CircleCI や GitLab CI などのツールで使いやすくするためです。

そのため、要件としては下記の 2 点が挙げられます。

1. **ログインなし** で docker pull ができるようにしたい（公開していいイメージなので、CI に Credential 入れて... とか面倒）
2. GitHub に commit したり、リリースしたときに自動でレジストリに追加したい
3. 無料でやりたい（マネタイズする見込みは全くないツールなので）

この 2 つの要件を満たす方法は大きく分けて 2 つの方針を考えました。

1. ホスティングは DockerHub のまま、ビルドだけ GitHub Actions などで自動化する
2. ビルドもホスティングも GitHub でやる（ビルドは GitHub Actions, レギストリは GitHub Container Registry へ）

どうせ自分しか使っていないので DockerHub にこだわる理由もないかなと思い、ここでは 2 を採用しました。GitHub Container Registry は使ったことなかったですし。

## 移行手順

やることはシンプルです。Docker Build と Push をする Actions を作るときに、Push 先を GitHub Container Registry にするだけです。


