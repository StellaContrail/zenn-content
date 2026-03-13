---
title: "safetensors形式のモデルをOllamaで立ち上げる"
emoji: "📝"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["ollama"]
published: true
---

# 経緯
チューニングした safetensors モデル（Gemma 3）を Ollama で立ち上げる必要があった。
独自にチューニングしたモデルのため、Ollama のレジストリには当然無く、自前でモデルをビルドする必要があった。

# 結論
llama.cpp で GGUF 形式に変換した後、Ollama でモデルをビルドし、モデルを立ち上げる。

モデルアーキテクチャによっては safetensors から直接できるものもあるが、一般的には GGUF 形式に一度変換する必要がある。

手順は以下の通り。
1. モデルデータを safetensors から GGUF に変換する
1. GGUF 形式のモデルデータを Ollama 形式にビルドする
1. Ollama で立ち上げる


# はじめに
ローカルにあるモデルデータを Ollama で動かすには、モデルを Ollama でビルドしなければならない。

ビルド元のモデル形式は safetensors と GGUF に対応しているが、safetensors の場合は下記のモデルアーキテクチャにしか対応していないため注意である。
> - Llama (including Llama 2, Llama 3, Llama 3.1, and Llama 3.2)
> - Mistral (including Mistral 1, Mistral 2, and Mixtral)
> - Gemma (including Gemma 1 and Gemma 2)
> - Phi3

https://github.com/ollama/ollama/blob/main/docs/modelfile.md#build-from-a-safetensors-model

今回は Gemma 3 で非対応だったため、まずは safetensors から GGUF に変換する必要があった。

# safetensors形式のモデルをOllamaで立ち上げる
今回は変換した後に Ollama でビルド＆モデルを立ち上げる手順を紹介する。

:::message alert
変換前後で Vision が使えなくなる可能性あり。Gemma 3 12B の safetensors から GGUF に変換したときは Vision が使えなくなってしまった
:::

## 全体像
safetensors 形式の Gemma 3 モデルデータから Ollama で立ち上げるまでの全体像を紹介する。

1. モデルデータを safetensors から GGUF に変換する
1. GGUF 形式のモデルデータを Ollama 形式にビルドする
1. Ollama で立ち上げる

## モデルデータを safetensors から GGUF に変換する
https://github.com/ggml-org/llama.cpp/discussions/2948

safetensors 形式の Gemma 3 モデルデータに Ollama が対応していないため、まずは GGUF に変換する。

llama.cpp をダウンロードする
```bash
$ git clone git@github.com:ggml-org/llama.cpp.git
$ cd llama.cpp
```

モデルデータを変換する（LoRA Adapter は `convert_lora_to_gguf.py` という別ファイルがありそう）
```bash
$ pip install -r requirements.txt
$ python3 convert_hf_to_gguf.py <model directory>
```

GGUF 形式のモデルデータは指定したモデルディレクトリの中に出力される


## GGUF 形式のモデルデータを Ollama 形式にビルドする
https://github.com/ollama/ollama?tab=readme-ov-file#import-from-gguf

`Modelfile` を作成してモデルを定義する。

```bash
$ vim Modelfile
```

```dockerfile: Modelfile
FROM <GGUF model path>
```

`Modelfile` ができたら、Ollama でモデルをビルドする
```bash
$ ollama create <model name> -f ./Modelfile
```

Ollama でモデルを起動する
```bash
$ ollama run <model name>
```

完了。

# 余談
safetensors 形式であれば、公式に読み込みをサポートしている vLLM で立ち上げたほうが良いかもしれない。

https://docs.vllm.ai/en/v0.4.0.post1/models/engine_args.html#cmdoption-load-format

一方で、vLLM は先に GPU リソースを予約して KV Cache を詰め込むといった作りになっているため、複数モデルが立ち上がるシナリオでのリソースの管理が難しい。
