---
title: "GitHub Container Registry に自作の Docker イメージを公開する"
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

1. **ログインなし** で docker pull ができるようにしたい（公開していいイメージなので、CI に Credential 入れて... とかしたくない）
2. GitHub に commit したり、リリースしたときに自動でレジストリに追加したい
3. 無料でやりたい（マネタイズする見込みは全くないツールなので）

この 2 つの要件を満たす方法は大きく分けて 2 つの方針を考えました。

1. ホスティングは DockerHub のまま、ビルドだけ GitHub Actions などで自動化する
2. ビルドもホスティングも GitHub でやる（ビルドは GitHub Actions, レギストリは GitHub Container Registry へ）

どうせ自分しか使っていないので互換性や DockerHub にこだわる理由もないかなと思い、ここでは 2 を採用しました。GitHub Container Registry は使ったことなかったですし。

## 移行手順

やることはシンプルです。Docker Build と Push をする Actions を作るときに、Push 先を GitHub Container Registry にするだけです。

Docker イメージを push するのに必要なステップは、次の 3 ステップです。

- `docker login`
- `docker build`
- `docker push`

順に追っていきます。

### Login

push するために GitHub Container Registry にログインする必要があります。それには既に Docker 公式で Action が提供されているのでこれをそのまま使います。

[docker/login\-action: GitHub Action to login against a Docker registry](https://github.com/docker/login-action)

今回は GitHub Container Registry を使うので、パスワードは Actions 標準のトークン `secrets.GITHUB_TOKEN` を利用してログインします。特に準備しないといけないものはありません。

```yaml
# Actions の YAML の抜粋
      - uses: actions/checkout@v3
      - name: Login to GitHub Container Registry
        uses: docker/login-action@v2
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
```

### Build

Docker イメージのタグは `レジストリの URL:tag` という形式にする必要があるので、そこだけ注意します。

GitHub Container Registry の URL は `ghcr.io/$GITHUB_USERNAME/$REPOSITORY_NAME` という形式です。Actions ではユーザー名は `$GITHUB_ACTOR` という環境変数で参照できるのでこれを利用します。

```
docker build --tag ghcr.io/$GITHUB_ACTOR/pnovel:latest
```

タグは latest のみでなくコミットごとにレジストリに保存したい場合、`$GITHUB_SHA` を使って push するごとにイメージを保存することもできます。そうしたい場合、ショートハッシュを使ってタグをつけると衝突を（ほとんど）考えずに済みます。

```
docker build --tag ghcr.io/$GITHUB_ACTOR/pnovel:$(echo $GITHUB_SHA | head -c7)
```

```yaml
      - name: Build Docker Image
        run: |
          docker build --tag ghcr.io/$GITHUB_ACTOR/pnovel:latest \
          --tag ghcr.io/$GITHUB_ACTOR/pnovel:$(echo $GITHUB_SHA | head -c7) \
          .
```

### Push

ここまできたら push は簡単です。イメージを push するだけです。

```yaml
      - name: Push Docker Image
        run: |
          docker push ghcr.io/$GITHUB_ACTOR/pnovel:latest
          docker push ghcr.io/$GITHUB_ACTOR/pnovel:$(echo $GITHUB_SHA | head -c7)
```

## YAML 全容

上記の YAML を完全形にすると次のような形になります。

※ もし利用する場合は `pnovel` はお使いのリポジトリ名に変えてください。

```yaml
on:
  push:
    branches:
      - main

name: Docker Build and Push

jobs:
  publish_docker_image:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Login to GitHub Container Registry
        uses: docker/login-action@v2
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Build Docker Image
        run: |
          docker build --tag ghcr.io/$GITHUB_ACTOR/pnovel:latest \
          --tag ghcr.io/$GITHUB_ACTOR/pnovel:$(echo $GITHUB_SHA | head -c7) \
          .
      - name: Push Docker Image
        run: |
          docker push ghcr.io/$GITHUB_ACTOR/pnovel:latest
          docker push ghcr.io/$GITHUB_ACTOR/pnovel:$(echo $GITHUB_SHA | head -c7)
```

## 要件通りになっているかチェック

### ログインなしで使う

GitHub Container Registry はイメージをログインなしで取得できます。ということで、GitLab CI の YAML などでログインなしで取得することができます。つまり、単に下記のように宣言するだけで GitHub Container Registry にあるイメージは使えるわけです。

```yaml
# .gitlab-ci.yml
image: ghcr.io/515hikaru/pnovel:latest
```

試してませんが、たぶん CircleCI など Docker イメージを利用する前提の CI サービスではログインなしで使えると思います。

### コミットごとにイメージを生成

`:latest` や `:$COMMIT_HASH` でタグ付けすることでコミットごとにイメージの生成・保存ができています。

また、今回は紹介していませんが Git の tag をつけて push したときに Docker イメージをビルド & push もできるので、npm に公開したバージョンで Docker イメージを作る、といったこともできます（し、やっています）。

### 無料

パブリックリポジトリなので GitHub Actions も Container Registry も全て無料です。パブリックリポジトリを使っている人はぜひ活用ください。今更か。。

## まとめ、感想

- Docker イメージをログインなしで pull できるように公開するのは DockerHub だけでなく GitHub でもできる
- Actions を使えば多少 YAML を書く必要はあるけれども DockerHub の Automated Build のワークフローをほぼそのまま再現できる
- パブリックリポジトリであれば無料

最初とっつきづらかったけれど、分かってしまえばただの Docker イメージの Registry なのでいつも通り扱えば良いのでおすすめです。

DockerHub の制限は正当なものではあると思うけれど、趣味用途にはちょっと厳しい制限なのもまた事実。個人用途のイメージで、利便性のために公開したいものは GitHub Container Registry を使うのも一考かもしれません。

## 参考

- 公式ドキュメント: [Working with the Container registry \- GitHub Docs](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
