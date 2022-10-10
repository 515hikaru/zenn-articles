---
title: "Google API の OAuth 認可を理解する"
emoji: "👌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

Google の API クライアントライブラリを使う際、OAuth のトークンリクエストを自前実装したくなった。

API を呼び出す際に必要な認証情報は Credentials オブジェクトにまとめられ、この Credentials オブジェクトを API クライアントに渡す。しかし、僕の利用ではトークンリクエストの直後に API を呼び出すわけではない。また必ずしも API クライアントを必要としない OAuth 認可のライブラリの部分のコードを読んでいたのでそれをなんとなく書く。

[googleapis/google-api-python-client: 🐍 The official Python client library for Google's discovery based APIs.](https://github.com/googleapis/google-api-python-client)



## 