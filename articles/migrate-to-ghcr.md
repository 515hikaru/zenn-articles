---
title: "GitHub の Container Registry に自作の Docker イメージを公開する"
emoji: "⛳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Docker", "GitHub Actions", "GitHub Packages"]
published: false
---

DockerHub が 1 年ほど前に Automated Builds を有料プランに移行しました。

[Changes to Docker Hub Autobuilds - Docker](https://www.docker.com/blog/changes-to-docker-hub-autobuilds/)

これを受けてずっと移行しないといけないなぁと思っていましたが、重い腰をあげたのがつい最近なので備忘録的に何をしたのかを書いておきます。

