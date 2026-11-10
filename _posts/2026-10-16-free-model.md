---
title: 免费的ai模型
date: 2026-10-16 08:00 +0800
categories: [AI]
tags: [免费的大模型]
---





### 模型的大模型

### modelscope
> 阿里云的魔塔
1. 注册官网 [地址](https://modelscope.cn/)
2. 准备一个邮箱即可以
3. 登录申请apikey
4. 调用模型例子


```python
import os
from openai import OpenAI

client = OpenAI(
  base_url = "https://api-inference.modelscope.cn/v1",
  api_key = os.environ["apiKey"]
)


completion = client.chat.completions.create(
  model="deepseek-ai/DeepSeek-V4.1-Flash-1017",
  messages=[{"role": "user", "content": [{"type": "text", "text": "Read the traffic sign in this image. What does it say, and what color is the sign?"}, {"type": "image_url", "image_url": {"url": "https://assets.ngc.nvidia.com/products/api-catalog/phi-3-5-vision/example1a.jpg"}}]}],
  temperature=1,
  top_p=0.95,
  max_tokens=16384,

  parallel_tool_calls=True,



  seed=42,
  extra_body={"chat_template_kwargs": {"enable_thinking": True}},
  stream=True
)

for chunk in completion:
  if not chunk.choices:
    continue
  reasoning = getattr(chunk.choices[0].delta, "reasoning_content", None)
  if reasoning:
    print(reasoning, end="")
  if chunk.choices[0].delta.content is not None:
    print(chunk.choices[0].delta.content, end="")

```




### AMD
> AMD 免费调用模型有 deepseek 以及部分其他的模型，支持api调用，同时支持免费的cloud部署测试模型，需要领amd的算力卷，可以获取200个小时免费额度

1. 注册官网 [地址](https://developer.amd.com.cn/)
2. 准备一个邮箱即可以
3. 登录申请apikey
4. 调用模型例子

```python
import os
from openai import OpenAI

client = OpenAI(
  base_url = "https://radeon-global.anruicloud.com/api/v1",
  api_key = os.environ["apiKey"]
)


completion = client.chat.completions.create(
  model="DeepSeek-V4.1-Flash",
  messages=[{"role": "user", "content": [{"type": "text", "text": "Read the traffic sign in this image. What does it say, and what color is the sign?"}, {"type": "image_url", "image_url": {"url": "https://assets.ngc.nvidia.com/products/api-catalog/phi-3-5-vision/example1a.jpg"}}]}],
  temperature=1,
  top_p=0.95,
  max_tokens=16384,

  parallel_tool_calls=True,



  seed=42,
  extra_body={"chat_template_kwargs": {"enable_thinking": True}},
  stream=True
)

for chunk in completion:
  if not chunk.choices:
    continue
  reasoning = getattr(chunk.choices[0].delta, "reasoning_content", None)
  if reasoning:
    print(reasoning, end="")
  if chunk.choices[0].delta.content is not None:
    print(chunk.choices[0].delta.content, end="")

```



### Nvidia
> 里面有很多免费的模型，但是有不少会出现429请求问题，而且很多都是特定领域模型，不是多模态的模型，需要自己去看的调用方式，有的是sse,有的是http,不过平台有调用例子


1. 注册官网 [地址](https://build.nvidia.com/)
2. 准备一个邮箱即可以
3. 登录申请apikey
4. 调用模型例子

```python
import os
from openai import OpenAI

client = OpenAI(
  base_url = "https://apihub.agnes-ai.com/v1",
  api_key = os.environ["apiKey"]
)


completion = client.chat.completions.create(
  model="nvidia/nemotron-3.5-super-vl-120b-a12b",
  messages=[{"role": "user", "content": [{"type": "text", "text": "Read the traffic sign in this image. What does it say, and what color is the sign?"}, {"type": "image_url", "image_url": {"url": "https://assets.ngc.nvidia.com/products/api-catalog/phi-3-5-vision/example1a.jpg"}}]}],
  temperature=1,
  top_p=0.95,
  max_tokens=16384,

  parallel_tool_calls=True,



  seed=42,
  extra_body={"chat_template_kwargs": {"enable_thinking": True}},
  stream=True
)

for chunk in completion:
  if not chunk.choices:
    continue
  reasoning = getattr(chunk.choices[0].delta, "reasoning_content", None)
  if reasoning:
    print(reasoning, end="")
  if chunk.choices[0].delta.content is not None:
    print(chunk.choices[0].delta.content, end="")

```
5. **免费模型列表**
  - . **nemotron-3-ultra-550b-a55b**｜总 550B，激活 55B MoE，1M 上下文，Agent、代码、工具调用、长文本推理，可下载 NIM 容器NVIDIA
  - . **nemotron-3-super-120b-a12b**｜总 120B，激活 12B，1M 上下文，多语言（含中文），智能体、RAG、代码
  - . **nemotron-3.5-lightning-30b-a3b**｜30B MoE，轻量高速 Agent 模型
  - . **nemotron-3-nano-4b**｜4B 小参，适合 RTX PC 本地跑智能体，轻量对话
  - . **nemotron-3-embed-1b**｜Embedding 向量模型，RAG 检索、语义向量（免费端点）NVIDIA

  - **多模态 VLM（图文）**

    - . **nemotron-3.5-super-vl-120b-a12b**｜Free Endpoint，图文理解、Agent、工具调用、代码，早期访问版本NVIDIA
    - . **nemotron-parse-2.0**｜文档 OCR、图片表格文本提取

  - **语音相关**

    - . **nemotron-voicechat**｜语音对话模型，英文语音交互（Free Endpoint）NVIDIA
    - . **riva-translate-4b-instruct-v2**｜Riva 翻译模型，37 语种翻译，少样本提示

  - **通用对话 LLM**

    - `qwen2.5-7b-instruct` / `qwen2.5-14b-instruct`｜通义千问，中文强，代码、问答
    - `llama-3.1-8b-instruct` / `llama-3.1-70b-instruct`｜Meta Llama3.1
    - `mistral-small-3`｜Mistral 轻量对话
    - `deepseek-v3`｜DeepSeek，代码、长文本推理
    - `kimi-k3`｜月之暗面 Kimi K3，超长上下文、多模态、代码
    - `step5-preview`｜阶跃 Step5，多模态 MoE，1M 上下文

  - **代码专用模型**

    - `starcoder2-7b`｜BigCode 代码模型，支持数百种编程语言
    - `deepseek-coder`｜DeepSeek 代码模型

  - 多模态 VLM

    - `qwen2.5-vl-7b-instruct`｜Qwen2.5-VL，图文理解
    - `muse-glimmer-30b`｜Muse Glimmer 30B，图文推理、工具调用NVIDIA
   


> 注意以上模型有的不是很稳定，会出现429等等问题，但是如果是Nvidia本身的模型效果会比较好，但是比较慢 

### agnes(多模态)
1.  官网 [地址](https://agnes-ai.com/)
2. 准备一个邮箱即可以
3. 登录申请apikey
4. 调用模型例子


```python
import os
from openai import OpenAI

client = OpenAI(
  base_url = "https://integrate.api.nvidia.com/v1",
  api_key = os.environ["apiKey"]
)


completion = client.chat.completions.create(
  model="agnes-2.5-flash",
  messages=[{"role": "user", "content": [{"type": "text", "text": "Read the traffic sign in this image. What does it say, and what color is the sign?"}, {"type": "image_url", "image_url": {"url": "https://assets.ngc.nvidia.com/products/api-catalog/phi-3-5-vision/example1a.jpg"}}]}],
  temperature=1,
  top_p=0.95,
  max_tokens=16384,

  parallel_tool_calls=True,



  seed=42,
  extra_body={"chat_template_kwargs": {"enable_thinking": True}},
  stream=True
)

for chunk in completion:
  if not chunk.choices:
    continue
  reasoning = getattr(chunk.choices[0].delta, "reasoning_content", None)
  if reasoning:
    print(reasoning, end="")
  if chunk.choices[0].delta.content is not None:
    print(chunk.choices[0].delta.content, end="")

```

> 注意以上模型可能会淘汰旧版本，需要留意更新模型


### cloudflare 模型（Workers AI）

1.  官网 [地址](https://dash.cloudflare.com/)
2. 准备一个邮箱即可以
3. 登录申请apikey
4. 调用模型例子
5. 模型列表
  - . `@cf/meta/llama-3.1-8b-instruct` ｜ Llama3.1 8B，通用对话、多语言、函数调用，最常用
  - . `@cf/zai-org/glm-4.7-flash`｜智谱 GLM 4.7 flash，中文表现很好
  - . `@cf/meta/llama-3.1-8b-instruct-fast`｜Llama3.1 8B 快速版，延迟更低
  - . `@cf/google/gemma-2-9b-it`｜Gemma 2 9B，谷歌开源模型
  - . `@cf/qwen/qwen2.5-7b-instruct`｜通义千问 Qwen2.5 7B，中文强，适合写代码、问答
  - . `@cf/meta/llama-guard-3-8b`｜内容安全审核模型

>

```python
import requests

API_BASE_URL = "https://api.cloudflare.com/client/v4/accounts/[Account ID]/ai/run/"
headers = {"Authorization": "Bearer [api-key]"}

def run(model, inputs):
    input = { "messages": inputs }
    response = requests.post(f"{API_BASE_URL}{model}", headers=headers, json=input)
    return response.json()

inputs = [
    { "role": "system", "content": "You are a friendly assistan that helps write stories" },
    { "role": "user", "content": "Write a short story about a llama that goes on a journey to find an orange cloud "}
];
output = run("@cf/moonshotai/kimi-k2.7-code", inputs)
print(output)
```


> 注意每日：**10,000 Neurons**，所有模型共用，用完当天请求直接报错；文本生成速率限制：**300 请求 / 分钟**；