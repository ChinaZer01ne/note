## 基础概念

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
* MCP
* Agent


* 
* 
* 梯度下降：局部最优
	* 随机梯度下降
	* mini-batch
* 反向传播算法
	* 前馈
	* 反馈