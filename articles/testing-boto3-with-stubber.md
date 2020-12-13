---
title: "boto3の単体テスト"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Python", "Testing", "boto3"]
published: false
---

# はじめに

普段 Python でサーバーサイド開発をしています。インフラは AWS を利用しており、S3 や Cognito など AWS のリソースを使うには boto3 などを利用します。

もちろん単体テストを書きたいわけですが、自動実行されるテストで実際に AWS のリソースを使うわけにはいかず、mock する必要があります。

当初はナイーブに unittest.mock を使っていたのですが、いろいろと調べると Stubber という boto3 用の単体テストをするためのクラスが提供されていることに気づいたのでその紹介です。

# Stubber クラス

## 正常系の mock

## 異常系の mock

# テストコード例

## 関数やクラスのメソッドの単体テスト

### 実装

### テスト

## API のテスト( Django rest framework)

### 実装

### テスト
