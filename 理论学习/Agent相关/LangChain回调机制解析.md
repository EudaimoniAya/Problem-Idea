学习目的：[[优化：26-01-11 对AI调用进行成本追踪的实现和优化记录&Callback回调机制的深入分析]]
#### 1. 相关类的源码分析
##### a. `langchain_core/callbacks/base.py`文件结构
该文件为管理器抽象基类`BaseCallbackManager`和执行器抽象基类（同步`BaseCallbackHandler`；异步`AsyncCallbackHandler`）的定义。虽然该文件中存在9个类，但它们的关系很简单：
* **6个`Mixin`类**：只提供特定功能，但不是为了独立使用，而是为了让其他类继承这些功能，包括6个：`RetrieverManagerMixin`、`LLMManagerMixin`、`ChainManagerMixin`、`ToolManagerMixin`、`CallbackManagerMixin`、`RunManagerMixin`，它们分别提供了六个方面的功能；
* **2个执行器基类**：继承了这六个`Mixin`类的`BaseCallbackHandler`（回调执行器基类）、继承了`BaseCallbackHandler`的`AsyncCallbackHandler`（异步回调执行器）
* **1个管理器基类**：继承了`CallbackManagerMixin`的`BaseCallbackManager`
###### 1) `RetrieverManagerMixin`
负责为检索器提供回调管理，包含2个方法：
* `on_retriever_error`：检索知识库出错事件；
* `on_retriever_end`：检索知识库完成时事件。
###### 2) `LLMManagerMixin`
负责为LLM提供回调管理，包含3个方法：
* `on_llm_new_token`：仅限流式输出时起作用，生成新token的事件。
```python
"""Run on new output token. Only available when streaming is enabled.  
For both chat models and non-chat models (legacy LLMs)"
```
* `on_llm_end`：调用LLM完成事件；
* `on_llm_error`：调用LLM出错事件。
###### 3) `ChainManagerMixin`
负责为链提供回调管理，包含4个方法：
* `on_chain_end`：调用链结束事件；
* `on_chain_error`：调用链出错事件；
* `on_agent_action`：Agent执行动作事件；
* `on_agent_finish`：Agent执行动作完成事件。
###### 4) `ToolManagerMixin`
负责为工具提供回调管理，包含2个方法：
* `on_tool_end`：调用工具结束；
* `on_tool_error`：调用工具出错。
###### 5) `CallbackManagerMixin`
回调管理器的功能，包含5个方法：
* `on_llm_start`：开始调用大模型事件（单次调用）；
* `on_chat_model_start`：开始调用聊天模型事件；
* `on_retriever_start`：开始调用检索器事件；
* `on_chain_start`：开始调用链事件；
* `on_tool_start`：开始调用工具事件；
###### 6) `RunManagerMixin`
运行管理器的功能，包含3个方法：
* `on_text`：运行任意文本；
* `on_retry`：重试事件；
* `on_custom_event`：重写它以自定义一个事件。
###### 7) `BaseCallbackHandler`
LangChain的回调执行器基类，**它继承了六个`Mixin`类**，包含了两个字段`raise_error`（是否发起异常）和`run_inline`（是否启用同步，默认为`False`异步执行），以及7个`ignore_xxx() -> bool`方法用于选择性忽略一些回调：`llm`、`retry`、`chain`、`agent`、`retriever`、`chat_model`、`custom_event`
###### 8) `AsyncCallbackHandler`
LangChain的异步回调执行器基类，虽然占据了将近一半的文件内容，但是它内部只是使用`async def`把 1) ~ 6) 的所有事件重写了一遍。
###### 9) `BaseCallbackManager`
LangChain的基类回调管理器，继承自`CallbackManagerMixin`，包含了各种对`metadata`、`tags`、`handler`的添加、移除、设置操作方法，和如`merge`、`copy`这种对管理器的合并、复制方法。

其中，在源码中可以发现每个事件存在两个`UUID`类型的参数：`run_id`和`parent_run_id`。因为在Agent调用中，比如Agentic RAG，它在进行检索时需要调用工具，那么就有这样的事件流程： `on_agent_action` -> `on_tool_start` -> `on_retriever_start` -> `on_retriever_end` -> `on_tool_end` -> `on_agent_finish`，可见事件的执行是有层级的，这个过程会像数一样展开，使**以`run_id`和`parent_run_id`构成了一个”父节点表示法“的事件树**。
在”观察者模式“的视角下：运行中的`LLM`、`Chain`、`Retriever`、`Agent`、`Tool`等组件就是这个系统所**观察的对象**；子类`CallbackHanler`就是观察某几个组件的事件的**观察者**；子类`CallbackManager`负责集成所有执行器，并在事件发生时，及时地发送给它们，它是**作为调度中心**存在的。
**总之这个基类文件做的事情就只有两个：一个是按功能使用6个`Mixin`类定义了一堆事件（LangSmith可视化测试中模型图的边），涵盖了agent调用这套完整过程中各环节；二是使用管理器和执行器将这些事件按需求集成。**

##### b. `langchain_core/callbacks/manager.py`的`CallbackManager`类
`CallbackManager`是`BaseCallbackManager`的子类，并且是LangChain内部提供的**核心默认实现**。在**a.** 中梳理过：`BaseCallbackManager`是`CallbackManagerMixin`的子类，它增加的是管理层面的方法，如设置、移除、添加处理器、元数据等操作。所以`CallbackManager`同时拥有管理层面的能力和`CallbackManagerMixin`的能力。
**Q：** 在前部分的源码分析中可以看到，在观察者模式下，**被观察的对象`Retriever`、`Chain`、`Tool`、`LLM`均只有`on_xxx_error`和`on_xxx_end`事件**（Agent除外，它使用`on_agent_action`和`on_agent_finish`），**而它们的`on_xxx_start`事件则统一由`CallbackManagerMixin`类进行管理**（新增了一个聊天模型`on_chat_model_start`事件）。
* 为什么要这么设计？这就好像回调管理器在集中控制所有观察对象的开始。
* 为什么`LLMManagerMixin`类没提供聊天模型相关的`end`和`error`事件，也没单独设置聊天模型的类，但是`CallbackManager`类中却有聊天模型的`start`事件？
**A：** `on_xxx_start`事件另外被集中管控是因为，**这些`start`事件负责创建`RunManager`系列或其子类的作用**。这与`langchain_core/callbacks/manager.py`中存在的大量`BaseRunManager`的子类相关（**`BaseRunManager`通过继承`RunManagerMixin`得到**）：

| 同步                                      | 异步                                    | 组合                              |
| --------------------------------------- | ------------------------------------- | ------------------------------- |
| `RunManager`                            | `AsyncRunManager`                     | `BaseRunManager`                |
| `ParentRunManager`（同步异步一起简称为父Run，Run同理） | `AsyncParentRunManager`               | `RunManager`/ `AsyncRunManager` |
| `CallbackManagerForLLMRun`              | `AsyncCallbackManagerForLLMRun`       | Run+LLM                         |
| `CallbackManagerForChainRun`            | `AsyncCallbackManagerForChainRun`     | 父Run+Chain                      |
| `CallbackManagerForToolRun`             | `AsyncCallbackManagerForToolRun`      | 父Run+Tool                       |
| `CallbackManagerForRetrieverRun`        | `AsyncCallbackManagerForRetrieverRun` | 父Run + Retriever                |
| `CallbackManager`                       | `AsyncCallbackManager`                | Callback基类                      |
| `CallbackManagerForChainGroup`          | `AsyncCallbackManagerForChainGroup`   | Callback                        |
表格中的16个类加上`BaseRunManager`一共17个类就是`manager.py`中的全部类，同步类大多继承于`RunManager`和`ParentRunManager`，异步类大多基于`AsyncRunManager`和`AsyncParentRunManager`。其中被观察的`Retriever`、`Chain`、`Tool`、`LLM`相关回调管理器由Run或父Run与对应`Mixin`类组合得到。

所以**表格中的前6行能够归为一类**，现在主要分析`CallbackManager`：
由于继承了回调管理器基类的基类（回调`Mixin`），所以**该类实现了基类定义的5个`on_xxx_start`，涵盖了`Retriever`、`Chain`、`Tool`、`LLM`、`ChatModel`这5个对象**，另外还新增了支持自定义事件的`on_custom_event`，和用于配置的类方法`configure`。**而这5个被观察对象分别需要其他的类支持，每个`start`事件最终都会返回对应的被观察对象的回调函数类**：

| 同步事件                  | 返回的类                                  | 描述                          |
| --------------------- | ------------------------------------- | --------------------------- |
| `on_llm_start`        | `CallbackManagerForLLMRun`            | 同步LLM事件由同步的运行时管理器+LLM混合类支持  |
| `on_chat_model_start` | `CallbackManagerForLLMRun`            | 同步聊天模型事件由同步的管理器+LLM混合类支持    |
| `on_chain_start`      | `CallbackManagerForChainRun`          | 同步Chain事件由同步的管理器+Chain混合类支持 |
| `on_tool_start`       | `CallbackManagerForToolRun`           | 同步LLM事件由同步的管理器+LLM混合类支持     |
| `on_retriever_start`  | `CallbackManagerForRetrieverRun`      | 同步Tool事件由同步的管理器+Tool混合类支持   |
| **异步事件**              | **返回的类**                              | \                           |
| `on_llm_start`        | `AsyncCallbackManagerForLLMRun`       | 异步LLM事件由异步的运行时管理器+LLM混合类支持  |
| `on_chat_model_start` | `AsyncCallbackManagerForLLMRun`       | 异步聊天模型事件由异步的管理器+LLM混合类支持    |
| `on_chain_start`      | `AsyncCallbackManagerForChainRun`     | 异步Chain事件由异步的管理器+Chain混合类支持 |
| `on_tool_start`       | `AsyncCallbackManagerForToolRun`      | 异步LLM事件由异步的管理器+LLM混合类支持     |
| `on_retriever_start`  | `AsyncCallbackManagerForRetrieverRun` | 异步Tool事件由异步的管理器+Tool混合类支持   |
**总结一下，就是该文件的类结构为：`BaseRunManager`作为基类，构建同/异步的`RunManager`和`ParentRunManager`用于继承（根据情景是否会生成子事件选择继承的类，因为父Run仅新增了`get_child`这个方法，返回一个回调管理器），然后基于这两对类又得到了对于不同观察对象的回调管理器类，这些类分别作用于同/异步`CallbackManager`中的对应`on_xxx_start`方法。另外还有一对针对同/异步的ChainGroup回调函数，用于支持工作流**。
所以存在一个这样的流程：`CallbackManager`管理**被观察对象的`start`方法，这些方法会动态地生成对应的`CallbackForXXX`实例**，对于LLM或对话模型，这些实例被存储在`managers`这个列表中（初始为空）作为返回值；对于Chain、Tool、Retriever则是直接返回实例。
【但是这些都是静态结构，如何理解它们在一套调用过程中的运作机制（比如说最后这些实例谁来用），**需要考察`Runnable`的`invoke()`方法做了什么**】
#### 2. 关于回调机制流程的问题与总结
**最初的浅显理解**：当`Runnable.invoke()`执行时，LangChain框架会将传入的`callbacks`列表加入`CallbackManager`，其中`callbacks`是`config`的参数，LangChain对其示例为：
```python
from langchain_core.tracers import ConsoleCallbackHandler  
chain.invoke(..., config={"callbacks": [ConsoleCallbackHandler()]})
```
在执行过程中，这个管理器会在关键时刻触发事件，然后将它们分发给注册的所有`CallbackHandler`，由它来完成业务逻辑，由此可以完成成本追踪等业务逻辑。
##### 深入提问
**Q1：** **`CallbackManager`何时创建**？在下面的”Agent添加回调执行器实现成本追踪“部分，使用了基于`BaseCallbackHandler`的`CostTrackingCallback`，通过`invoke`的`config`参数传入了回调执行器，LangChain框架再把执行器注册进`CallbackManager`里面。那么既然它和loguru是同一设计模式，那么它**是否像loguru一样存在一个全局的单例Logger，LangChain存在一个全局单例`CallbackManager`**？这是我最初查看`base.py`和`__init__.py`的目的，想查看是否有一处是像日志一样，在文件最末尾完成`CallbackManager`的实例化。
**A1：** `CallbackManager`是在**每次运行时动态创建的（也可以使用环境变量设置的默认回调处理器）**，它不像loguru那样有一个始终存在的单例`Logger`。调用`invoke()`并传入`callbacks`时，LangChain会将传入的执行器和所有预定义的默认处理器合并，并**为这次调用实例化一个`CallbackManager`对象，它只存在于这次调用期间，一旦结束就被回收**。（为什么不是全局单例？①并发安全：每个用户请求需要独立的回调跟踪；②上下文隔离：不同调用可能有不同配置，如有的要日志，有的还要附加成本追踪）

**Q2：** 分析：当注册执行器成功后，`CallbackManager`持续运行，和`Logger`一样观察着目标，当`CallbackManager`内部定义的`start`方法发现执行的进度时（这里猜想这个进度发现来自于`AgentState`内部的`jump_to`字段，它指定了下一状态。即使猜想错误，`CallbackManager`也一定会是通过图的某个提供状态的字段来获取当前事件发生状态的。至于为什么是图，是因为`create_agent`方法返回的就是一个图，这也是为什么说LangGraph在1.0版本就变成了LangChain的底层）【猜想错误】，它就调用方法（发布事件），完成业务逻辑。这就是回调，`CallbackManager`无需考虑执行器内部细节，它只需要提供几个操作地点（这里猜想就是指与执行层的LLM客户端相关的概念），这些地点就是指各种的`on_xxx_start`、`on_xxx_end`等事件，相当于开发人员把方法作为参数传入业务逻辑内部。
问题：但是在`CallbackManager`的源码中查看时，所有的`strat`方法都返回了一个笔记中提到的Run/父Run+`Mixin`的继承组合类，那么这些组合类是干什么的？为什么要返回它们？**是谁在需要这些方法（事件），返回的这些事件调用者又会拿它干什么**？**应该如何将这些`RunManager`的信息传递给执行器**？**以及`RunManager`系列类是做什么的**？
**A2：** `CallbackManager`类似于注册中心，它知道有哪些处理器，**而`RunManager`是执行上下文，它知道当前运行的具体消息**。当`start`返回一个专门用于该次运行的特定类型`RunManager`实例，这代表了回调管理器发现了某个特定事件，并**将当前运行的特定信息绑定到这个实例上**（如`run_id`、`parent_run_id`、输入参数等）作为返回值。返回值的特定`RunManager`子类实例将会**提交给调用`invoke()`方法LangChain组件**。
所以，当调用`agent.invoke()`时，LangChain会基于`callbacks`列表创建`CallbackManager`实例（**A1**），然后调用`callback_manager.on_chain_start()`，得到一个`CallbackManagerForChainRun`，LangChain框架将这个`RunManager`子类实例交给Agent组件。这第一个`RunManager`子类实例由框架创建，后续的所有`RunManager`都是从它发散出来。这个子类实例就能够被分发给对应的回调执行器——继承了所有`Mixin`类的`BaseCallbackHandler`实例。当需要创建子事件时，就执行`get_child()`方法，获得一个初始的`CallbackManager`，然后再通过子`CallbackManager`触发`on_tool_start`，返回一个专门为Tool运行的`RunManager`。
至于状态的传递，在LangChain 1.0的图执行中，**状态通过边在节点间传递，回调事件通过`RunManager`在组件之间传递**。**两者并行但是独立，因为状态包含业务数据，回调包含元数据**。当图执行到某个节点时，该节点先收到输入状态，然后从父节点处获得或创建新的`RunManager`，触发相应事件的开始并执行业务逻辑，直到触发事件结束后输出新的状态。**所以回调机制的事件发现并不直接来源于状态，而是来源于组件执行代码的主动触发**。
```text
    执行流程
       ↓
┌─────────────────────┐
│   组件执行代码       │
│                     │
│ 1. 读取输入状态      │
│ 2. 触发回调开始事件  │ ←── 回调系统入口
│ 3. 执行业务逻辑      │
│ 4. 更新输出状态      │
│ 5. 触发回调结束事件  │ ←── 回调系统出口
└─────────────────────┘
       ↓
    状态流转
```

**Q3：** 当回调函数在执行时，会存在生命周期闭包的关系，比如当使用Agentic RAG时，检索器作为工具调用，那么Tool的生命周期就会包含Retriever的生命周期：`on_tool_start` -> `on_retriever_start` -> `on_retriever_end` -> `on_tool_end`。那么，比如检索器可以看作是工具调用的一个子事件，如果**Q2**中的猜想合理的话，只靠状态就应该能够完成业务逻辑了，为什么还需要这个`ParentRunManager`以及它唯一新增的`get_child()`方法？以及为什么还要管理一个事件树来实现事件有序分发？
**A3：** 一个工具可能并行调用多个检索器，状态只能够告诉LangChain框架”发生了什么“，毕竟`AgentState`中的`jump_to`字段的注释表示该字段的值就像是一张写了下一步指示的便签条，仅作为临时值，进入下一状态就要换新的值了（而另外两个字段，一个是结构化输出，一个是笼统的消息列表）。可见它难以描述”何时发生事件“和”上下文关系“【这一部分未看源码，只是推论，真实性存疑】所以如果不基于事件树，那么在检索器识别需要回滚，或复杂工作流中出现嵌套调用智能体，进行成本追踪时这些场景就难以更细粒度地分析。

##### 回调机制的最终梳理
1. LangChain为回调管理器`CallbackManager`的设计是动态的，灵活的，生命周期仅在调用功能期间维持。所以它能够支持对每个组件的调用维持一个回调管理器（由根管理器不断地创建子管理器），而非全局单例，这样既提高并行度也避免上下文管控问题。
2. 回调管理器能够在运行时，**子组件自己不会触发开始事件，而是由父组件在调用子组件前主动触发**（默认未实现所以一般无事发生）以此”发现事件“，并返回运行时管理器`RunManager`的某个子类，这个子类被绑定了运行的特定信息，并能够被回调执行器`BaseCallbackHandler`子类实例执行。从而实现了管理器获取到事件 --> 事件转发给执行器。
3. 当出现组件需要调用组件，即某个`RunManager`子类实例（且一定是`ParentRunManager`子类）下面又会出现`RunManager`子类实例的情况时，就需要使用它继承的`get_child`方法新构造一个子类专属的`CallbackManager`实例，至于它的事件，由它对应组件自行调用。只需要最后下层组件执行完后，可以通过调用栈自然返回上层组件`RunManager`即可。
4. 另外，**`CallbackManager`无需传给子组件，因为子组件不需要它，本该属于子组件的`CallbackManager`的`start`方法被父组件帮忙执行了，从头到尾子组件就没碰过它。而`CallbackManager`只管开始，结束由真正属于子组件的`RunManager`中的`end`方法执行**。也就是说，回调管理器是”生“的工具，运行管理器是”命“的载体，由父组件决定”出生“，由组件自身决定”死亡“，每个继承了`ParentRunManager`的类对应的组件，都可以通过`get_child()`方法实现”生育“。
###### 下面是一套Agentic RAG流程：Input -> ( AI -> Tool -> Retriever -> AI ) -> Output
* Agent调用方法：`agent.invoke(..., config={"callbacks": [handler]})`
* LangChain尝试查找环境变量中设置的默认回调处理器，并将`callbacks`列表传回的处理器合并，为这次调用**创建根`CallbackManager`实例**（简称为`root_callback_manager`），并将处理器都注册到`root_callback_manager`
* **框架自动调用`root_callback_manager.on_chain_start()`** 返回一个`CallbackManagerForChainRun`实例（简称`chain_run_manager`），绑定运行时信息，假设`run_id = 1`
   * ====== Agent调用LLM =======
   * Agent需要调用LLM进行分析，通过`chain_run_manager.get_child()`创建一个新的`CallbackManager`实例（简称为`llm_child_callback_manager`）
   * **父组件Agent执行`llm_child_callback_manager.on_llm_start()`** 返回一个`CallbackManagerForLLMRun`实例（简称`llm_run_manager`）。绑定子组件的运行时信息，假设`run_id = 2`，**此时的`parent_run_id = 1`**
   * Agent将`llm_run_manager`作为配置的一部分传递给LLM组件
   * LLM组件执行完毕，调用`llm_run_manager.on_llm_end()`
   * **控制流通过函数调用栈返回到父组件（Agent）的代码继续执行**
   
   * ====== Agent调用Tool ========
   * Agent根据LLM返回结果希望调用工具来检索知识库，通过`chain_run_manager.get_child()`创建一个新的`CallbackManager`实例（简称为`tool_child_callback_manager`）
   * **父组件Agent执行`tool_child_callback_manager.on_tool_start()`** 返回一个`CallbackManagerForToolRun`实例（简称`tool_run_manager`），绑定运行时信息，假设`run_id = 3`，**此时的`parent_run_id = 1`**
   * Agent将`tool_run_manager`作为配置的一部分传递给Tool组件
   
    * ====== Tool调用Retriever ======= 
    * Tool组件希望执行Retriever，通过`tool_run_manager.get_child()`为检索器的调用创建一个新的`CallbackManager`实例（简称为`retriever_child_callback_manager`）
    * **父组件Tool执行`retriever_child_callback_manager.on_retriever_start()`** 返回一个`CallbackManagerForRetrieverRun`实例（简称`retriever_run_manager`），绑定运行时信息，假设`run_id = 4`，**此时的`parent_run_id = 3`**
    * Tool将`retriever_run_manager`作为配置的一部分传递给Retriever组件
    * Retriever组件执行完毕，调用`retriever_run_manager.on_retriever_end()`
    * **控制流通过函数调用栈返回到父组件（Tool）的代码继续执行**
    
   * Tool组件执行完毕，调用`tool_run_manager.on_tool_end()`
   * **控制流通过函数调用栈返回到父组件（Agent）的代码继续执行**
   
* Agent组件执行完毕，调用`chain_run_manager.on_chain_end()`
* Agent为根节点，上层无父组件，所以`invoke()`方法执行完毕