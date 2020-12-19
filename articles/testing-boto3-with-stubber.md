---
title: "Stubber クラスを用いた boto3 を使うメソッドのテストコード実装例"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Python", "Testing", "boto3"]
published: true
---

# はじめに

普段 Python でサーバーサイド開発をしています。インフラは AWS を利用しており、AWS のリソースを使うには boto3 などを利用します。

もちろん単体テストを書きたいわけですが、自動実行されるテストで実際に AWS のリソースを作ったり削除したりするわけにはいかないので、mock する必要があります。

当初はナイーブに unittest.mock を使っていたのですが、いろいろと調べると botocore モジュールには Stubber というクラスが提供されていることに気づいたので、その紹介をします。

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

[^1]: 公式ドキュメントの例は S3 で、 Cognito のものではないですが。

## メソッドの単体テスト

ここでは `admin_get_user` を使った関数をテストするテストコードを書きます。

記事内にもコードは書きますが、リポジトリはここにあります: https://github.com/515hikaru-sandbox/example-unittest-with-boto3/

### テスト対象の関数のサンプル

テスト対象の関数の実装は次のような内容にします。

```python
# main.py
import os

import boto3
from botocore.exceptions import ClientError

os.environ['AWS_ACCESS_KEY_ID'] = 'DUMMY_VALUE'
os.environ['AWS_SECRET_ACCESS_KEY'] = 'DUMMY_VALUE'
os.environ['AWS_DEFAULT_REGION'] = 'ap-northeast-1'

def get_user(name: str) -> dict:
    client = boto3.client('cognito-idp')

    try:
        user = client.admin_get_user(
            UserPoolId='DUMMY_USER_POOL_ID',
            Username=name,
        )
    except ClientError as error:
        if error.response['Error']['Code'] == 'UserNotFoundException':
            return None
        raise

    return user['UserAttributes']
```

環境変数を設定しているのはサンプルを動作させる都合で仕方なく、です[^2]。実際には環境変数はアプリケーションコードの外で管理しましょう。

[^2]: 未設定だと boto3 のクライアントのコンストラクタでエラーが出てしまうので書いているだけです。

`Username` をもとにユーザーを取得し、返り値の `UserAttributes` を返すコードです。`Username` を持つユーザーが存在しなかった場合は `None` を返却するようにしています。

### テスト

テストですが、大きく 2 つ実装したくなるでしょう。

1. 正常系（UserAttributes を正しく取得できるか）
2. 異常系（UserNotFoundException が出たときに None が返ってくるか）

これらをテストするために、`boto3.client` を Stubber クラスを使ってモックします。

#### 正常系テスト

ここでは標準ライブラリにある unittest を使います。

```python
# tests/test_main.py
import unittest
from unittest import mock
import boto3
from botocore.stub import Stubber

from main import get_user

class TestGetUser(unittest.TestCase):

    def test_get_user(self):
        client = boto3.client('cognito-idp')
        stubber = Stubber(client)
        stubber.add_response('admin_get_user', {
            'Username': 'user',
            'UserAttributes':
                [
                    {'Name': 'sub', 'Value': 'aa45403e-8ba5-42ab-ab27-78a6e9335b23'},
                    {'Name': 'email', 'Value': 'user@example.com'}
                ]
        })
        stubber.activate()
        with mock.patch('boto3.client', mock.MagicMock(return_value=client)):
            user = get_user('user')

        self.assertEqual(user,
                [
                    {'Name': 'sub', 'Value': 'aa45403e-8ba5-42ab-ab27-78a6e9335b23'},
                    {'Name': 'email', 'Value': 'user@example.com'}
                ]
        )
```

ポイントは `with mock.patch('boto3.client', mock.MagicMock(return_value=client)):` です。

先述した通り、 Stubber クラスにクライアントを渡します。ただ、Stubber を activate しただけでは `get_user` 関数の中で新たなクライアントを作成してしまい、モックしたクライアントを使ってくれません。そのため `get_user` 内にある `boto3.client` の返り値を、モックしたクライアントに差し替える必要があります。それを行っているのがこの with ブロックになります。

あとは通常の Python のユニットテストと同じです。返り値（`user`）が意図通りかを確認します。

#### 異常系テスト

異常系も正常系とほぼ同様です。異常系については import 文やクラス定義は省略します（同じ内容です）。

```python
def test_not_found_user(self):
    client = boto3.client('cognito-idp')
    stubber = Stubber(client)
    stubber.add_client_error('admin_get_user', 'UserNotFoundException')
    stubber.activate()
    with mock.patch('boto3.client', mock.MagicMock(return_value=client)):
        user = get_user('user')

    self.assertIsNone(user)
```

`add_response` ではなく `add_client_error` を使うことで意図したクライアントエラーを疑似的に起こすことができます。`get_user` の中で `UserNotFoundException` が起きているので `user` が期待通り None になっています。

# 終わりに

boto3 のクライアントをモックし、自動テストを書く方法についてまとめました。特に公式ドキュメント等でも `unittest.mock` とどう組み合わせて使うのかなどのサンプルがないので、実際にテストコードを書いたときに試行錯誤をしたのでその結果をここに書いています。

boto3 のクライアントの返り値は情報量も多く、非常にモックしづらいです。しかし、実際にアプリケーションとして扱う値、テストに関係する値はごく少数のはずです。適切にモックをすることで、結合テストをするまでもなく、アプリケーションの実装ミスや問題を発見することができます。

もちろん最終的には結合テストも実施する必要がありますが、適切な単体テストを書いておけば、動作が保証されているアプリケーションのコードが増えるので、工程も多少は省力化できるはずです。

単体テストを書きましょう、自動テストを実施しましょうというだけでなく、正しく継続的に動く、そして動作保証の範囲を広くするテストコードを書いていきたいですね。
