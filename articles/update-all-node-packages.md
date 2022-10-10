---
title: "package.json のバージョンを一気にアップデートする"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

自分用ツールを [npm Registry](https://www.npmjs.com/) に公開しています。

怠惰なのであまりメンテナンスしてないのですが、たまにメンテナンスをするたびに面倒だなと思っていることが依存パッケージのバージョンアップです。セキュリティアップデートなどは [dependabot](https://docs.github.com/ja/code-security/dependabot/dependabot-security-updates/configuring-dependabot-security-updates) などに任せれば良いです。

セキュリティアップデートじゃなくても、下記のコマンドでアップデートができるパッケージのリストアップができます。

```
npm outdated 
```

ということで真面目にやるならコマンドの出力を見て、ひとつひとつアップデートすればよいです。しかし

僕のツールは個人用ツールなので、テストさえ通ればあまり肩肘張らずに依存パッケージのメジャーバージョンアップをしても構いません。

ということで、メジャーバージョンも含めたパッケージのバージョンアップを（手動操作でよいので）サクッとやりたいなと思っていたんですが、そのためのツールを見つけたので紹介します。

### npm-check-updates

npm-check-updates