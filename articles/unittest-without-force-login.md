---
title: "認証にトークンを利用する API の単体テストを書く"
emoji: "✨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Django", "Python", "Test"]
published: true
---

この記事は [Django Advent Calendar 2022 - Qiita](https://qiita.com/advent-calendar/2022/django) の 12/18 の記事です。

## 前提

- Django REST Framework を利用していることを前提とします
- JWT で認証する API のことを書きますが、JWT についてや JWT をどのように作るかについては触れません

## JWT で認証する API を作成する

JWT トークンを認証に利用する Web API のエンドポイントを書いた。

このアプリは Django を利用しているが、 `django.contrib.auth` のユーザーモデルを利用していない。そのため、ローカルで立ち上げて動かす分には JWT を Authorization ヘッダーに載せてしまえばよい。こんなふうに。

```sh
curl -H 'Authorization: Bearer ${JWT_TOKEN}' https://example.com/api/foobar
```

認証クラスは自作した、JWT をデコードして一意のユーザーID(uuid形式)を取得し、DB にユーザーが存在することを問い合わせるものである。

```python
from rest_framework.authentication import BaseAuthentication
from .models import User  # NOTE: django.contrib.auth.models.User **ではない**
class CustomAuthentication(BaseAuthentication):
    def authenticate(self, request):
        # JWT の扱いはあまり書きたいことと関係ないので省略する
        jwt_string = ...  
        payload = ... # JWT の payload を取りだす
        user_id = payload["sub"]
        try:
            user = User.objects.get(id=user_id)
        except User.DoesNotExists:
            return None

        return (user, None)
```

このクラスを view クラスの `authentication_classes` に指定した。そして無事、上記の curl コマンドで作った HTTP リクエストが動いた。

## 単体テストを書く

しかし、単体テストのときに困った。自分で定義したユーザーのため、force_login が使えないのだ。

```python
from django.test import TestCase, Client
from .models import User

class TestUseApi(TestCase):

    def setUp(self):
        self.u = ... # テスト用ユーザー準備

    def test_get(self):
        c = Client()
        c.force_login(self.u)  # auth_user のモデルを使っていないので動かない
        res = c.get('/api/foobar')
        self.assertEqual(res.status_code ,200)
```

困ったが、対処方法は別に難しくない。要するに token がないことが問題なので token を渡すようにカスタマイズすればよい。

Client オブジェクトに `login_with_token` メソッドを定義しその場で生成したトークンをヘッダーに載せる。

```python
from django.test import TestCase, Client

# test.py の Client をカスタマイズする
class CustomClient(Client):
    def __init__(self, enforce_csrf_checks=False, raise_request_exception=True, **defaults):
        self.user_id = None
        super().__init__(enforce_csrf_checks, raise_request_exception, **defaults)

    def login_with_token(self, user_id: str):
        self.user_id = user_id

    def request(self, **request):

        if self.user_id:
            request.update(
                {
                    "HTTP_AUTHORIZATION": f'Bearer {generate_jwt(self.user_id)}',  # ここで JWT トークンを生成する
                }
            )
        return super().request(**request)

class TestUseApi(TestCase):

    def setUp(self):
        self.u = ... # テスト用ユーザー準備

    def test_get(self):
        c = CustomClient()
        c.login_with_token(self.u.id)  # トークンを生成するユーザーの id を指定
        res = c.get('/api/foobar')
        self.assertEqual(res.status_code ,200)

        j = res.json()
        self.assertEqual(len(j), 1)
```

これでテストができた。

## まとめ

単体テストでも HTTP の Authorization ヘッダーにトークンを載せたいときは、 Client オブジェクトをカスタマイズして request を送る前にトークンをヘッダーに載せればいい。
