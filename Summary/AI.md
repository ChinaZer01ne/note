## 通识
### 基础概念

* LLM：生成式AI
* Token：Token 是LLM处理文本的最基本单元
* Tokenizer：
	* 编码：将自然语言切分成Token，映射成LLM能看懂的Token ID
	* 解码：将LLM返回的Token ID，转换成Token，让人类能看懂
* Context：模型在单次任务处理中所接收到的信息总和，临时记忆体。
	* Context Window：模型能够单次容纳的最大 Token 数量上限，现代主流模型（如 GPT-5.4, Gemini, Claude）的窗口已普遍达到 100万至200万 Token 的级别。1个Token约等于1.5到2个汉字。
* RAG：检索增强生成
* Prompt Engineering：通过清晰、具体、明确的指令，提高模型的输出质量。
	* System Prompt (系统提示词)：定义模型的人设（如“你是一个数学老师”）、做事规则、语言风格及禁忌。用户通常感知不到其存在。
	* User Prompt (用户提示词)：用户在对话框中输入的具体问题或任务请求。
* Tool
	* 大模型具有知识截断（无法感知实时信息）和闭源性（无法直接操作外部软件）的弱点。工具允许模型感知并影响外部环境（如查询实时天气、操作文件）。
	* Tool就是函数，实际LLM并不能调用Tool，LLM只能生成文字，LLM通过平台调用Tool，Tool是接入在平台里的。
	* 工具调用的完整流程：涉及四个关键角色：用户、大模型、工具、平台 (传话筒)
		1. 平台发送：将用户问题与可用工具列表（函数定义）发送给模型。
		2. 模型决策：分析需求，决定是否调用工具，并生成包含工具名和参数的指令。
		3. 平台执行：平台捕获指令，真实执行外部函数，并获取结果。
		4. 模型总结：模型拿到工具返回的数据，进行归纳总结，输出最终的人类语言。
* MCP
	* MCP (模型上下文协议)：由 Anthropic 提出的一套统一的工具接入规范。
	* 标准化：以往每个大模型平台（OpenAI, Google, Anthropic）的工具接入标准不一，导致开发者需重复开发。
	- 通用性：开发者只需按 MCP 标准编写一次工具，即可在所有支持该协议的 AI 平台间通用，类似于硬件领域的 Type-C 接口标准。
	- Tool通过MCP接入平台，供LLM调用。
* Agent
	* 特征：具备自主规划能力。
	* 运作模式：面对复杂任务，Agent 能自动拆解步骤，并连续、多次调用工具，直至目标达成。
	- 构建模式：常见的包括 ReAct (Reason + Act) 和 Plan and Execute。
	- Agent产品：Claude Code、Codex、Gemini CLI这些都是Agent，他们都具备规划能力，拆解任务，完成任务。他们底层的LLM是比如Claude Opus4.7、GPT5.5、Gemini3等
- Agent Skill
	- 本质：一份给 Agent 阅读的说明文档（通常为 Markdown 格式），用于规范其特定场景下的行为。 
	- 结构组成：
		- 元数据层 (Metadata)：包含名称 (Name) 和描述 (Description)，用于 Agent 匹配判断。
		- 指令层 (Instructions)：规定目标、执行步骤、判断逻辑、输出格式及示例 (Few-shots)。
	- 工程规范（以 Cline 为例）：
		- 需存放在特定目录（如 `skills/`）。
		- 文件夹名称必须与技能名一致。
		- 文件名必须固定为 `SKILL.md`。


> LLM 作为核心大脑，通过 Token 处理信息，在 Context 窗口内运作。通过 User 和 System Prompt 的配合，利用 Tool 和 MCP 协议扩展能力边界。当系统具备自主规划能力时即进化为 Agent，而 Agent Skill 则为其提供了精细化执行特定任务的专业指南。


### 其他概念

* GPT和ChatGPT
	* gpt是LLM，ChatGPT是大模型应用：web应用 + LLM
* 温度（temprature）：越高越随机、有创造力；越低越确定越保守
	* 通过影响softMax函数，来控制每个Token的概率大小，温度越低，概率差距越大，温度越高，概率差距越小
* top_p：越高越随机、有创造力；越低越确定越保守
	* 通过控制长尾词的概率阈值来影响输出，比如0.5的意思就是只保留概率靠前50的词
* 重复性惩罚
* 模型幻觉
* 流式输出（SSE）
* Transformer
* 梯度下降：局部最优
	* 随机梯度下降
	* mini-batch
* 反向传播算法
	* 前馈
	* 反馈

## LLM部署

#### 云服务
#### 私有部署

##### ollama

###### 下载

https://ollama.com/

###### 安装

windows安装：

```powershell
// 可以指定安装目录
OllamaSetup.exe /DIR=D:\develop\ollama
```



## 对接LLM

### 千问

```python
import requests
import json

API_KEY = "你的通义key"
url = "https://dashscope.aliyuncs.com/api/v1/services/aigc/text-generation/generation"

headers = {
    "Authorization": f"Bearer {API_KEY}",
    "Content-Type": "application/json"
}

data = {
    "model": "qwen-turbo",
    "input": {
        "messages": [
            {"role":"system","content":"你是专业Java技术顾问"},
            {"role":"user","content":"解释一下Spring AI是什么"}
        ]
    },
    "parameters": {"result_format":"message"}
}

res = requests.post(url, headers=headers, json=data)
print(res.json()["output"]["choices"][0]["message"]["content"])

```
### OpenAI
```python
import requests

api_key = "你的key"
url = "https://api.openai.com/v1/chat/completions"

headers = {
    "Authorization": f"Bearer {api_key}",
    "Content-Type":"application/json"
}

body = {
    "model":"gpt-3.5-turbo",
    "messages":[
        {"role":"user","content":"介绍RAG原理"}
    ],
    "temperature":0.7
}

res = requests.post(url, headers=headers, json=body)
print(res.json()["choices"][0]["message"]["content"])

```

#### 流式调用

```python
# 加上 stream: true 就是流式输出
body = {
    "model":"gpt-3.5-turbo",
    "messages":[{"role":"user","content":"简单介绍Java"}],
    "stream":True
}
```

### Deepseek

```python
from openai import OpenAI

client = OpenAI(
    api_key="sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",  # DeepSeek API Key
    base_url="https://api.deepseek.com/v1"
)

resp = client.chat.completions.create(
    model="deepseek-chat",  # 或 deepseek-coder
    messages=[
        {"role": "system", "content": "你是一个技术专家"},
        {"role": "user", "content": "解释一下模型幻觉"}
    ],
    temperature=0.7
)

print(resp.choices[0].message.content)

```

### Claude
```python
import anthropic

client = anthropic.Anthropic(
    api_key="sk-ant-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
)

msg = client.messages.create(
    model="claude-3-sonnet-20240229",  # opus/sonnet/haiku
    max_tokens=1024,
    system="你是技术顾问",
    messages=[
        {"role": "user", "content": "写一段 Python 快速排序"}
    ]
)

print(msg.content[0].text)

```

### Gemini
```python
import google.generativeai as genai

genai.configure(api_key="xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx")  # Gemini API Key

model = genai.GenerativeModel("gemini-pro")  # 文本用 gemini-pro，多模态用 gemini-pro-vision
resp = model.generate_content("解释什么是对话上下文")
print(resp.text)

```