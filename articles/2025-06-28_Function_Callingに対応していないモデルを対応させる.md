---
title: "Function Callingに対応していないモデルを対応させる"
emoji: "📝"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["LLM"]
published: true
---

# 経緯
Gemma 3 を使ったシステムでエージェント機能を作りたかったが、Gemma 3 は Function Calling に対応していなかった。公式ページには「関数呼び出し」って書いてあるのに何故…。

> プログラミング インターフェース用にインテリジェントな自然言語コントロールを構築します。Gemma 3 では、特定の構文と制約を使用してコーディング関数を定義できます。モデルはこれらの関数を呼び出してタスクを完了できます。

https://ai.google.dev/gemma/docs/core?hl=ja#function-calling

モデルを変えるという選択肢もあったが、色々あって Gemma 3 のまま対応する方針となった。

ちなみに Ollama が対応している Function Calling モデルは以下の通り。

https://ollama.com/search?c=tools

# 結論
プロンプトエンジニアリング的に解決する。

```plaintext
# Tools
You may call one or more functions to assist with the user query.
You are provided with function signatures within <tools></tools> XML tags:
<tools>

{{- range $.Tools }}
{"type": "function", "function": {{ .Function }}}
{{- end }}
</tools>

For each function call, return a json object with function name and arguments within <tool_call></tool_call> XML tags:
<tool_call>
{"name": <function-name>, "arguments": <args-json-object>}
</tool_call>
```

# 解決手順
Ollama を前提に解説する。vLLM でも Chat template を上書きすることはできるので、同様の手順が適用できるはずである[^1]
[^1]: https://docs.vllm.ai/en/v0.7.0/getting_started/examples/chat.html

`Modelfile` 内に TEMPLATE で以下のチャットテンプレートを定義する。

```dockerfile: Modelfile
TEMPLATE """{{- if .Messages }}
{{- if or .System .Tools }}<start_of_turn>user
{{- if .System}}
{{ .System }}
{{- end }}
{{- if .Tools }}
# Tools
You may call one or more functions to assist with the user query.
You are provided with function signatures within <tools></tools> XML tags:
<tools>

{{- range $.Tools }}
{"type": "function", "function": {{ .Function }}}
{{- end }}
</tools>

For each function call, return a json object with function name and arguments within <tool_call></tool_call> XML tags:
<tool_call>
{"name": <function-name>, "arguments": <args-json-object>}
</tool_call>
{{- end }}<end_of_turn>
{{ end }}
{{- range $i, $_ := .Messages }}
{{- $last := eq (len (slice $.Messages $i)) 1 -}}
{{- if eq .Role "user" }}<start_of_turn>user
{{ .Content }}<end_of_turn>
{{ else if eq .Role "assistant" }}<start_of_turn>model
{{ if .Content }}{{ .Content }}
{{- else if .ToolCalls }}<tool_call>
{{ range .ToolCalls }}{"name": "{{ .Function.Name }}", "arguments": {{ .Function.Arguments}}}
{{ end }}</tool_call>
{{- end }}{{ if not $last }}<end_of_turn>
{{ end }}
{{- else if eq .Role "tool" }}<start_of_turn>user
<tool_response>
{{ .Content }}
</tool_response><end_of_turn>
{{ end }}
{{- if and (ne .Role "assistant") $last }}<start_of_turn>model
{{ end }}
{{- end }}
{{- else }}
{{- if .System }}<start_of_turn>user
{{ .System }}<end_of_turn>
{{ end }}{{ if .Prompt }}<start_of_turn>user
{{ .Prompt }}<end_of_turn>
{{ end }}<start_of_turn>model
{{ end }}{{ .Response }}{{ if .Response }}<end_of_turn>{{ end }}"""
```

:::message alert
プロンプトエンジニアリング的に解決しているので、モデルの推論能力は前提としてある程度確保しておきたい。一応 Gemma 3 27B Q8_0 までは安定して稼働することを確認済み
:::


# 解説
Gemma 3 は公式に Function Calling / Tool Calling (FC) に対応しているわけではない。少なくとも HF 版は。

https://huggingface.co/google/gemma-3-27b-it

だが、プロンプトエンジニアリング的に実現させることは可能である。今回は Ollama を例に Gemma 3 を FC に対応させてみる

## Request to Ollama
まず Ollama に、以下のような Chat Completions w/ Function Calling リクエストを送るとする。

```
curl http://localhost:11434/api/chat -d '{
  "model": "gemma3:27b",
  "messages": [
    {
      "role": "user",
      "content": "What is the weather today in Paris?"
    }
  ],
  "stream": false,
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_current_weather",
        "description": "Get the current weather for a location",
        "parameters": {
          "type": "object",
          "properties": {
            "location": {
              "type": "string",
              "description": "The location to get the weather for, e.g. San Francisco, CA"
            },
            "format": {
              "type": "string",
              "description": "The format to return the weather in, e.g. 'celsius' or 'fahrenheit'",
              "enum": ["celsius", "fahrenheit"]
            }
          },
          "required": ["location", "format"]
        }
      }
    }
  ]
}'
```

LLM はリクエストを直接理解できるわけではなく、LLM が解釈できるのはひと続きの String データとなるため、Chat Template に従って次の文字列に変換される。

```plaintext
<start_of_turn>user
# Tools
You may call one or more functions to assist with the user query.
You are provided with function signatures within <tools></tools> XML tags:
<tools>
{"type": "function", "function": {
  "name": "get_current_weather",
  "description": "Get the current weather for a location",
  "parameters": {
    "type": "object",
    "properties": {
      "location": {
        "type": "string",
        "description": "The location to get the weather for, e.g. San Francisco, CA"
      },
      "format": {
        "type": "string",
        "description": "The format to return the weather in, e.g. 'celsius' or 'fahrenheit'",
        "enum": ["celsius", "fahrenheit"]
      }
    },
    "required": ["location", "format"]
  }
}}
</tools>

For each function call, return a json object with function name and arguments within <tool_call></tool_call> XML tags:
<tool_call>
{"name": <function-name>, "arguments": <args-json-object>}
</tool_call>
<end_of_turn><start_of_turn>user
What is the weather today in Paris?<end_of_turn><start_of_turn>model
```

ここで元のリクエストボディからは次のように文字列へ変換されている

- 全ての `$.Tools` が `"tools"` 配列から展開
- 展開された Tool は `"function"` に `{{ .Function }}}` を値として展開
- その他の部分も Chat Template に従って展開

重要なのはプロンプトで Function Calling のやり方を推論モデルに教えていることである。

```
For each function call, return a json object with function name and arguments within <tool_call></tool_call> XML tags:
<tool_call>
{"name": <function-name>, "arguments": <args-json-object>}
</tool_call>
```
モデルはこのプロンプトを解釈し、Ollama が理解できる形で回答文章を生成する。

## LLM Generation

推論モデルは変換後の文字列を受け取り、件のプロンプトを踏まえた Chat Completions を始める。


処理が終わった推論モデルは、Ollama に次のような文字列を返却する。

```
<tool_call>
{"name": "get_current_weather", "arguments": {"location": "Paris, FR", "format": "celsius"}}
</tool_call>
```

## Ollama Response

Ollama はこの文字列を解釈し、`"tool_calls"` の値として設定、クライアントに返却する。

```
{
  "model": "gemma3:27b",
  "created_at": "2024-07-22T20:33:28.123648Z",
  "message": {
    "role": "assistant",
    "content": "",
    "tool_calls": [
      {
        "function": {
          "name": "get_current_weather",
          "arguments": {
            "format": "celsius",
            "location": "Paris, FR"
          }
        }
      }
    ]
  },
  "done_reason": "stop",
  "done": true,
  "total_duration": 885095291,
  "load_duration": 3753500,
  "prompt_eval_count": 122,
  "prompt_eval_duration": 328493000,
  "eval_count": 33,
  "eval_duration": 552222000
}
```

これが一連の流れであり、プロンプトで制御している以上、**モデルの性能が顕著に Function Calling の安定性に直結する** ことがわかる。

故に、量子化モデルや軽量モデルを使っている場合は、そもそも十分な回答精度が得られるかに注意したほうが良い。

# 実運用上の注意点
- プロンプトで制御しているため、FC の安定性は **モデル性能に強く依存**
- 量子化／軽量モデルでは誤呼び出しや構文エラーが増える可能性がある
- 高精度が必要な場面では、パースロジックの厳格化やリトライ戦略を検討すべき

# 参考文献
https://www.reddit.com/r/LocalLLaMA/comments/1jauy8d/giving_native_tool_calling_to_gemma_3_or_really/
