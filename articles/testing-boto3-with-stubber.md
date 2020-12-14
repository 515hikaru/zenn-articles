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

こんな風に使います。

```python
>>> import boto3
>>> from botocore.stub import Stubber
>>> client = boto3.client('cognito-idp')
>>> stubber = Stubber(client)
>>> stubber.add_response('list_users', {'Users': []})
>>> stubber.add_client_error('admin_get_user', service_error_code='UserNotFoundException')
>>> stubber.activate()
>>> client.list_users(UserPoolId='dummpy_id')
{'Users': []}
>>> client.admin_get_user(UserPoolId='dummpy_id', Username='user@example.com')
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/home/hikaru/.local/lib/python3.8/site-packages/botocore/client.py", line 357, in _api_call
    return self._make_api_call(operation_name, kwargs)
  File "/home/hikaru/.local/lib/python3.8/site-packages/botocore/client.py", line 676, in _make_api_call
    raise error_class(parsed_response, operation_name)
botocore.errorfactory.UserNotFoundException: An error occurred (UserNotFoundException) when calling the AdminGetUser operation:
```

順に見ていきます。

## 初期化

どのサービスのクライアントを作るかを指定して、通常通りクライアントを作ります（この例では Cognito のクライアントを作っています）。そのクライアントを Stubber クラスのコンストラクタに渡します。

```python
>>> client = boto3.client('cognito-idp')
>>> stubber = Stubber(client)
```

## 正常系の mock

正常系の mock は Stubber クラスの `add_response` メソッドで行います。第一引数がメソッド名（文字列）、第二引数がメソッドの返す値ですね。ここでは `Users` が空の JSON を返すように定義しています。

```python
>>> stubber.add_response('list_users', {'Users': []})
```

## 異常系の mock

異常系も mock できます。異常系は `add_client_error` メソッドで、第一引数は正常系と同様にメソッド名です。第二引数が `service_error_code` で、 ClientError のエラーコードに相当する情報です。

`admin_get_user` メソッドはユーザーが見つからなかったときに、エラーコードが `UserNotFoundException` の ClientError を raise します。下記のように書くことで、その状況を疑似的に作ることができます。

```python
>>> stubber.add_client_error('admin_get_user', service_error_code='UserNotFoundException')
```

エラーメッセ―ジも指定できますが、ここでは指定していません。

## Stub の有効化

`activate` メソッドを呼ぶか、`with` 句を作ればいいです。

```
stubber.activate()
client.list_users()

# or

with stubber:
    client.list_users()
```

より詳細はドキュメントを参照してください。

[Stubber Reference — botocore 1\.19\.35 documentation](https://botocore.amazonaws.com/v1/documentation/api/latest/reference/stubber.html)

# テストコード例

上記の内容はドキュメントにもほぼそのまま書いてあるようなことです[^1]が、この情報から実際に単体テストのコードを起こしてみます。

[^1]: 例が S3 で Cognito ではないですが。

## メソッドの単体テスト


### 実装

### テスト

## API のテスト( Django rest framework)

### 実装

### テスト
