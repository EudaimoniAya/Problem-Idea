## 一、YAML文件
YAML是一种文本数据格式，常用于配置文件（如K8S、SpringBoot等），扩展名为`.yaml`（或是缩写`.yml`，因为早期Windows系统，比如DOS对文件扩展名长度有限制，只支持3个字符）。
![[Pasted image 20260309171056.png]]
YAML相比于Json拥有更高的易读性，假设有Json格式数据：
```json
{
	"name": "张三",
	"age": 16,
	"height": 180,
	"class": "高二(3)班",
	"hobbies": [
		"篮球",
		"音乐",
		"编程"
	]
}
```
将其转换为YAML格式则是：
```yaml
name: 张三
age: 16
height: 180
class: 高二(3)班
hobbies:
  - 篮球
  - 音乐
  - 编程 
```
### 1.1 对象、数组
![[Pasted image 20260309163747.png]]
YAML的缩进只允许使用空格，tab制表符不被允许（除非使用能够将tab键输入改为2个或4个空格的代码编辑器）
![[Pasted image 20260309164119.png]]
另外，YAML行内可设置JSON格式数据（但必须一行写完），如以下的语法正确：
```yaml
list object:
  - {"key 1-1":"value", "key 1-2":"value"}
  - {"key 2-1":"value 2"}
```
### 1.2 字符串
![[Pasted image 20260309164430.png]]
因为YAML字符串无需双引号，所以`"`也无需转义，如果字符串以空格开头或空格结尾，则需要双引号或单引号包裹。但是单双引号在YAML中有区别：单引号内容不会转义，如` value\n `在JSON中应该写成`' value\\n '`；双引号内容会转义，等价于JSON的双引号，如`" value\n "`在JSON中也是`" value\n "`
![[Pasted image 20260309165051.png]]
![[Pasted image 20260309165433.png]]
![[Pasted image 20260309165715.png]]
![[Pasted image 20260309165519.png]]
### 1.3 数字、布尔值、Null
和JSON基本一致
### 1.4 其他类型和tag标签
![[Pasted image 20260309170042.png]]
![[Pasted image 20260309170114.png]]
### 1.5 注释和空行
![[Pasted image 20260309170141.png]]
YAML和JSON的空行都会被忽略
### 1.6 锚点/引用
![[Pasted image 20260309170242.png]]
并不一定好用，当定义的引用多了会难以阅读
### 1.7 多文档
![[Pasted image 20260309170348.png]]
一般不用，因为`---`分隔不方便查看文档切分情况。

## 二、GitHub Actions
### 2.1 介绍
GitHub Actions是GitHub官方提供的CI/CD服务，能够通过YAML文件自动化软件开发工作流。其本质就是一个事件驱动的自动化引擎，通过YAML文件中`on`下面指定的事件作为触发，安排多个动作按顺序编排成工作流，最后这些任务将在GitHub提供的运行器Runner上执行。类似于n8n的可视化节点编排，但是GitHub Actions通过YAML代码来定义。
![[Pasted image 20260310092334.png]]
一个GitHub Actions的YAML文件存放在项目根目录的`.github/workflows`下面，文件格式如下：
```yaml
name: CI  # 工作流名称
on:       # 触发事件
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  # workflow_dispatch可以再GitHub用户界面手动触发工作流
  workflow_dispatch:
   

jobs:
  test:    # job ID
	# 每个job运行在全新环境中，runner每次都是新容器
    runs-on: ubuntu-latest  
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'
      - name: Run tests
        run: |
          pip install -r requirements.txt
          pytest
```
其中，`name`字段指定一个**工作流的名字**；`on`下的代码块用于**指定某个事件（GitHub Event）作为触发器**，如`workflow_dispatch`允许用户从GitHub用户界面手动触发工作流程；`jobs`是**一组在同一个Runner上（由`runs-on`指定）并行执行的step**。在每个`job`的配置中，作业`steps`在同一环境内按顺序执行。
![[Pasted image 20260310093153.png]]
**每一步要么执行shell命令`run`（自定义逻辑），要么重用现有操作`uses`**（调用现成的action，类似于n8n会提供各种节点）
![[Pasted image 20260310094748.png]]
设定`workflow_dispatch`手动触发工作流：
![[Pasted image 20260310074549.png]]
运行Hello World工作流（打印一句话到控制台上）的结果：
![[Pasted image 20260310093952.png]]

### 2.2 基本用法
#### 2.2.1 工作、依赖关系和触发器
##### 工作：`job`
GitHub Actions有这样一个流程：用户正常开发并编写工作流文件，上传代码到远程仓库，然后GitHub会根据`.github/workflows/`下的`.yaml`或`.yml`文件执行安排好的工作流：根据不同的事件（如提交、合并请求、上图的手动执行），触发`jobs`定义的工作。
##### 依赖关系：`needs`
另外`jobs`的工作执行可以用`needs`来实现同步，而未被同步约束的工作能够被并行执行：
```yaml
# 略

jobs: 
  job-1: 
    runs-on: ubuntu-24.04 
    steps: 
      - run: echo "A job consists of" 
      - run: echo "one or more steps" 
      - run: echo "which run sequentially" 
      - run: echo "within the same compute environment" 
  job-2: 
    runs-on: ubuntu-24.04 
      steps: 
        - run: echo "Multiple jobs can run in parallel" 
  job-3: 
    runs-on: ubuntu-24.04 
    needs: 
      - job-1 
      - job-2 
    steps: 
      - run: echo "They can also depend on one another..." 
  job-4: 
    runs-on: ubuntu-24.04 
    needs: 
      - job-2 
      - job-3 
    steps: 
      - run: echo "...to form a directed acyclic graph (DAG)"
```
GitHub会自动简化工作流定义的有向无环图（DAG），如`job-4`需要`job-2`和`job-3`完成，而`job-3`本来就需要`job-2`完成，所以`job-4`前置任务只有`job-3`，对应的DAG为：
![[Pasted image 20260310075605.png]]
##### 触发器：`on`
事件决定工作流何时运行，最常见的触发条件是：`push`（每次git提交到远程仓库）、`pull_request`（每次PR）、`workflow_dispatch`（手动执行）、`schedule`（定时）如：
```yaml
on: 
  push: 
    branches: 
      - "example-branch/*" 
    
  pull_request: 
    paths: 
      - "03-core-features/filters/*.md" 
      - "!03-core-features/filters/*.txt" 
  
  schedule: 
    - cron: "0 0 * * *" # Midnight UTC
       
  workflow_dispatch:
```
其中，过滤器`branches`可以限制触发的分支，过滤器`paths`能够在不相关文件更改时跳过测试代价高昂的作业，触发器`schedule`使用corn调度表达式，即使当天没有提交也会运行，手动调度会在用户界面提供一个运行工作流程的按钮，用于临时构建、紧急修复或演示。如给PR请求加上过滤器`paths`，限定对应目录下会对`.md`相关的修改执行工作流，而忽略`.txt`文件的。
#### 2.2.2 环境变量和范围
![[Pasted image 20260310175048.png]]
上图示例中，工作流范畴的环境变量`WORKFLOW_VAR`值为`I_AM_WORKFLOW_SCOPED`；工作范畴的环境变量`JOB_VAR`的值为`I_AM_JOB_SCOPED`；步骤范畴的环境变量`STEP_VAR`的值为`I_AM_STEP_SCOPED`。
所以三个步骤的结果输出应该如下：
```text
# job 1 step 1
WORKFLOW_VAR: I_AM_WORKFLOW_SCOPED
JOB_VAR:      I_AM_JOB_SCOPED
STEP_VAR:     I_AM_STEP_SCOPED

# job 1 step 2
WORKFLOW_VAR: I_AM_WORKFLOW_SCOPED
JOB_VAR:      I_AM_JOB_SCOPED
STEP_VAR:     <UNSET>

# job 2
WORKFLOW_VAR: I_AM_WORKFLOW_SCOPED
JOB_VAR:      <UNSET>
STEP_VAR:     <UNSET>
```
实际执行结果如下：
![[Pasted image 20260310180053.png]]
![[Pasted image 20260310180113.png]]
#### 2.2.3 在步骤和工作之间传递数据
![[Pasted image 20260310180342.png]]
```yaml
name: Passing Variables Between jobs

on:
  workflow_dispatch:
  
jobs:
  producer:   # 第一个作业的名称
    runs-on: ubuntu-24.04
    # 定义该作业的输出变量，它可被依赖它的其它作业引用
    outputs:
      foo: ${{ steps.generate-foo.outputs.foo }}
    steps:
      - name: Generate and export foo
        id: generate-foo # 方便后续步骤或作业输出引用
        run: |
          # 在shell中定义局部变量`foo`，值为`bar`
          foo=bar
          # 1) Step output (for job output)
          echo "foo=${foo}" >> "$GITHUB_OUTPUT"
          # 2) Job-scoped environment variable
          echo "FOO=${foo}" >> "$GITHUB_ENV"
      - name: Inspect values inside producer
        run: |
          echo "FOO (set via GITHUB_ENV):   $FOO"
          echo "foo (step output):          ${{ steps.generate-foo.outputs.foo }}"
          
  consumer:   # 第二个作业的名称，用于获取输出
    runs-on: ubuntu-24.04
    needs: producer
    steps:
      - name: Inspect values inside consumer (note FOO is unset)
        run: |
          echo "Value from producer:        ${{ needs.producer.outputs.foo }}"
          echo "FOO in consumer:            ${FOO:-<UNSET>}"
```
其中`outputs` 定义该作业的输出变量，这些变量可以被依赖它的其他作业（如`consumer`）通过`needs`上下文引用。而`foo: ${{ steps.generate-foo.outputs.foo }}` 声明一个名为`foo`的输出，它的值来自`generate-foo`步骤的输出（`steps.generate-foo.outputs.foo`）。注意，这里`steps.generate-foo`是后面步骤的ID。
在第一个`run`的命令中：
* `echo "foo=${foo}" >> "$GITHUB_OUTPUT"`表示将`foo=bar`写入到`GITHUB_OUTPUT`**环境变量指定的文件中**，这是GitHub Actions设置步骤输出的标准方式。`GITHUB_OUTPUT`是一个**临时文件路径，写入的内容会被解析为步骤的输出键值对**
* `echo "FOO=${foo}" >> "$GITHUB_ENV"`表示将`FOO=bar`写入到`GITHUB_ENV`环境变量指定的文件中，写入后该环境变量会在**当前作业的后续步骤中自动生效**
![[Pasted image 20260310184054.png]]
所以在第二个`run`命令中：
* `echo "FOO (set via GITHUB_ENV): $FOO"`: 打印环境变量`FOO`的值（通过`$FOO`引用），由于上一步通过`GITHUB_ENV`设置了`FOO=bar`，这里会输出`bar`
* `echo "foo (step output): ${{ steps.generate-foo.outputs.foo }}"`: 使用表达式`${{ steps.generate-foo.outputs.foo }}`引用第一个步骤的输出，预期输出`bar`。
![[Pasted image 20260310184112.png]]
在第三个`run`命令中：
* `echo "Value from producer: ${{ needs.producer.outputs.foo }}"`: 通过`needs.producer.outputs.foo`引用`producer`作业定义的输出`foo`，这里会输出`bar`，证明作业输出成功跨作业传递
* `echo "FOO in consumer: ${FOO:-<UNSET>}"`: 打印环境变量`FOO`的值，**但因为`FOO`只在`producer`作业内部通过`GITHUB_ENV`设置，并未传递到`consumer`，所以`FOO`未定义**。这里使用`${FOO:-<UNSET>}`表示如果`FOO`未设置，则输出`<UNSET>`，结果将是`<UNSET>`，说明环境变量不跨作业共享
![[Pasted image 20260310184024.png]]
**总结**：作业输出通过步骤输出`GITHUB_OUTPUT`（`>>`写入）和作业的`outputs`字段定义，**适用于跨作业传递数据**；而环境变量通过`GITHUB_ENV`设置，只作用于当前作业内的后续步骤，**不会自动传递到其他作业**。
具体做法是：
* 添加`key=value`追加进`$GITHUB_ENV`中，设置环境变量
* 添加`key=value`追加进`$GITHUB_OUTPUT`中，设置步骤对应的`id`，然后就可以将它们作为作业输出公开，并使用**上下文**从下游作业中使用它们的`needs`对应的作业，通过`needs.<job>.outputs.<key>`调用
上下文是GitHub Actions中的一组预定义的对象，它们提供了工作流运行时的各种信息（如触发事件、仓库信息、步骤输出、环境变量等）。
#### 2.2.4 密钥和变量
另外，对于隐私信息（如数据库密码、凭证等）或变量，需要在GitHub的项目页面手动设置
![[Pasted image 20260310190629.png]]
`${{}}`调用，如`${{ secrets.EXAMPLE_REPOSITORY_SECRET }}`：
![[Pasted image 20260310190757.png]]
### 2.3 进阶用法
#### 2.3.1 运行器类型和执行环境
每个作业最终都是要在服务器上运行的，`runs-on`的值决定了作业可用的OS、架构、CPU和内存，运行器的选择有三种：
* GitHub托管的运行器：开箱即用，有一定免费额度
* 第三方托管运行器：像Namespace这种托管高性能运行器的服务提供商，通常能够以更低的成本实现更快的构建速度，并且具有技术支持和提升用户体验的功能。
* 自托管运行器：需要自行运行底层计算资源（如使用Kubernetes上的Actions Runner Controller或RunsOn等工具）实现私有网络执行，但是需要自行管理基础设施
另外还可以通过**添加容器或自定义标签来进一步定制环境**：
```yaml
alpine-container-on-github-hosted-ubuntu-vm: 
  name: Alpine container on Ubuntu VM 
  runs-on: ubuntu-24.04 
  container:   # 添加容器
    image: alpine:3.20 
  steps: 
    - name: Show runner info 
      run: | 
        echo "Hello from ${{ runner.os }}-${{ runner.arch }}" 
        echo "Runner name (type): ${{ runner.name }}" 
        echo "Container image: $(grep PRETTY_NAME /etc/os-release)"
```
