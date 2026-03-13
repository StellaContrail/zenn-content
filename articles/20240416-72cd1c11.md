---
title: "Flask-RESTX でエラーハンドリングを扱う"
emoji: "🐍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Python","Flask","Flask-RESTX"]
published: true
---

# 概要
Flask-RESTX のエラーハンドリングは、（バグも含めて）かなり複雑な仕様となっている。
本記事では、Flask-RESTX を使って適切にエラーハンドリングをする方法を紹介する。

# 基本的な実装
基本的な Flask-RESTX でのエラーハンドリングの実装方法を以下に示す。
```python:app.py
from flask import Flask
from flask_restx import Resource, Api, abort
from werkzeug.exceptions import HTTPException

app = Flask(__name__)
# Enable custom error handler for 404 and 406.
api = Api(catch_all_404s=True)
api.init_app(app)


@api.route("/error")
class TrivialError(Resource):
    def get(self):
        abort(500, "Trivial Error")
        return {"hello": "world"}


@api.errorhandler(HTTPException)
def httpexception(error):
    # Workaround for #33 issue.
    # ref. https://github.com/python-restx/flask-restx/issues/33#issuecomment-585329009
    message = error.name
    if hasattr(error, "data"):
        message = getattr(error.data, "message", error.name)
    raise error.__class__(message)


@app.errorhandler(HTTPException)
def httpexception(error):
    return {"message": "HTTPException: " + error.description}, error.code


@api.errorhandler(Exception)
def exception(error):
    return {"message": "Internal Server Error"}, 500


if __name__ == "__main__":
    app.run(debug=False)

```

# 注意点
実装にあたって、いくつのか注意点が存在する。

## `catch_all_404s` を有効にする
`catch_all_404s` がデフォルトでは無効になっており、そのままでは 404 Not Found や 406 Not Acceptable エラーが Flask 側で処理されてしまう。カスタムエラーハンドラを使いたい場合は、この設定を有効にする必要がある。

## HTTPException エラーハンドラを２重に定義する
`@api.errorhandler(HTTPException)` がある一方で `@app.errorhandler(HTTPException)` が存在しているのは、レスポンスがカスタムエラーハンドラで生成したものではなく、abort 関数の第２引数で指定したメッセージが直接入ったもので上書きされてしまうからである。

abort 関数で指定したメッセージをそのまま返却するのではなく、カスタムエラーハンドラで加工したものを返却したい場合は、例のように２重で定義する必要がある。
