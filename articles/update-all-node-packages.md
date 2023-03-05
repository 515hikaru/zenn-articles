---
title: "自作コマンドラインツールの依存関係を時間かけずにメンテナンスし続けるためにしていること"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["npm", "github"]
published: true
---

自分用のコマンドラインツールを[npm Registry](https://www.npmjs.com/)に公開しています。

僕が作っているツールは本当に個人用途のツールなので、機能追加やバグ修正もあまり頻繁に行っていません。

しかし、Node.jsを使っている都合どうしても依存パッケージがあります。そのため、自作ツールの機能追加の頻度よりもパッケージをアップデートする頻度のほうが高くなります。

当初は放置していたんですが、セキュリティパッチをあてないでいるとGitHubからメール通知が来たり、そもそも（現実的に脅威となり得るかはさておいても）脆弱性がある状態で公開し続けるのもなんか無責任な気がします。いくら自分用ツールとはいえ公開する以上は最低限の責任は果たしたほうがよさそうだと思いました。

とはいえそんなに長い時間かけてもいられません。プログラマーは怠惰なのです。というわけでそんなに時間をかけずにパッケージのアップデートをするフローを考えて実現してみることにしました。

特にdependabotを取り入れることにより、僕は週に15分もかけることなくパッケージの更新を継続しています。dependabotの導入は

## dependabotの導入

自作ツールのコードはGitHubにホストしています。ということで、最初に考えるのはdependabotの導入でしょう。

[Dependabot を使う - GitHub Docs](https://docs.github.com/ja/code-security/dependabot/working-with-dependabot)

毎日来るとさすがに大変なので、週に1回まとめてくるように設定します。

```yml
# .github/dependabot.yml 
version: 2
updates:
  - package-ecosystem: npm
    directory: '/'
    schedule:
      interval: weekly
      time: '00:00'
    open-pull-requests-limit: 10
    reviewers:
      - 515hikaru
    assignees:
      - 515hikaru
```

これで月曜日の朝9時ごろ（UTCで月曜日の午前0時ごろ）に毎週PRが届くようになります。あとはGitHub Actionsでテストを動かしているので、それが通っていたらマージする、くらいの操作です。

これであれば通勤時間にスマホからでもパッケージのアップデートが安全にできます。これだけでも、パソコンを取り出して npm install してテストを動かして......とかするのに比べればだいぶ楽になりますね。

### 自動マージ

patchバージョンの変更でかつテストが通れば、どのような変更であってもマージしてしまってほとんどの場合問題ありません。

ということで、dependabotの自動マージも設定しています。これは[GitHub公式に詳しくやり方が書いてあるのでこちらを参照ください](https://docs.github.com/ja/code-security/dependabot/working-with-dependabot/automating-dependabot-with-github-actions)。

### Actions自体のメンテナンス

結構、単体テストなどでGitHub Actionsに利用するActionsのメンテナンスが盲点です。.github/workflows配下のYAMLに書く、

```yaml
- uses: actions/checkout@v3
```

みたいなやつです。`@v3` とあるように、これもバージョン管理がなされています。ということは新しいバージョンにするメンテナンス作業が必要です。

実はdependabotはActionの依存関係も管理できます。dependabot.ymlの `package-ecosystem` の値をgithub-actionsにするだけです。

Actionsを使って単体テストなどさまざまなジョブを実行している場合、設定を必ずしたほうがいいです。

参考: [Configuration options for the dependabot.yml file - GitHub Docs](https://docs.github.com/ja/code-security/dependabot/dependabot-version-updates/configuration-options-for-the-dependabot.yml-file#package-ecosystem)

## セキュリティ以外のアップデート

セキュリティアップデートではないパッケージのバージョンも月に1回程度アップデートしています。

真面目にやるなら、`npm outdated` コマンドでアップデートができるパッケージのリストアップをしてひとつひとつアップデートしていくのが正攻法でしょう。

しかし何度も繰り返しているように、今回の目的はなるべく楽をすることです。僕のツールは個人用ツールなので、テストさえ通ればあまり肩肘張らずに依存パッケージのメジャーバージョンアップさえやっても構いません。

ということで、メジャーバージョンも含めたパッケージのバージョンアップを（手動操作でよいので）サクッとやる方法を探したところ `npm-check-updates` コマンドを見つけたので紹介します。

### npm-check-updates

[raineorshine/npm-check-updates: Find newer versions of package dependencies than what your package.json allows](https://github.com/raineorshine/npm-check-updates)

使い方はとても簡単です。まず自分のプロジェクトディレクトリ（package.jsonがありところ）で

```
npx ncu
```

を呼び出します。すると、下記のような感じでアップデートできるパッケージのリストが出てきます（[以下のリストは上記リポジトリのREADMEから引用](https://github.com/raineorshine/npm-check-updates/blob/e50fff0e777fc8fb2d2b55652dd74effaaa276ed/README.md)）。

```
Checking package.json
[====================] 5/5 100%

 eslint             7.32.0  →    8.0.0
 prettier           ^2.7.1  →   ^3.0.0
 svelte            ^3.48.0  →  ^3.51.0
 typescript         >3.0.0  →   >4.0.0
 untildify          <4.0.0  →   ^4.0.0
 webpack               4.x  →      5.x
```

このバージョンアップに問題がなければ、`npx ncu -u` とうつだけです。これでpackage.jsonの更新が終わります。簡単でしょ？

## 終わりに

以上、パッケージ管理に関する工夫をざっくり書いてみました。

上記のような自動化や、パッケージバージョンの大雑把なアップデートを安全に実行するには前提として自動テストが整備されていることが重要です。ある程度自動的に動作することを保証し続けられることで、初めて安全に自動化ができます。 

ということで、個人用のツールだしとサボらずテストを書いてみるのもいかがでしょうか。テストを書くのはセキュリティ向上やメンテナンスの自動化にも繋がります。

## 参考

- 実際に設定しているリポジトリ [515hikaru/pnovel: 日本語小説用簡易マークアップ言語](https://github.com/515hikaru/pnovel)
