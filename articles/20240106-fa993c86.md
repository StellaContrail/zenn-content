---
title: "LangChainを用いてHuggingFaceモデルをローカルで使う"
emoji: "📝"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["huggingface","LangChain"]
published: true
---

# 概要
HuggingFace Hubに登録されているモデルをローカルにダウンロードして、LangChain経由で対話型のプログラムを作成する。

# 前提条件
ランタイムは Python 3.11.6 を使用する。今回使用するモデルには、比較的汎用的な計算機でも動作する `"TinyLlama/TinyLlama-1.1B-Chat-v1.0"` を採用した

使用する Python パッケージは以下の通り。
| パッケージ名 | バージョン |
| ---- | ---- |
| langchain | 0.0.322 |
| transformers | 4.35.2 |
| huggingface_hub | 0.20.2 |

# 実装コード
実装は以下の通り。

`snapshot_download`関数でリポジトリのスナップショットを撮り、そのスナップショットにあるファイルを元に tokenizer と model を準備する。次に`HuggingFacePipeline`クラスをインスタンス化し、これと`ChatPromptTemplate`でチェーンを作成し、`chain.invoke`メソッドで実行する。
```python: app.py
from langchain.llms.huggingface_pipeline import HuggingFacePipeline
from langchain.prompts import ChatPromptTemplate
from transformers import AutoModelForCausalLM, AutoTokenizer, pipeline
from huggingface_hub import snapshot_download

# 1.HuggingFaceからモデルをダウンロード
model_id = "TinyLlama/TinyLlama-1.1B-Chat-v1.0"
download_path = snapshot_download(repo_id=model_id)

# 2.TokenizerとModelをスナップショットからインスタンス化
tokenizer = AutoTokenizer.from_pretrained(download_path)
model = AutoModelForCausalLM.from_pretrained(download_path)

# 3.HuggingFacePipelineを作成
# ref. https://python.langchain.com/docs/integrations/llms/huggingface_pipelines
pipe = pipeline("text-generation", model=model, tokenizer=tokenizer, max_new_tokens=256)
hf = HuggingFacePipeline(pipeline=pipe)

# 4.ChatPromptTemplateを作成
prompt = ChatPromptTemplate.from_template(
    "<|system|>You are a chatbot who can help code!</s><|user|>{question}</s><|assistant|>"
)
chain = prompt | hf

# 5.Chainの実行
question = "Write me a function to print out 'Hello, World!' to the CLI in Python."
answer = chain.invoke({"question": question})
print(answer)
```

# 実行結果
実行した結果は以下の通り。
スナップショットから `"TinyLlama/TinyLlama-1.1B-Chat-v1.0"` のデータをフェッチし、LangChainを介してモデルとの対話を行えている事がわかる。

````text: bash
$ python app.py
Fetching 10 files: 100%|█████████████████████████████████████████████████████████████| 10/10 [00:00<00:00, 10860.45it/s]

Certainly! Here's a Python function that prints 'Hello, World!' to the CLI:

```python
def print_hello_world():
    print("Hello, World!")
```

To use this function, simply call it with no arguments:

```python
print_hello_world()
```

This will print 'Hello, World!' to the CLI. Note that this function assumes that you have already imported the `print` function from the `sys` module. If you're not using Python 3.6 or later, you may need to import the `print` function explicitly:

```python
from sys import print as print_
```
````

# 補足：GPUによる推論時間の短縮
GPUが利用できるのであれば、引数に`device=0`を指定することで、計算時間をかなり短縮できる。
```python: app.py (抜粋)
pipe = pipeline("text-generation", model=model, tokenizer=tokenizer, max_new_tokens=256, device=0)
```

以下は i7-13700F + RTX 4070 の構成で、10回推論を行ったときの計算時間の平均である。
※ かなり雑な計測なので、実際に手元で試してほしい
| 計測ケース | 計測時間（秒） |
| ---- | ---- |
| CPU のみ | 24.3  |
| GPU サポート | 2.3 |

# 参考文献
https://huggingface.co/TinyLlama/TinyLlama-1.1B-Chat-v1.0/tree/main

https://python.langchain.com/docs/integrations/llms/huggingface_pipelines

https://qiita.com/suzuki_sh/items/0b43ca942b7294fac16a
