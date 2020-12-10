---
title: "boto3の単体テスト"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Python", "Testing", "boto3"]
published: false
---

# boto3 のテスト

- 正常系
    - クライアントを mock する
    - レスポンスを mock する
- 異常系
    - ClientError でエラーハンドリング
    - 例外を mock で起こす
- API を叩くテストをする w/restframework
    - mock.patch と with でよしなにこうやっていく
