## 总结
在完成日志功能的开发后，想到它能够通过日志装饰器实现在函数内部自由调用，于是试图用它完成企业级项目的一个必备模块——成本追踪的开发。在实现的过程中发现并解决了一些业务问题：
* 在实际需要提取信息时才意识到的计数（`input_token`、`output_token`、`total_token`等）在`AIMessage`和`ToolMessage`中的位置，不同大模型服务提供商可能会不同，如何设计一种通用的方法累计token消耗总数并正确地处理？【第一节的结论】
* token的价格不同厂商的不同模型价格不同，如何方便且可扩展地记录？【第三节的优化】
* 假设在视图层某个功能（函数）中调用了`invoke()`方法，成本分析只能够追踪外部的返回信息（如在返回的消息列表中一个个地累计），是否有方法能够深入Agent调用过程内部，在每个`BaseMessage`子类实例被创建出的时候就进行成本分析？【第二节分析的原理】
在认识并理解LangChain的回调机制过程中，完成了对回调机制的大量源码阅读，深入梳理了智能体启动前内部的静态结构。并结合`Runnable`机制，对该机制的运行过程进行了分析，最后对整个智能体，以及成熟的AI应用中一些工具的必要性有了更深的理解。


## 学习过程
### 一、`BaseMessage`及其子类中的token信息
#### 1. `Message`系列类的关系
LangChain框架就是通过几个抽象基类发散功能构成的，其中所有的消息类都继承自`BaseMessage`（流式输出除外，它们继承的是`BaseMessageChunk`），它们的主要区别在于角色`role`，这决定了消息在对话中的语义。
在`langchain_core/messages/__init__.py`中，暴露了所有的`Message`相关的类：

| 类名                  | 继承消息基类                          | 职能（源码注释）                                                                     | 位置                     |
| ------------------- | ------------------------------- | ---------------------------------------------------------------------------- | ---------------------- |
| **`SystemMessage`** | `BaseMessage`                   | 规范AI行为的消息，一般作为输入消息序列中的第一个消息                                                  | `messages/system.py`   |
| **`HumanMessage`**  | `BaseMessage`                   | 来自用户的消息                                                                      | `messages/human.py`    |
| **`AIMessage`**     | `BaseMessage`                   | 代表了AI的输出结果，包含模型返回的原始输出，以及LangChain框架添加的标准化字段（`tool_calls`、`usage_metadata`等） | `messages/ai.py`       |
| **`ToolMessage`**   | `BaseMessage` `ToolOutputMixin` | 传递工具执行结果的消息，通过`content`字段给出工具执行结果，通过`tool_call_id`，在多工具场景下将工具调用请求与工具调用结果联系   | `messages/tool.py`     |
| `FunctionMessage`   | `BaseMessage`                   | **是`ToolMessage`的过时版本**，且不包含`tool_call_id`字段                                 | `messages/function.py` |
| `ChatMessage`       | `BaseMessage`                   | 可任意分配角色的消息，有两个字段：`role`和`type`（消息类型）                                         | `messages/chat.py`     |
| `RemoveMessage`     | `BaseMessage`                   | 负责删除其他消息的消息                                                                  | `messages/modifier.py` |
| `AnyMessage`        | 无                               | 表示任何已定义的`Message`或`MessageChunk`类型                                           | `messages/utils.py`    |
可见最常用的几种消息：`SystemMessage`、`HumanMessage`、`AIMessage`都统一由一个抽象基类管理，而其派生出的`MessageChunk`类，则通过多继承的方式额外支持流式输出（`ToolMessage`例外）：

| 类名                     | 继承流式消息基类                                  | 职能（源码注释）                        | 位置                     |
| ---------------------- | ----------------------------------------- | ------------------------------- | ---------------------- |
| `SystemMessageChunk`   | `BaseMessageChunk`、`SystemMessageChunk`   | 流式输出时产生（yielded when streaming） | `messages/system.py`   |
| `HumanMessageChunk`    | `BaseMessageChunk`、`HumanMessageChunk`    | 同上                              | `messages/human.py`    |
| `AIMessageChunk`       | `BaseMessageChunk`、`AIMessage`            | 同上                              | `messages/ai.py`       |
| `ToolMessageChunk`     | `BaseMessageChunk`、`ToolMessage`          | 同上                              | `messages/tool.py`     |
| `ChatMessageChunk`     | `BaseMessageChunk`、`ChatMessage`          | 同上                              | `messages/chat.py`     |
| `FunctionMessageChunk` | `BaseMessageChunk`、`FunctionMessageChunk` | 同上                              | `messages/function.py` |
#### 2. `BaseMessage`类与`BaseMessageChunk`类
`BaseMessage`是**所有非流式输出消息的唯一基类**（`ToolMessage`除外，它还继承了工具相关的`Mixin`混合类），它继承于Pydantic中`BaseModel`的自定义子类`Serializable`，源码中的定义为：
```python
class BaseMessage(Serializable):
	content: str | list[str | dict]
	additional_kwargs: dict = Field(default_factory=dict)
	response_metadata: dict = Field(default_factory=dict)
	type: str
	name: str | None = None
	id: str | None = Field(default=None, coerce_numbers_to_str=True)
```
另外`BaseMessageChunk`继承于`BaseMessage`，并且只添加了一个`__add__`方法用于拼接。

#### 3. `Message`类中的token统计
**实例1：** 使用deepseek模型创建智能体的返回结果：
```json
{
	'messages': [
		AIMessage(
			content='你好！我是你的智能助手，随时准备为你提供帮助。无论是回答问题、解决问题、提供建议，还是陪你聊天，我都在这里！请告诉我你需要什么帮助吧！ 😊', 
			additional_kwargs={'refusal': None}, 
			response_metadata={
				'token_usage': {
					'completion_tokens': 38, 
					'prompt_tokens': 7, 
					'total_tokens': 45, 
					'completion_tokens_details': None, 
					'prompt_tokens_details': {
						'audio_tokens': None, 
						'cached_tokens': 0
					}, 
					'prompt_cache_hit_tokens': 0, 
					'prompt_cache_miss_tokens': 7
				}, 
				'model_provider': 'deepseek', 
				'model_name': 'deepseek-chat', 
				'system_fingerprint': 'fp_eaab8d114b_prod0820_fp8_kvcache', 
				'id': '3f0bdd13-64fe-4b34-8a1f-d5ff1885e60b', 
				'finish_reason': 'stop', 
				'logprobs': None
			}, 
			id='lc_run--019bb097-9a26-78f2-8d4e-21e705f51bdc-0', 
			tool_calls=[], 
			invalid_tool_calls=[], 
			usage_metadata={
				'input_tokens': 7, 
				'output_tokens': 38, 
				'total_tokens': 45, 
				'input_token_details': {
					'cache_read': 0
				}, 
				'output_token_details': {}
			}
		)
	]
}
```
分解一下这个`AIMessage`的结构：在源码定义中它新增了`tool_calls`、`invalid_tool_calls`、`usage_metadata`字段，并将基类的`type`字段赋值为`"ai"`（不显示）。这就是上面返回结果的最后三个字段。并且能够注意到，**和消息耗费token相关的字段有`response_metadata`和`usage_metadata`，且两个字段中token计数重合。**
**实例2：** 结合多轮工具调用返回的结果，它包含了最常用的消息类型（Human -> AI -> Tool -> AI）：
```json
{
'messages': [
	HumanMessage(
		content='请帮我查询2024年诺贝尔物理学奖得主是谁？', 
		additional_kwargs={}, 
		response_metadata={}, 
		id='9e05fe69-413e-430a-8c09-cfc91416a589'
	), 
	
	%% --- 第一次调用工具解决问题 --- %%
	AIMessage(
		content='我来帮您查询2024年诺贝尔物理学奖得主的信息。', 
		additional_kwargs={'refusal': None}, 
		response_metadata={
			'token_usage': {
				'completion_tokens': 99, 
				'prompt_tokens': 1864, 
				'total_tokens': 1963, 
				'completion_tokens_details': None, 
				'prompt_tokens_details': {
					'audio_tokens': None, 
					'cached_tokens': 0
				}, 
				'prompt_cache_hit_tokens': 0, 
				'prompt_cache_miss_tokens': 1864
			}, 
			'model_provider': 'deepseek', 
			'model_name': 'deepseek-chat', 
			'system_fingerprint': 'fp_eaab8d114b_prod0820_fp8_kvcache', 
			'id': 'e743c6a3-5c2e-425b-b152-466bf44e7a3c', 
			'finish_reason': 'tool_calls', 
			'logprobs': None
		}, 
		id='lc_run--019bb0a3-0a51-7092-9b52-e8f4aa9566ee-0', 
		tool_calls=[{
			'name': 'tavily_search', 
			'args': {
				'query': '2024年诺贝尔物理学奖得主', 
				'search_depth': 'advanced', 
				'time_range': 'year'
			}, 
			'id': 'call_00_6RJOId6kxsYmMbiUwvoSkv4m', 
			'type': 'tool_call'
		}], 
		invalid_tool_calls=[], 
		usage_metadata={
			'input_tokens': 1864, 
			'output_tokens': 99, 
			'total_tokens': 1963, 
			'input_token_details': {
				'cache_read': 0
			}, 
			'output_token_details': {}
		}
	), 
	
	ToolMessage(
		content='{
			"query": "2024年诺贝尔物理学奖得主", 
			"follow_up_questions": null, 
			"answer": null, 
			"images": [], 
			"results": [
				{
					"url": "https://bimonthly.ps-taiwan.org/articles/67bd3bf214c2b3f65f312b8f", 
					"title": "2024年諾貝爾物理獎--- 人工智慧上的奠基與發明", 
					"content": "國立中央大學物理系 黎璧賢教授2025年2月25日2857\\n\\n圖片來源：www.pexels.com\\n\\n分享：\\n\\n2024年諾貝爾物理獎頒給美國普林斯頓大學的約翰‧霍普菲爾德(John J. Hopfield，91歲)，以及加拿大多倫多大學的傑佛瑞‧辛頓(Geoffrey E. Hinton，76歲)，「表揚他們在人工智慧與機器學習上的奠基與發明」。最近，生成式人工智慧變得越來越流行和易於使用，對科學和日常應用產生了巨大並深遠的影響。儘管電腦無法思考，但人工智慧(Artificial Intelligence，AI)機器可以模仿記憶和學習等功能，而機器學習則一直主導著人工智慧的成長與發展。過去十五到二十年，機器學習的發展呈現爆炸性的成長主要是利用一種稱為人工神經網絡(Artificial Neural Network，ANN)的工具。巧合的是今年是易辛模型 【截断】", 
					"score": 0.90849644, 
					"raw_content": null
				}, 
				{
					"url": "https://www.jfdaily.com/news/detail?id=995353", 
					"title": "刚刚揭晓！三名科学家获诺贝尔物理学奖", 
					"content": "### 关注我们\\n\\n官方微博\\n\\n分享到微信朋友圈\\n\\n打开微信，点击底部的“发现”，使用 “扫一扫” 即可将网页分享到我的朋友圈。\\n\\n# 刚刚揭晓！三名科学家获诺贝尔物理学奖\\n\\n科科哒\\n2025-10-07 05:53\\n\\n来源：上观新闻\\n作者：新民晚报 郜阳\\n\\n稍后带来更多解读\\n\\n {{brTitle}}\\n\\n01北京时间6日17时45分，2025年度诺贝尔物理学奖揭晓。\\n\\n02今年物理学奖授予约翰·克拉克、米歇尔·H·德沃雷、约翰·M·马丁尼斯，以表彰他们发现电路中的宏观量子力学隧穿效应和能量量子化。【截断】", 
					"score": 0.8812988, 
					"raw_content": null
				}
			], 
			"response_time": 3.45, 
			"request_id": "b2caa602-0ee1-47a7-ba83-2d3cb5841dde"
		}', 
		name='tavily_search', 
		id='3925e83a-33d2-49fc-a33f-c215c70d2dc1',
		tool_call_id='call_00_6RJOId6kxsYmMbiUwvoSkv4m'),
	
	%% --- 第二次调用工具解决问题 --- %% 
	AIMessage(
		content='让我再搜索一些更权威的官方信息来确认。', 
		additional_kwargs={'refusal': None}, 
		response_metadata={
			'token_usage': {
				'completion_tokens': 98, 
				'prompt_tokens': 4362, 
				'total_tokens': 4460, 
				'completion_tokens_details': None, 
				'prompt_tokens_details': {
					'audio_tokens': None, 
					'cached_tokens': 1920
				}, 
				'prompt_cache_hit_tokens': 1920, 
				'prompt_cache_miss_tokens': 2442
			}, 
			'model_provider': 'deepseek', 
			'model_name': 'deepseek-chat', 
			'system_fingerprint': 'fp_eaab8d114b_prod0820_fp8_kvcache', 
			'id': '159649fe-18e3-46a2-adf7-fe9be742ee1d', 
			'finish_reason': 'tool_calls', 
			'logprobs': None
		}, 
		id='lc_run--019bb0a3-2ba7-7173-87b5-d680f66574d5-0', 
		tool_calls=[{
			'name': 'tavily_search', 
			'args': {
				'query': '2024 Nobel Prize in Physics winners official announcement', 
				'search_depth': 'advanced', 
				'time_range': 'year'
			}, 
			'id': 'call_00_ste2P3uecFQELA77gGmjXFtZ', 
			'type': 'tool_call'
		}], 
		invalid_tool_calls=[], 
		usage_metadata={
			'input_tokens': 4362, 
			'output_tokens': 98, 
			'total_tokens': 4460, 
			'input_token_details': {'cache_read': 1920}, 
			'output_token_details': {}
		}
	),
	 
	ToolMessage(
		content='{
			"query": "2024 Nobel Prize in Physics winners official announcement",
			"follow_up_questions": null, 
			"answer": null, 
			"images": [], 
			"results": [
				{
					"url": "https://blog.google/【截断】", 
					"title": "Googler Michel 【截断】", 
					"content": "x.com Facebook 【截断】", 
					"score": 0.8820324, 
					"raw_content": null
				}, {
					"url": "https://www.nobelprize.org/【截断】", 
					"title": "Nobel Prize Dialogue", 
					"content": "【截断】", 
					"score": 0.84918517, 
					"raw_content": null
				}
			], 
			"response_time": 2.07, 
			"request_id": "07b2c776-dec7-4ad8-95de-8b5c68b5eefe"
		}', 
		name='tavily_search', 
		id='09004523-6baf-47f9-90af-c0d2c407b188', 
		tool_call_id='call_00_ste2P3uecFQELA77gGmjXFtZ'), 
	
	%% --- 第三次调用工具解决问题 --- %%
	AIMessage(
		content='让我再搜索一下更具体的官方信息。', 
		additional_kwargs={'refusal': None}, 
		response_metadata={
			'token_usage': {
				'completion_tokens': 101, 
				'prompt_tokens': 5078, 
				'total_tokens': 5179, 
				'completion_tokens_details': None, 
				'prompt_tokens_details': {
					'audio_tokens': None, 
					'cached_tokens': 4416
				}, 
				'prompt_cache_hit_tokens': 4416, 
				'prompt_cache_miss_tokens': 662
			}, 
			'model_provider': 'deepseek', 
			'model_name': 'deepseek-chat', 
			'system_fingerprint': 'fp_eaab8d114b_prod0820_fp8_kvcache', 
			'id': '6c7411ea-d0db-47b7-bfd1-b1b5ce6afeab', 
			'finish_reason': 'tool_calls', 
			'logprobs': None
		}, 
		id='lc_run--019bb0a3-47e1-79b1-919a-3d2933014513-0', 
		tool_calls=[{
			'name': 'tavily_search', 
			'args': {
				'query': '"2024 Nobel 【截断】', 
				'search_depth': 'advanced', 
				'time_range': 'year'
			}, 
			'id': 'call_00_7ywqfC5G9migCbsxl2b68L14', 
			'type': 'tool_call'
		}], 
		invalid_tool_calls=[], 
		usage_metadata={
			'input_tokens': 5078, 
			'output_tokens': 101, 
			'total_tokens': 5179, 
			'input_token_details': {'cache_read': 4416}, 
			'output_token_details': {}
		}
	),
			 
	ToolMessage(
		content='{
			"query": "2024 Nobel Prize 【截断】", 
			"follow_up_questions": null, 
			"answer": null, 
			"images": [], 
			"results": [
				{
					"url": "https://www.nobelprize.org/【截断】", 
					"title": "The Nobel Foundation annual review 2024", 
					"content": "Foundation 【截断】", 
					"score": 0.65861565, 
					"raw_content": null
				}, {
					"url": "https://www.youtube.com/watch?v=XDE9DjpcSdI", 
					"title": "Nobel Prize lecture: 【截断】", 
					"content": "# Nobel Prize lecture: 【截断】", 
					"score": 0.63750535, 
					"raw_content": null
				}
			], 
			"response_time": 2.96, 
			"request_id": "a6c7c5b5-9253-4a41-a107-8679b35f2def"
		}', 
		name='tavily_search', 
		id='60d7f77e-0ab3-4a5f-af07-8a3c0d26a0c5',
		tool_call_id='call_00_7ywqfC5G9migCbsxl2b68L14'
	), 
	
	%% --- 最终回答 --- %%
	AIMessage(
		content='根据搜索结果，我可以为您提供2024年诺贝尔物理学奖得主的准确信息：\n\n## 2024年诺贝尔物理学奖得主\n\n**获奖者：**\n1. **约翰·J·霍普菲尔德（John J. Hopfield）** - 美国普林斯顿大学教授，91岁\n2. **杰弗里·E·辛顿（Geoffrey E. Hinton）** - 加拿大多伦多大学教授，76岁\n\n**获奖理由：**\n"表彰他们在人工神经网络机器学习方面的奠基性发现和发明"（"for foundational discoveries and inventions that enable machine learning with artificial neural networks"）\n\n**主要贡献：**\n1. **约翰·霍普菲尔德**：发明了霍普菲尔德网络（Hopfield network），这是一种基于物理学中伊辛模型（Ising model）的人工神经网络，能够存储和重建模式，在图像分析等领域有重要应用。\n\n2. **杰弗里·辛顿**：以霍普菲尔德网络为基础，开发了玻尔兹曼机（Boltzmann machine），这是一种能够学习识别特定数据类型特征元素的新型网络。他的工作推动了当前机器学习的爆炸性发展。\n\n**重要意义：**\n- 这两位科学家的研究将物理学概念（特别是统计力学）应用于人工智能领域\n- 他们的工作为现代机器学习奠定了基础\n- 人工神经网络现在已成为物理学、科学和技术各个领域建模和分析的强大工具\n\n值得注意的是，这是诺贝尔物理学奖首次颁发给人工智能/机器学习领域的研究者，反映了人工智能技术在当代科学中的重要性日益增长。', 
		additional_kwargs={'refusal': None}, 
		response_metadata={
			'token_usage': {
				'completion_tokens': 311, 
				'prompt_tokens': 5908, 
				'total_tokens': 6219, 
				'completion_tokens_details': None, 
				'prompt_tokens_details': {
					'audio_tokens': None, 
					'cached_tokens': 5120
				}, 
				'prompt_cache_hit_tokens': 5120, 
				'prompt_cache_miss_tokens': 788
			}, 
			'model_provider': 'deepseek', 
			'model_name': 'deepseek-chat', 
			'system_fingerprint': 'fp_eaab8d114b_prod0820_fp8_kvcache', 
			'id': '8a76cc25-bceb-4975-b76d-85292e52c41d', 
			'finish_reason': 'stop', 
			'logprobs': None
		}, 
		id='lc_run--019bb0a3-66d5-78d0-b319-bee820c5197e-0', 
		tool_calls=[], 
		invalid_tool_calls=[], 
		usage_metadata={
			'input_tokens': 5908, 
			'output_tokens': 311, 
			'total_tokens': 6219, 
			'input_token_details': {'cache_read': 5120}, 
			'output_token_details': {}
			}
		)
	]
}
```
以上结果进行了三次工具调用，可以发现：
* `ToolMessage`虽然将结果加入下一轮调用的上下文中，**但并不提供token信息**，因为最终`AIMessage`会统计包含了工具返回结果的总token消耗。**所以成本追踪只需参考`AIMessage`中的信息**。另外源码中对`response_metadata`字段的解释为（**仅继承，但不用**）：
```python
class ToolMessage(BaseMessage, ToolOutputMixin):
	additional_kwargs: dict = Field(default_factory=dict, repr=False)  
	"""Currently inherited from `BaseMessage`, but not used."""  
	response_metadata: dict = Field(default_factory=dict, repr=False)  
	"""Currently inherited from `BaseMessage`, but not used."""
```
* `AIMessage`新加入的`usage_metadata`字段与`BaseMessage`定义的`response_metadata`中的`token_usage`高度重合，并且有以下对应：

| `BaseMessage`的`response_metadata["token_usage"]` | `AIMessage`新增属性`usage_metadata`     |
| ------------------------------------------------ | ----------------------------------- |
| `prompt_tokens`                                  | `input_tokens`                      |
| `completion_tokens`                              | `output_tokens`                     |
| `total_tokens`                                   | `total_tokens`                      |
| `prompt_tokens_details["cached_tokens"]`         | `input_token_details["cache_read"]` |
| `prompt_cache_hit_tokens`                        | `input_token_details["cache_read"]` |
目前只分析`input_tokens`、`output_tokens`、`total_tokens`，后续分别称为输入token、输出token、总token。然后对四个AI消息对进行token数据对比：

|                  | 输入token | 输出token | 总token |
| ---------------- | ------- | ------- | ------ |
| 第一次AI消息（第一次工具调用） | 1864    | 99      | 1963   |
| 第二次AI消息（第二次工具调用） | 4362    | 98      | 4460   |
| 第三次AI消息（第三次工具调用） | 5078    | 101     | 5179   |
| 第四次AI消息（输出结果）    | 5908    | 311     | 6219   |
从这三次调用可以看出LangChain框架进行的token计算的机制：
1. 首次调用虽然问题简单：“你好，你叫什么？”，但是仍有1864的token输入，可能来源于提示词、工具列表（包含工具解释和它的参数文档）等等。
2. 第二次的token数飙升，达到4362。这来源于第一次工具调用的返回结果+上下文
3. 第三次的token数只增不减，可能因为没有配备上下文管理器，导致只会不断地加入新信息
4. 第四次的输出token与前三次不同，因为前三次的输出是用作Function Call，有固定的格式，且调用的工具相同，第四次输出是最终结果。另外，输入token数仍在在增加。

从这个输出结果的分析中可以发现以下问题：
**Q：** 多轮的调用大模型必定会造成成本上涨，目前只有三轮的工具调用，其总token消耗量就达到了接近两万，甚至工具调用（网页搜索）只要求输出了两个页面。如果在生产环境下，开销一定会更大，所以缩减token消耗是非常重要的。目前已知**上下文管理器可以缓解这个问题**，但是如果**引入多智能体分工协作，是否一定程度上可以解决这个问题**？
**A：** 传统多轮调用Agent的token成本是线性，甚至指数级增长的。多智能体工作流是一种前沿的解决该问题的方法，它的核心思路是**分工与记忆隔离**：由”调度员“Agent分析任务，将子任务分发给专门的”执行员“Agent。**每个执行员只需关注自己的上下文，避免了全局历史堆积**；另外每个Agent可以拥有独立的短期记忆，可以**通过各种方法压缩上下文，而非传递原始长度的文本**。

**Q：** 实现回调或中间件监控程序执行流程的方法很多，FastAPI有处理HTTP请求和发送HTTP响应之间作用的中间件，LangChain也有能够在智能体工作流之间作用的中间件，回调函数、日志提供的装饰器都能够实现监控和追踪函数内部执行流程的功能。**成本追踪应该处于哪一层**？
**A：** 由以上消息列表中的AI消息可以看出，大模型的每次调用所产生的`AIMessage`中的**token计数并非累加的，而是仅限于单次调用的**。所以如果只是在`invoke()`层面调用是粗粒度的，它只会得到最后一次调用的token消耗，造成成本误算。**所以需要在模型每次调用的层面进行成本追踪，可以使用LangChain提供的回调机制（`Callback`）成本追踪嵌入LLM的调用环节**。

### 二、LangChain的`Callback`机制
LangChain的中间件由`Callback`机制支持，它允许开发人员在应用执行的关键节点（如LLM调用前、工具执行后）插入自定义逻辑，用于监控、记录、调试，而无需改动核心业务代码。它的工作流程为：Agent执行`invoke()`方法 -> 生成LangChain可运行对象（`Runnable`） -> 由**回调管理器`CallbackManager`将事件分发给其他处理器**（如回调处理器`BaseCallbackHandler`、日志处理器），再由这些处理器内部的自定义逻辑完成监控、记录、调试等功能。
**Q：** 为什么感觉`Callback`机制和日志收集器`Logger`收集日志后分发给各种执行器`Handler`的机制很像？
**A：** 虽然存在侧重点的差异（**日志系统侧重异步记录，不改变代码执行；而同步的`Callback`则可能影响执行**），这两者是**同一种设计模式（观察者模式）在不同场景下的应用**：
* 同样有**发布事件的管理组件**（`Logger`、`CallbackManager`）；
* 同样有**被分发的消息**（`logging.info("msg")`这种日志调用、`on_llm_start`这些框架触发的事件）；
* 同样有**能够订阅并处理事件的执行器**（`Handler`、`BaseCallbackHandler`）；
* 同样有**将执行器注册到管理器的方法**（`logger.addHandler()`、`callbacks=[]`）；
* 同样有**一对多的数据流**（`Logger`->多个`Handler`、`CallbackManager`->多个`Handler`）。
所以可以把`CallbackManager`理解为LangChain内部一个**专有的”事件日志记录器“**，它不断地发布”LLM调用开始“、”工具调用结束“等事件，而`BaseCallbackHandler`就能够订阅这些事件、并执行特定动作。

关于源码以及更深入的机制解读在[[LangChain回调机制解析]]中，理解完成成本分析需要知道的就是：LangChain会为`Agennt.invoke()`的调用创建一个根回调管理器`CallbackManager`，它负责让Agent组件启动（就是创建运行时管理器`RunManager`并与本次组件的运行绑定）；这两个管理器，前者是父组件负责的子组件启动（`RunManager`的生成），子组件无法干预，后者是子组件自己的业务逻辑（错误、结束等事件，或者创造新的子组件调用）。**回调管理器只负责开始，运行时管理器`RunManager`负责后续，这些管理器将它们的事件发送给对应的执行器**。
而执行器基类`BaseCallbackHandler`则是能够接收并处理事件（在与事件同名的函数中完成自己的业务逻辑），管理器层面是LangChain框架底层的部分，它只需要提供多样的钩子，由开发人员来选不同位置的钩子去实现他们的方法。至于管理器层面如何管理，即使不知道细节也能够保证正常使用【[[LangChain回调机制解析]]中有底层的分析】。

### 三、向Agent添加回调执行器实现成本追踪
#### 1. 前置知识
实现回调执行器需要通过继承`BaseCallbackHandler`，这个基类多继承了分别对应六种不同功能的`Mixin`类，其中每个`Mixin`类包含一部分的事件，所以这个基类涵盖了LangChain能够提供的所有事件。要实现回调执行器，只需继承这个基类，并按需选择对应的事件，实现它的同名函数即可。
**关于事件选择**：在上面`BaseMessage`的分析中，有提到所有消息中只有`AIMessage`和token计数有关，所以事件应该选择在LLM调用完成，生成新的一条`AIMessage`的时候。
##### `langchain_core/outputs/llm_result.py`的`LLMResult`类
这个类是是Pydantic中`BaseModel`的子类，LangChain将其作为LLM调用结果的容器（**信息的顶级容器**），聊天和非聊天模型都生成这个对象。它包含了生成的结果和模型提供商想返回的任何附加信息。
```python
"""
In addition, users can access the raw output of either LLMs or chat models via callbacks. The `on_chat_model_end` and `on_llm_end` callbacks will return an LLMResult object containing the generated outputs and any additional information returned by the model provider.
"""
```
上面的注释是 `langchain_core/outputs/__init__.py`对`LLMResult`的补充信息，即可以通过回调机制，在`on_chat_model_end`和`on_llm_end`中访问这个类的实例**获取模型原始输出**。
**模型原始输出获取测试：** 在`on_llm_end`中输出`LLMResult`（源码中没找到`on_chat_model_end`的存在，和源码的注释冲突），智能体有一个查询天气（只返回固定值）的工具：
```python
from langchain_core.callbacks.base import BaseCallbackHandler

class TestingCallback(BaseCallbackHandler):
	def __init__(self):
		super().__init__()
		
	def on_llm_end(self, response: LLMResult, **kwargs):
		print("[LLM处理完成]")
		print("response类型: ", type(response))
		print("response内容: ",response)
```
将这个执行器的实例通过`config`的`callbacks`参数列表传入，即可让`TestingCallback`订阅`on_llm_end`事件。之后在LLM调用结束，生成新的`AIMessage`时就能够**在事件内获取到原始输出**：
```json
[LLM处理完成]
response类型: <class 'langchain_core.outputs.llm_result.LLMResult'>
response内容: generations=【略】

[LLM处理完成]
response类型: <class 'langchain_core.outputs.llm_result.LLMResult'>
%% 这里整理了格式 %%
response内容: 
generations=[
	[ChatGeneration(
		text='北京的天气情况如下：\n- **天气状况**：晴朗\n- **温度**：25°C\n- **湿度**：50%\n- **风力**：东南风3级\n\n今天北京的天气很不错，晴朗温暖，湿度适中，风力也不大，是个适合户外活动的好天气。', 
		generation_info={'finish_reason': 'stop', 'logprobs': None},
		message=AIMessage(
			content='北京的天气情况如下：\n- **天气状况**：晴朗\n- **温度**：25°C\n- **湿度**：50%\n- **风力**：东南风3级\n\n今天北京的天气很不错，晴朗温暖，湿度适中，风力也不大，是个适合户外活动的好天气。', 
			additional_kwargs={'refusal': None}, 
			response_metadata={'token_usage': {'completion_tokens': 61, 'prompt_tokens': 406, 'total_tokens': 467, 'completion_tokens_details': None, 'prompt_tokens_details': {'audio_tokens': None, 'cached_tokens': 384}, 'prompt_cache_hit_tokens': 384, 'prompt_cache_miss_tokens': 22}, 'model_provider': 'deepseek', 'model_name': 'deepseek-chat', 'system_fingerprint': 'fp_eaab8d114b_prod0820_fp8_kvcache', 'id': '36c4c3ad-8281-438a-8d9e-b81c2926ad51', 'finish_reason': 'stop', 'logprobs': None}, 
			id='lc_run--019bbaa8-22fb-7d00-b1e6-97cacbfa2543-0', 
			tool_calls=[], 
			invalid_tool_calls=[], 
			usage_metadata={'input_tokens': 406, 'output_tokens': 61, 'total_tokens': 467, 'input_token_details': {'cache_read': 384}, 'output_token_details': {}}
		)
	]
] 
llm_output={'token_usage': {'completion_tokens': 61, 'prompt_tokens': 406, 'total_tokens': 467, 'completion_tokens_details': None, 'prompt_tokens_details': {'audio_tokens': None, 'cached_tokens': 384}, 'prompt_cache_hit_tokens': 384, 'prompt_cache_miss_tokens': 22}, 'model_provider': 'openai', 'model_name': 'deepseek-chat', 'system_fingerprint': 'fp_eaab8d114b_prod0820_fp8_kvcache', 'id': '36c4c3ad-8281-438a-8d9e-b81c2926ad51'} 
run=None 
type='LLMResult'
```
第二次返回的`response`的内容中（与第一次格式相同）出现了一些新的字段和类：`generations`、`ChatGeneration`、`generation_info`、`run`、`type`；其中`ChatGeneration`包含三个参数：
* **`text`，它与`message`参数的消息的`content`字段一模一样（原理在类验证器中）**；
* `generation_info`，包含一些生成器使用的信息；
* `message`，它就是平时输出的消息列表中的`Message`，同样包含消息基类的`content`、`additional_kwargs`、`response_metadata`、`id`，以及`AIMessage`继承后新增的`tool_calls`、`invalid_tool_calls`、`usage_metadata`
所以要理解回调机制内部的LLM原始输出，需要分析`generations`和`ChatGeneration`：
* `generations`是二维列表，每个元素列表又是`generation`列表，这使得它可以在一次LLM调用中处理多个输入的问题（prompt），每次问题又可以生成多个候选的结果
```python
generations: list[  
    list[Generation | ChatGeneration | GenerationChunk | ChatGenerationChunk] 
]
"""
The first dimension of the list represents completions for different input prompts.  
The second dimension of the list represents different candidate generations for a given prompt.
"""
```
* `Generation`是对于非聊天模型的，它的输入输出不是消息而是字符串。
* `ChatGeneration`才是结果示例里出现的对应聊天模型的结果生成器，它有一个`message`字段，类型是`BaseMessage`，不是`List[BaseMessage]`，在结果示例中也是直接传入`AIMessage`，说明**这个结果生成器只针对一条消息输出，且通常针对AI消息**
```python
"""
The `message` attribute is a structured representation of the chat message.  
Most of the time, the message will be of type `AIMessage`.  
  
Users working with chat models will usually access information via either  
`AIMessage` (returned from runnable interfaces) or `LLMResult` (available  
via callbacks).
"""
```
* 另外，`ChatGeneration`包含一个`set_text()`模型验证器，**在模型实例化之后，将`message`字段中消息的`content`赋值给`text`字段**，源码中有明显的赋值逻辑：
```python
@model_validator(mode="after")  
def set_text(self) -> Self:
	if isinstance(self.message.content, str):  
	    text = self.message.content
	elif isinstance(self.message.content, list):
		... # 省略部分逻辑为：将列表中的文本拼接进text
	self.text = text
	return self
```
所以现在对于LLM的原始输出可以有这样一个理解：当调用了`Agent.invoke()`之后，框架将prompt（可以是多个）交给LLM分析，输出了示例中的结果。这个结果由`LLMResult`这个类的实例作为容器接收，它的`generations`字段是个二维列表，每个列表元素对应一个prompt（示例只有一个prompt所以只有一个列表）；而其中每个列表内的元素是`Generation`系列类，里面每个`Generation`实例对应了LLM针对同一prompt的所有候选结果（示例中未设置多个候选结果所以只有一个`ChatGeneration`），最终呈现出的效果就是一个1×1的矩阵里只有一个`ChatGeneration`实例。
所以这个示例中，在进行成本追踪时，想要获取到`AIMessage`的字段，就需要使用`first_gen = response.generations[0][0]`获取到`ChatGeneration`实例，再通过`ai_message = first_gen.message`获取到`AIMessage`实例，最后按照前面对字段的理解，选择对应的`usage_metadata`或`response_metadata`字段，获取到token的值。
#### 2. 正式的成本追踪回调执行器
定义一个回调执行器`CostTrackingCallback`，它订阅`on_llm_end`事件，记录每条消息的token消耗。另外，需要有记录累计token消耗的属性，和token的输入输出成本：
```python
class CostTrackingCallback(BaseCallbackHandler):  
    """累计Token消耗的回调处理器"""  
    def __init__(self):  
        super().__init__()  
        # 初始化累计变量  
        self.total_input_tokens = 0  
        self.total_output_tokens = 0  
        self.total_cost = 0.0  
      
    def on_llm_end(self, response: LLMResult, **kwargs):  
        print(f"[LLM处理完成]")  
          
        try:  
            if response.generations:  
                first_gen = response.generations[0][0]  
                  
                if hasattr(first_gen, 'message'):  
                    ai_message = first_gen.message  
                      
                    if hasattr(ai_message, 'usage_metadata'):  
                        tokens = ai_message.usage_metadata  
                        input_tokens = tokens.get('input_tokens', 0)  
                        output_tokens = tokens.get('output_tokens', 0)  
                          
                        # 核心：累加Token到实例变量  
                        self.total_input_tokens += input_tokens  
                        self.total_output_tokens += output_tokens  
                          
                        print(f"单次Token数据 -> 
		                        输入: {input_tokens}, 
		                        输出: {output_tokens}")  
                        print(f"累计Token数据 -> 
		                        输入: {self.total_input_tokens}, 
		                        输出: {self.total_output_tokens}")  
        except Exception as e:  
            print(f"解析响应时出错: {e}")
```
然后将执行器的实例在智能体调用时传入：
```python
# 创建工具、模型、Agent
web_search = TavilySearch(max_results=2)  
model = ChatDeepSeek(model="deepseek-chat")
  
agent = create_agent(  
    model=model,  
    tools=[web_search],  # 导入的tavily网页搜索工具
    system_prompt="你是一名多才多艺的智能助手，可以调用工具帮助用户解决问题。"  
) 
 
# 创建回调执行器实例
callback = CostTrackingCallback()  
  
# 运行Agent获得结果  
result = agent.invoke(  
    {"messages": [{"role": "user", "content": "请帮我查询2024年诺贝尔物理学奖得主是谁？"}]},  
    config={"callbacks": [callback]}  
)  
print(result)

"""输出结果为：
[LLM处理完成]
单次Token数据 -> 输入: 1864, 输出: 99
累计Token数据 -> 输入: 1864, 输出: 99
[LLM处理完成]
单次Token数据 -> 输入: 4687, 输出: 98
累计Token数据 -> 输入: 6551, 输出: 197
[LLM处理完成]
单次Token数据 -> 输入: 5608, 输出: 340
累计Token数据 -> 输入: 12159, 输出: 537
{'messages': [HumanMessage(content='请帮我查询2024年诺贝尔物理学奖得主是谁？'【略】
"""
```
这样就完成了token的成本追踪，只需要在成本类中定义价格属性就可以在统计过程中计算开销了。
#### 3. 优化：结合配置管理的成本追踪
当使用多家模型时，如果把模型定价全部放入成本追踪类的话，那么模型就会很庞大，并且每有一次调用，就会连着这些定价信息一起，创建一个庞大的实例。但是其实这些定价信息是可以，且应该共享的，使用[[优化：25-12-31 模块解耦思想的学习过程&实现配置管理和代码分离]]总结的判断方法：**这些值在项目部署到服务器时是否固定？若固定则属于配置数据，若仍然可变则是业务数据**。定价显然应该存放于配置类中：
```python
# 在operations/settings/config.py中定义AI配置类
class AIServerSettings(BaseSettings):  
    """AI相关配置组"""  
    # --- API密钥配置 ---    
    deepseek_api_key: str = Field(...)  
    deepseek_api_base: str | None = Field("https://api.deepseek.com")  
  
    # --- 模型计费配置 ---    
    model_pricing: Dict[str, ModelPricing] = Field(  
        default_factory=lambda:{  
            "deepseek-chat": ModelPricing(  
                input_price_per_1k=0.00055,  
                output_price_per_1k=0.0017,  
            ),  
            "deepseek-reasoner": ModelPricing(  
                input_price_per_1k=0.00055,  
                output_price_per_1k=0.0017,  
            ),  
            # 之后可在此扩展更多模型  
        },  
        description="各模型的Token计价标准"  
    )  
  
    # 获取价格的辅助方法  
    def get_pricing(self, model_name: str) -> ModelPricing:  
        """安全地获取模型定价，如果未配置则返回一个默认值"""  
        return self.model_pricing.get(  
            model_name,  
            ModelPricing(  
                input_price_per_1k=0.0025,  # 设置较高默认值，确保成本被注意到  
                output_price_per_1k=0.01,  
            )  
        )
        
class Settings(DatabaseSettings, AIServerSettings): ...  # 让总配置类继承
```
在使用配置类后，就可以通过总配置类`Settings`的单例`settings`来实现可扩展的成本计算了：
```python
class CostTrackingCallback(BaseCallbackHandler):  
    """使用集中配置的成本追踪回调"""  
    def __init__(self):  
        super().__init__()  
        self.total_input_tokens = 0  
        self.total_output_tokens = 0  
        self.total_cost_usd = 0.0  
    
    def on_llm_end(self, response: LLMResult, **kwargs):  
	    print(f"[LLM处理完成]")  
	    try:  
	        if response.generations:  
	            first_gen = response.generations[0][0]  
	  
	            if hasattr(first_gen, 'message'):  
	                ai_message = first_gen.message  
	  
	                if hasattr(ai_message, 'usage_metadata'):  
	                    tokens = ai_message.usage_metadata  
	                    input_tokens = tokens.get('input_tokens', 0)  
	                    output_tokens = tokens.get('output_tokens', 0)  
	  
	                    # 从配置系统获取价格并计算成本  
	                    # 需要实现一个模型名检测方法  
	                    model_name = self._detect_model_name(ai_message)    
	                    pricing = settings.get_pricing(model_name)  # 使用全局配置  
	  
	                    # 计算本次调用成本  
	                    call_cost = (  
	                        (input_tokens / 1000) * pricing.input_price_per_1k +  
	                        (output_tokens / 1000) * pricing.output_price_per_1k  
	                    )  
	  
	                    # 累加Token和单价到实例变量  
	                    self.total_input_tokens += input_tokens  
	                    self.total_output_tokens += output_tokens  
	                    self.total_cost_usd += call_cost  
	                    print(f"单次成本: ${call_cost:.6f} | 
	                    累计成本: ${self.total_cost_usd:.6f}")  
	  
	                    print(f"单次Token数据 ->
		                     输入: {input_tokens}, 
		                     输出: {output_tokens}")  
	                    print(f"累计Token数据 -> 
			                 输入: {self.total_input_tokens}, 
			                 输出: {self.total_output_tokens}")  
	    except Exception as e:  
	        print(f"解析响应时出错: {e}")
        
    def _detect_model_name(self, ai_message) -> str:    # noqa  
	    """从AIMessage中检测模型名称（根据你的提供商调整）"""  
	    # 示例：从 response_metadata 提取  
	    metadata = getattr(ai_message, 'response_metadata', {})  
	    # 不同提供商字段可能不同，需要适配  
	    name = metadata.get('model_name') or metadata.get('model') or 'unknown'    
	    return name.lower()
```
还是刚才的调用，现在就能够在每条`AIMessage`生成后进行成本的打印：
```text
[LLM处理完成]
单次成本: $0.000266 | 累计成本: $0.000266
单次Token数据 -> 输入: 319, 输出: 53
累计Token数据 -> 输入: 319, 输出: 53
[LLM处理完成]
单次成本: $0.000336 | 累计成本: $0.000601
单次Token数据 -> 输入: 406, 输出: 66
累计Token数据 -> 输入: 725, 输出: 119
{'messages': [AIMessage(content='我来帮您查询北京的天气情况。', 【略】}
```
#### 4. 优化：结合配置管理和日志的成本追踪
上一部分的优化采用的是`print()`函数打印，但是不方便打印到日志，以及print和logger不是一个流，容易出现控制台打印不同步的问题。所以可以使用[[日志进阶]]中的知识，用loguru模块完成成本追踪的日志管理（）：
```python
# operations/loggings/logger.py
class SimpleLogger(object):  
    """简单日志管理器类"""  
    def __init__(self, log_dir: Path | None = None):  
        """  
        初始化日志管理器  
        Args:            
	        log_dir: 日志目录路径，默认为“logs”  
        """        # 确保日志目录存在  
        self.log_dir = log_dir or Path("logs")  
        self.log_dir.mkdir(exist_ok=True)  
  
        # 保存处理器ID以便后续关闭  
        self._handler_ids = []  
  
        # 移除loguru默认处理器  
        loguru_logger.remove()  
  
        # 设置各种处理器  
        self._setup_console()  
        self._setup_file_logging()  
        self._setup_error_logging()  
  
        # 打印初始化信息  
        loguru_logger.info("日志系统初始化完成")  
        loguru_logger.info(f"日志文件保存在: {self.log_dir.absolute()}")  
  
        # 程序退出时自动释放所有处理器  
        atexit.register(self.close)  
  
    def _setup_console(self):  
        """设置控制台日志输出"""  
        console_format = (  
            "<green>{time:YYYY-MM-DD HH:mm:ss}</green> | <level>{level: <8}</level> | " 
            "<cyan>{name}</cyan>:<cyan>{function}</cyan>:<cyan>{line}</cyan> | "        
            "<level>{message}</level>"        
        )  
        handler_id = loguru_logger.add(  
            sys.stderr,  
            format=console_format,  
            level="INFO",  
            colorize=True,  
            backtrace=True,  
            diagnose=False  
        )  
        self._handler_ids.append(handler_id)  
  
    def _setup_file_logging(self):  
        """设置文件日志输出"""  
        file_format = (  
            "{time:YYYY-MM-DD HH:mm:ss} | {level: <8} | "  
            "{name}:{function}:{line} | {message}"        )  
        handler_id = loguru_logger.add(  
            self.log_dir / "app_{time:YYYY-MM-DD}.log",  
            format=file_format,  
            level="DEBUG",  
            rotation="00:00",  
            retention="7 days",  
            compression="zip",  
            encoding="utf-8",  
            enqueue=True  
        )  
        self._handler_ids.append(handler_id)  
  
    def _setup_error_logging(self):  
        """设置错误日志单独输出"""  
        error_format = (  
            "{time:YYYY-MM-DD HH:mm:ss} | {level} | "  
            "{name}:{function}:{line} | {message}\n"  
            "Exception: {exception}"        )  
  
        handler_id = loguru_logger.add(  
            self.log_dir / "error_{time:YYYY-MM-DD}.log",  
            format=error_format,  
            level="ERROR",  
            rotation="00:00",  
            retention="30 days",  
            compression="zip",  
            enqueue=True  
        )  
        self._handler_ids.append(handler_id)  
  
    @staticmethod  
    def log_func(func):  
        """函数调用日志装饰器，用于自动记录函数执行情况、耗时和异常"""  
        @wraps(func)  
        def wrapper(*args, **kwargs):  
            """func为被装饰器装饰的函数"""  
            func_name = func.__name__       # 函数名  
            module_name = func.__module__   # 模块名  
  
            start_time = time.time()  
            loguru_logger.debug(f"开始执行函数: {module_name}.{func_name}")  
            try:  
                result = func(*args, **kwargs)  
                execution_time = time.time() - start_time  
  
                loguru_logger.debug(  
                    f"函数执行完成: {module_name}.{func_name} | "                    
                    f"耗时: {execution_time:.3f}秒"  
                )  
                return result  
            except Exception as e:  
                execution_time = time.time() - start_time  
                loguru_logger.error(  
                    f"函数执行失败: {module_name}.{func_name} | "                    
                    f"耗时: {execution_time:.3f}秒 | "                    
                    f"异常: {type(e).__name__}: {str(e)}"  
                )  
                raise  
        return wrapper  
  
    def close(self):  
        """  
        关闭所有日志处理器，释放文件锁。Windows的文件锁定很严格，
        在没有close()时，文件如果被打开被锁定的话，即使程序结束后，
        文件仍可能锁定几秒  
        而有close()方法时，可以通过调用它显式地立即释放文件锁  
        """        
        # 如果已经关闭，直接返回  
        if not self._handler_ids:  
            return  
  
        # 记录要移除的处理器ID  
        handler_ids_to_remove = list(self._handler_ids)  
  
        # 先清空列表，避免重复操作  
        self._handler_ids.clear()  
  
        # 尝试移除每个处理器  
        for handler_id in handler_ids_to_remove:  
            try:  
                loguru_logger.remove(handler_id)  
            except ValueError as e:  
                # 处理器已经被移除的情况，这是预期的  
                # 可以静默处理，不打印异常  
                if "no existing handler" not in str(e):  
                    # 如果是其他ValueError，打印  
                    print(f"移除处理器 {handler_id} 时出现异常: {e}")  
            except Exception as e:  
                # 其他异常，打印日志  
                print(f"移除处理器 {handler_id} 时出现未知异常: {e}")  
  
    def __enter__(self):  
        """支持上下文管理器"""  
        return self  
  
    def __exit__(self, exc_type, exc_val, exc_tb):  
        """退出上下文时自动关闭"""  
        self.close()  
  
    @property  
    def logger(self):  
        """获取配置好的loguru logger实例"""  
        return loguru_logger
        
        
# operations/loggings/__init__.py
"""  
日志模块入口  
在此处导入`SimpleLogger`类的实例以及日志装饰器  
"""  
from functools import lru_cache  
from pathlib import Path  
from typing import Optional  
from .logger import SimpleLogger  
  
  
@lru_cache(maxsize=None)  
def get_logger(log_dir: Optional[Path] = None) -> SimpleLogger:  
    """  
    获取日志实例（单例模式）  
  
    Args:        log_dir: 日志目录，仅在第一次调用时生效  
  
    Returns:        SimpleLogger实例  
    """    return SimpleLogger(log_dir=log_dir)  
  
  
# 初始化默认日志实例（使用默认配置）  
_default_logger_instance = get_logger()  
  
# 导出用户需要的API  
logger = _default_logger_instance.logger  
log_func = SimpleLogger.log_func  
  
__all__ = ['logger', 'log_func']
```
有了以上的日志功能，项目的各模块就只需要调用它暴露出来的`logger.info()`、`logger.error()`等替代`print()`，使用`log_func`装饰器来装饰函数，让`logger`在函数内也能作用。
所以配备了日志功能的成本追踪回调执行器，就能够在`logger.info("本次成本为：...")`之类的语句中，不仅能让成本打印到控制台，还能够写入文件，且是否写入、格式如何都能实现自定义。