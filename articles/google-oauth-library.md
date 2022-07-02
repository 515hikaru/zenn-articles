---
title: "Google API の OAuth 認可を理解する"
emoji: "👌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

Google (Workspace など) の API クライアントライブラリを使う際、OAuth 認可の部分を諸般の事情で自前実装したくなった。そのため、OAuth 認可のライブラリの部分のコードを読んでいたのでそれをなんとなく書く。

[googleapis/google-api-python-client: 🐍 The official Python client library for Google's discovery based APIs.](https://github.com/googleapis/google-api-python-client)

## API をコールするために必要なもの

## Credential オブジェクトの生成方法

### expiry の計算

## Credential オブジェクトを自前で作る

### トークンリクエスト

### ストアしたデータから Credential オブジェクトを作る

## 実際に API を呼ぶ

## 終わりに
