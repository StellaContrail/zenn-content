---
title: "Flask-RESTX でレスポンス返却する JSON データで日本語を文字化けさせない"
emoji: "🐍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Python","Flask","Flask-RESTX"]
published: true
---

# 概要
デフォルトで [Flask-RESTX](https://flask-restx.readthedocs.io/en/latest/index.html) は `json.dumps` を ASCII モードでレスポンスを返却してしまう。このため、JSON データにひらがなや漢字が含まれる場合は、内容が文字化けしてしまうという事象が発生する。

# 解決方法
Flask-RESTX の [RESTX_JSON](https://flask-restx.readthedocs.io/en/latest/configuration.html#RESTX_JSON) 環境変数を使って、`json.dumps` 時の引数を上書きする。

```python
app.config["RESTX_JSON"] = {"ensure_ascii": False}
```

- 実装例
```python: app.py
from flask import Flask
from flask_restx import Api, Resource

app = Flask(__name__)
api = Api(app)

app.config["RESTX_JSON"] = {"ensure_ascii": False}

@api.route("/hello")
class Hello(Resource):
    def get(self):
        return {"message": "こんにちは、世界。"}, 200

if __name__ == '__main__':
    app.run(debug=True)
```

- 出力例
```bash
$ curl http://localhost:5000/hello
{
    "message": "こんにちは、世界。"
}
```
