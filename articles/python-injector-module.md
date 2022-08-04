---
title: "Python の DI コンテナ実装の紹介と活用例"
emoji: "💉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Python", "Dependency Injection"]
published: true
---

最近 fukabori.fm という Podcast で DI(Dependency Injection) の話を聞いた。

https://fukabori.fm/episode/48

この Podcast 内では DI に関しては PHP や Java の情報が多い、みたいな話があった。筆者はそれらの言語に疎いのであまり知らないが、Python にも DI コンテナのライブラリ((もしかしたらフレームワークと言った方が正しいかもしれないけど、あまり気にしないことにする。))がある。

ということで、今日はそのライブラリを紹介する。この記事では DI や DI コンテナの概念は既知として、Python で DI コンテナを作る方法と DI コンテナを使う実装例にフォーカスを当てる。

### injector モジュール

https://github.com/alecthomas/injector

PyPI (Pythhon Package Index) に injector という名前のパッケージで登録されている。つまり、下記のように pip でインストールできる。

```
pip install injector
```

Injector というクラスが DI コンテナに相当し、この Injector から DI を施したインスタンスを取得したり、インスタンスの生成方法を指定したりする。

### 基本的な使い方

まずは簡単に使い方を紹介しよう。サンプルコードを提示する。 class A と、 class B を次のように定義する。

```python
from injector import inject

class A:
    @inject
    def __init__(self, number: int, name: str):
        self.number = number
        self.name = name

class B:
    @inject
    def __init__(self, a: A):
        self.a = a
```

injector モジュールには inject デコレーターがあり、このデコレーターで DI を実施するメソッドであるというアノテーションをする。

具体的にはこんな風に使う。

```python
from injector import Injector, InstanceProvider

def configure(binder):
    binder.bind(A, to=InstanceProvider(A(10, 'hikaru')))


def main():
    # 型ヒントの初期値通りに injection
    injector = Injector()
    b = injector.get(B)
    print(b.a.number)  # 0
    print(b.a.name)  # ''

    # 指定した初期値通りに injection
    configured_injector = Injector(configure)
    b = configured_injector.get(B)
    print(b.a.number)  # 10
    print(b.a.name)  # 'hikaru'


if __name__ == '__main__':
    main()
```

Injector クラスが DI コンテナで、このクラスを介してインスタンスを生成すれば DI をしたインスタンスが取得できる。ここでは 明示的には一切 A インスタンスを生成せずに、A インスタンスに依存する B インスタンスを生成した。

特に設定がなければ、型ヒントの初期値通りにインスタンスが生成されるし、指定すればその通りにインスタンスを生成できる。

DI コンテナに生成方法を設定していない、かつ型ヒントも書いていないとエラーになる。

上記のサンプルではインスタンスの生成しかしなかったが、関数やメソッド呼び出しに活用することもできる。例えば class B のメソッドに対して DI をしてみよう。

```python
class B:
    @inject
    def __init__(self, a: A):
        self.a = a

    @inject
    def b(self, keyword: str):
        print(keyword, [self.a.name, self.a.number])
```

`b` というメソッドで print ができる。このとき、 keyword を DI することができて、そのためには `Injector` クラスの `call_with_injection` という API を使う。上記のコードの configure 等を使うと、

```python
configured_injector = Injector(configure)
configured_injector.call_with_injection(configured_injector.get(B).b)
# => ['hikaru', 10]
configured_injector.call_with_injection(configured_injector.get(B).b, args=('foo',))
# => foo ['hikaru', 10]
```

というような形でメソッドの引数に初期値を渡すことができる。

#### 補足

DI に関係はないけど補足。Python のタプルは `()` が作っているように思ってしまいがちだけど、実は `,` が作っている。なので要素ひとつのタプルを作るのに `('foo')` と書いてもタプルにならない。`('foo', )` と末尾にカンマをつける必要があるので注意しよう。

逆に `()` がなくても `,` さえあればカンマとして認識される。多重代入をするときとか、関数の返り値を tuple にするときとかに見たことがあるだろう。

```python
def into_tuple(a, b):
    return a, b

into_tuple(1, 2)  # => (1, 2)
```

### サンプル

閑話休題。DI の話に戻る。

fukabori.fm でも例が挙がったキャッシュを DI するサンプルを作ってみよう。まず、キャッシュの実装を用意する。ここでは Cahce の Interface ICache と `DictCache` という、辞書を Cache としてもつものを定義した（実用性はない）。

```python
from abc import ABCMeta, abstractclassmethod
from typing import Any, Dict

from injector import inject


class ICache(metaclass=ABCMeta):

    @abstractclassmethod
    def get(self, key: str):
        pass

    @abstractclassmethod
    def set(self, key: str, value: Any):
        pass


class DictCache(ICache):

    def __init__(self):
        self.content = {'key': 'value'}
    
    def get(self, key: str) -> Any:
        return self.content.get(key, None)
    
    def set(self, key: str, value: Any) -> None:
        self.content[key] = value
```

次にこの cache を DI するクラスのサンプルを作る。

```python
class FooCase:

    @inject
    def __init__(self, cache: ICache):
        self.cache = cache

    def execute(self):
        v = self.cache.get('foo')
        if v is None:
            v = 'no cache'
        print(v)
```

`FooCase` は cache を受け取り、cache に `'foo'` というキーがあるかを問い合わせ、あればその値を print, なければ 'no cache' という文字列を print する関数である。

このコンストラクターに cache を DI するには、 main 関数で DI コンテナを使う。

```python
def cache_config(binder):
    binder.bind(ICache, DictCache)  # ICache として DictCache を bind

def main():
    injector = Injector(cache_config)
    foo = injector.get(FooCase)
    foo.execute()  # => no cache
```

こうすることで main 関数で明示的に cache を生成することなく、 DI コンテナが DictCache を生成してくれるようになった。

今回は簡単のために `DictCache` という簡単に生成できるものを使ったが、実際には Cache は Redis などアプリケーションの外に持っていることが多い。だが、このコードにおいて DictCache から Redis のキャッシュに変更するのは容易だ。新たに RedisCache というクライアントクラスを(ICache を満たすように)作り、DI コンテナの configuration で `binder.bind(ICache, RedisCache)` とすればほとんどの処理を変更する必要はなく差し替えることができる。インスタンスの生成方法を DI コンテナだけが知っている状況だからこそできる技である。

また、アプリケーションを動作させるときとユニットテストをするときで Cache の実装を使いわけることも簡単であり、自動テストをしやすいコードにもなる。

### 終わりに

Python の DI コンテナの実装である injector モジュールを紹介した。

DI の威力はもちろんのこと、個人的にはインスタンスの生成に型ヒントを活用しているのが気に入っている。型ヒントが機能に寄与しているのは好ましく思う((筆者は [Python の型ヒントは真面目に付き合わない方が良いと思っている](https://blog.515hikaru.net/entry/2020/10/17/165228)けれど、このライブラリを使うときは例外的に型ヒントをつけるモチベーションがあってつけている。))。

他にも今日のサンプルでは使っていない API があるが、[詳細は公式のドキュメントを参照](https://injector.readthedocs.io/en/latest/api.html)して欲しい。

今回利用したコードの全体は下記にある。

https://github.com/515hikaru/junk-code/tree/main/python_injector
