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
![[Pasted image 20260315115614.png]]
#### 2.3.2 使用Artifacts输出持久化
GitHub Actions的Runner在执行作业期间创建的所有内容都会在作业完成后消失，而Artifacts则通过将文件存储在GitHub的对象中，提供了持久化途径，以供后续使用：
![[Pasted image 20260315122558.png]]
`artifact-producer`作业分为两步：第一步是将两句话写入`artifact.txt`文件中，第二步是通过`actions/upload-artifact` 动作将 `artifact.txt` 上传为 artifact，并命名为 `example-artifact`。
`aertifact-consumer`作业（事先通过`needs`声明依赖）也分为两步：第一步是使用`actions/download-artifact`动作下载名为`example-artifact`的artifact，它会将文件恢复到当前工作目录。第二步执行`cat artifact.txt`查看文件内容，验证下载是否成功。

> [!问题] 哈希码如何作用于Actions？
> 在`actions/upload-artifact`后面加上`@<commit-hash>`是指定了该Action的一个**具体版本**，但是这里的版本不是像`v4`这种语义化标签，而是40位的提交哈希。因为标签可变，维护者可以将其重新指向另一个提交，如`v4`可能会在`v4.2`更新后，从`v4.1`指向`v4.2`，当工作流依赖`v4`时可能结果就会随着更新变动。GitHub官方推荐**在生产工作流中使用完整的提交SHA来锁定Action版本，以确保稳定性和可重现性**。
> 对于这个哈希码，可以先使用标签`v4`进行开发和测试，确认无误后，再将标签替换为固定的提交哈希，以保证长期稳定性。哈希值需要先找到对应的版本：
> ![[Pasted image 20260315131110.png]]
> 然后点击作者右边的
> ![[Pasted image 20260315131322.png]]
> 跳转的页面中，地址栏部分就是对应的哈希码
> ![[Pasted image 20260315131222.png]]
> 

当在测试作业中运行测试**生成了Allure测试报告后**（通常是一个包含HTML文件的目录），就可以使用`actions/upload-artifact`**将整个报告目录上传为artifact**。工作流结束后就可以从GitHub Actions页面下载报告，在本地浏览器打开查看。（默认保留90天，过期后自动删除）
![[Pasted image 20260315121514.png]]
#### 2.3.3 使用Cache输出持久化
##### GitHub Actions提供的Cache
Artifacts用于长期存储输出，而Cache则存储中间数据（依赖项、构建工具链等），以加快后续运行速度。GitHub的托管缓存可以让每个仓库最多存储10GB的数据，并在添加新条目时自动清除旧条目，通过`actions/cache`使用缓存：
![[Pasted image 20260315134913.png]]
![[Pasted image 20260315134954.png]]
对于步骤一：`path: demo-cache`定义了要缓存/恢复的目录或文件路径；`key: demo-cache-v1`是缓存的唯一键名，当键匹配时缓存就会恢复；`id: demo-cache`可以在后续步骤通过`step.demo-cache.outputs.cache-hit`获取是否命中了缓存。
在第一次执行时缓存没有命中，耗时8秒：
![[Pasted image 20260315140035.png]]
但是当它存进了缓存后，再次执行作业查询缓存就可以命中了，所以包含了`if`判断的`Populate cache directory`和`Save cache`两个步骤都不执行，此次耗时只有4秒：
![[Pasted image 20260315140429.png]]
GitHub Actions 的缓存机制本质上是一个**键值存储系统**：用一个**键（key）** 来保存一个**路径（path）下的文件或目录**，后续运行中只要提供相同的键，就能恢复这些文件。`actions/cache`实际上**同时包含了”恢复“和”保存“两个功能**，具体行为取决于执行时的上下文和是否有缓存命中。
`key`和`path`不是绑定的，GitHub只关心`key`，不关心`path`（它只是存储一个由`key`索引的blob），所以**理论上可以在不同的运行中甚至同一运行的不同步骤中，用相同的`key`搭配不同的`path`。但是这样做容易出现混乱**，因为缓存的内容可能和恢复时的目标路径不一致，所以**尽量保证同一作业内恢复和保存的`key`和`path`一致**。
在第一次调用`actions/cache`时，Actions会根据提供的`key`去缓存中查找是否有匹配缓存，找到了就会把`cache-hit`设置为`true`并把内容下载恢复到指定的`path`，没找到只设置`cache-hit = false`并退出。但是**因为`actions/cache`不会在”恢复失败“时自动保存，所以需要显式地在生成内容后再次调用它来执行保存操作**。在同一工作再次使用`actions/cache`，且`key`和`path`与之前相同、`key`对应的缓存不存在（或明确要求覆盖），那么它就会**将当前`path`目录的所有内容打包上传**，与这个key关联起来，供后续运行使用。
##### 内置Cache机制
许多安装操作都包含专门设计的缓存机制。例如，可以根据锁定文件`actions/setup-node`自动缓存，以便后续安装可以直接从缓存中获取所需内容：`node_modules`

```yaml
jobs: 
  node-cache:
    name: npm dependency cache (setup-node)
    runs-on: ubuntu-24.04
    defaults:
      run:
        working-directory: 04-advanced-features/caching/minimal-node-project

    steps:
      - uses: actions/checkout@ff7abcd0c3c05ccf6adc123a8cd1fd4fb30fb493 # v5.0.0
      - uses: actions/setup-node@d7a11313b581b306c961b506cfc8971208bb03f6 # v4.4.0
        with:
          node-version: 20
          cache: npm
          # Point cache action at the specific lock-file path
          cache-dependency-path: 04-advanced-features/caching/minimal-node-project/package-lock.json
      - name: Install dependencies
        run: npm ci --prefer-offline --no-audit
      - name: List first few modules
        run: ls -R node_modules | head
```
这个工作流在GitHub Actions运行环境中安装Node.js项目依赖，利用缓存机制，避免每次运行都重新下载所有npm包，从而加速构建，并通过`setup-node`的内置缓存，自动管理`node_modules`的保存和恢复。这个工作流是 GitHub Actions 中**处理 Node.js 项目**依赖缓存的**标准实践**，而对于Python的pip包则需要使用`setup-python`的`cache: pip`。
比起手动使用`actions/cache`，使用`srtup-node`内置缓存将三步（恢复缓存、未命中则安装依赖、保存缓存。以及自己设计`key`）封装成了一个参数，所以**只需要声明`cache: npm`即可**，它就会自动在运行`setup-node`时尝试恢复缓存、在执行`npm ci`时如果缓存已恢复那么就会直接利用、在工作结束时，如果依赖有变化会自动保存。至于底层原理，和`actions/cache`一致，只是它会负责`key`和`path`的生成。因此对于项目依赖（如 npm 包、pip 包、maven 依赖等），**优先使用各语言官方 setup action 的内置缓存**。
但是**手动 `actions/cache` 则适用于非依赖项的通用缓存场景**，如测试报告、代码覆盖率数据、编译产物、模型文件等。虽然比内置缓存多写几行YAML，但是非常灵活，也是CI/CD的常用技巧。
##### 其他缓存方法
第三方运行器提供商有时会提供替代缓存模型。Namespace的缓存卷会对整个目录进行快照，并在后续运行时将其重新挂载，从而避免大量的上传/下载循环，并能显著缩短构建时间，尤其适用于依赖项较多的项目或语言工具链。
#### 2.3.4 控制GitHub权限
GitHub Actions会自动为每次运行生成一个临时的`secrets.GITHUB_TOKEN`，用于在工作流中通过GitHub API进行身份验证。每个工作流中注入的初始权限`GITHUB_TOKEN`都**仅允许读取存储库内容和软件包**。除了`pull-requests`，`permissions` 还可以控制对 `contents`（仓库内容）、`issues`、`packages`、`actions`、`checks`、`statuses` 等的访问：
![[Pasted image 20260315181125.png]]
使用`permissions`字段收紧或扩大这些权限，可以避免意外写入的影响，并有助于强制执行最小权限原则：
```yaml
jobs: 
  read-only-pr: 
    runs-on: ubuntu-24.04 
    permissions: 
      pull-requests: read # Can read PR data only 
    # 让job第二步失败后继续执行后续步骤，而不是立即终止
    continue-on-error: true 
  steps:
    - uses: actions/checkout@v5.0.0 
    - name: List the first 5 open PRs (allowed) 
      run: gh pr list --limit 5 
      env: 
        GH_TOKEN: ${{ secrets.GITHUB_TOKEN }} 
        
    - name: Attempt to add a label (expected to fail) 
      env: 
        GH_TOKEN: ${{ secrets.GITHUB_TOKEN }} 
      run: | gh pr edit 1 --add-label "documentation"
       
    - name: Confirm the failure 
      if: failure()   # 检测前一步是否失败
      run: echo "❌ Write operation was blocked as expected – token is read-only."
    
  read-write-pr: 
    runs-on: ubuntu-24.04 
    permissions: 
      pull-requests: write 
    steps: 
      ...
```
`permissions`字段可以定义在工作流级别（全局生效）或作业级别（仅对该job生效），上面的代码前一个工作限定`GITHUB_TOKEN`只能读取PR信息，而任何写入操作（如添加标签、修改标题等）都会被GitHub API拒绝，返回403错误；后一个工作的`GITHUB_TOKEN`只有对PR的读写权力，因此可以成功执行`gh pr edit --add-label`等写入操作。
在工作的步骤中，通过环境变量`GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}`将令牌传递给`gh`命令行工具，`gh`使用该令牌调用GitHub API，API服务器会根据令牌的权限范围决定是否允许操作
#### 2.3.5 矩阵策略、条件语句和并发控制
![[Pasted image 20260315185326.png]]
 **矩阵作业**：可以将单个作业定义扩展到多个输入，适用于在不同OS、语言版本或依赖项集合上进行测试。在`strategy.matrix`下面定义变量和取值列表，每个组合会生成一个独立的任务：
 ```yaml
jobs: 
  matrix-job: 
    runs-on: ubuntu-24.04 
    strategy: 
      fail-fast: true 
      matrix: 
        number: [1, 2] 
        letter: [a, b, c] 
        exclude: 
          - number: 1 
            letter: c 
    timeout-minutes: 5
 ```
 这会生成 `2 × 3 = 6` 个组合： `(1,a)`, `(1,b)`, `(1,c)`, `(2,a)`, `(2,b)`, `(2,c)`，`exclude`可以移除特定组合，如示例中排除了`(1,c)`，所以实际只有5个任务运行。在后续步骤中通过`${{ matrix.number }}`和`${{ matrix.number }}`获取当前组合的值：
 ```yaml
    steps: 
      # Step 1 – always runs 
      - name: Echo number and letter 
        run: | 
          echo "Number: ${{ matrix.number }}" 
          echo "Letter: ${{ matrix.letter }}" 
          
      # Step 2 – run for every combo EXCEPT number=2 ∧ letter=c 
      - name: Run everywhere except 2c 
        if: ${{ ! (matrix.number == 2 && matrix.letter == 'c') }} 
        run: echo "✔ This step runs for ${{ matrix.number }}${{ matrix.letter }}"
 ```
 **条件语句**：`if`表达式基于上下文（如矩阵变量、上一步输出、GitHub事件信息等）控制作业/步骤是否执行。上面示例中表示对于矩阵组合 `(2,c)` 不执行该步骤，其他组合都执行。
 **并发组**：控制工作流或作业的并发运行，避免重复或无意义的构建浪费资源。常用于同一分支多次推送时，只保留最新的一次运行，取消正在运行的旧运行和防止同一环境下多个部署任务冲突的场景，如：
 ```yaml
concurrency: 
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
 ```
 - `group`：定义并发组的标识。这里使用工作流名 + 当前 ref（分支或标签）作为组名，这样同一分支上的运行属于同一组。
- `cancel-in-progress: true`：当组内已有运行在进行时，是否取消它们。设为 `true` 会终止旧运行，启动新运行。
这段代码的效果就是：如果快速连续触发两次工作流，第一次运行会被第二次自动取消，因为它们共享同一个组（同一分支）

### 2.4 Actions市场
[GitHub Actions Marketplace](https://github.com/marketplace?type=actions)包含数千个可重用的构建模块，里面每个条目代表一个发布Actions的独立仓库，点击进入对应的操作详情界面，在该界面右边点击`View source code`进入远程仓库项目页面：
![[Pasted image 20260315194019.png]]
在选择完对应的Actions后，**尽量使用哈希码来锁定对应的Actions版本**：
```yaml
dangerous: 
  steps: 
    - uses: actions/checkout 
    - uses: actions/checkout@v5 
    - uses: actions/checkout@v5.0.0 

secure: 
  steps: 
    - uses: actions/checkout@ff7abcd0c3c05ccf6adc123a8cd1fd4fb30fb493 # v5.0.0
```
![[Pasted image 20260315192757.png]]
#### 经典Actions
**官方 GitHub Actions**：
- [`actions/checkout`](https://github.com/marketplace/actions/checkout-action)：将仓库克隆到运行器中。
- [`actions/cache`](https://github.com/marketplace/actions/cache)：在运行之间保留依赖项或构建输出。
- [`actions/upload-artifact`](https://github.com/marketplace/actions/upload-a-build-artifact)以及[`actions/download-artifact`](https://github.com/marketplace/actions/download-a-build-artifact)：在作业之间移动Artifacts。
- [`actions/github-script`](https://github.com/marketplace/actions/github-script)：运行经过 GitHub API 身份验证的简短 JavaScript 代码片段。
**运行时和依赖项安装程序**：
- [`actions/setup-node`](https://github.com/marketplace/actions/setup-node-js-environment)
- [`actions/setup-go`](https://github.com/marketplace/actions/setup-go-environment)
- [`actions/setup-java`](https://github.com/marketplace/actions/setup-java-jdk)
- [`actions/setup-python`](https://github.com/marketplace/actions/setup-python)
**身份验证助手**：
- [`aws-actions/configure-aws-credentials`](https://github.com/marketplace/actions/configure-aws-credentials-action-for-github-actions)
- [`azure/login`](https://github.com/marketplace/actions/azure-login)
- [`google-github-actions/auth`](https://github.com/marketplace/actions/authenticate-to-google-cloud)
它们简化了获取云提供商临时凭证的过程，无需在工作流程中硬编码密钥。
**其他公用设施**：
- [`github/super-linter`](https://github.com/marketplace/actions/super-linter)– 一步完成与语言无关的代码检查。
- [`docker/build-push-action`](https://github.com/marketplace/actions/build-and-push-docker-images)– 构建并发布多架构容器镜像。
### 2.5 编写Actions【略】
从零开始构建自定义JavaScript和Docker Action
#### 2.5.1 复合Action
#### 2.5.2 可复用工作流
#### 2.5.3 JavaScript/TypeScript Action
#### 2.5.4 容器Action

### 2.6 常用工作流【重要】
![[Pasted image 20260315200856.png]]
#### 2.6.1 Validate（验证）
- **Lint**：代码风格检查。例如用 ESLint（JavaScript）、Pylint（Python）确保代码格式一致，避免低级错误。
  最简化的Lint流程只需要：① 查看代码仓库；② 安装代码检查工具，或使用内置代码检查工具的容器；③对代码库运行代码检查工具。
  但是可以有这些改进：确定哪些文件发生了更改，以便仅对代码库的相关子集进行代码检查；缓存工具下载或依赖项安装，以免每次运行都要重新安装；在GitHub用户界面中通过任务摘要、拉取请求评论或状态检查来实现结果。
  Marketplace的Actions通常可以避免手动编写对应逻辑的麻烦，[`Super-Linter`](https://github.com/marketplace/actions/super-linter)集成了数十种代码检查工具，并自动将工作范围限定在工作流运行中更改的文件。
![[Pasted image 20260315201157.png]]
- **Test**：运行自动化测试（单元测试、集成测试）。比如用 pytest 测试 Python 代码，Jest 测试 JavaScript。
  测试的工作流程与代码检查类似，但引入了依赖管理和组件管理：① 检查代码库；② 安装语言运行时和包管理器；③ 恢复依赖关系；④ 运行测试组件
  ![[Pasted image 20260315202522.png]]
  改进之处在于缓存依赖项、上传测试报告以供后续检查，以及发布覆盖率指标，Marketplace都提供了对应的Actions
- **Static Analysis**：静态代码分析，不运行代码而是分析源码发现潜在缺陷、安全漏洞。如 SonarQube、CodeQL。
- **目的**：在合并代码前保证质量，减少 bug 引入。
#### 2.6.2 Build（构建）
- **Executable**：编译生成可执行文件，例如 Java 的 JAR，Go 的二进制文件。
- **Container Image**：构建 Docker 镜像，用于容器化部署。镜像生成后，将其推送到选中的镜像仓库，并可选择使用 [cosign](https://github.com/sigstore/cosign)或类似工具对其进行签名。
- **目的**：将源码转换为可部署的 artifact。
#### 2.6.3 Deploy（部署）
- **Push Based Deploys**：**基于推送的部署**。CI/CD 流水线直接连接目标服务器或云平台，在Actions中写部署脚本，执行部署命令（如 SSH 到服务器、运行 kubectl 或调用云 API）。这种方式简单直接。【即GitHub Actions有能力实现这个功能】
![[Pasted image 20260315204418.png]]
- **Update Tags for GitOps Deploys**：**GitOps 风格的部署**。CI/CD 只更新 Git 仓库中的配置文件（例如 Kubernetes 清单中的镜像 tag），然后由集群内的控制器（如 ArgoCD、Flux）检测变化并自动同步到实际环境。**这种方式更安全、可追溯，此时Actions只做了Validate和Build+更新配置，真正的部署动作由K8s完成**。但其实GitHub Actions有能力覆盖这个CI/CD流水线
![[Pasted image 20260315204443.png]]
#### 2.6.4 Repo Automations（仓库自动化）
- **Release Automations**：自动生成 Release 版本。例如用 `release-please` 根据 Conventional Commits 自动更新版本号、生成 changelog、创建 GitHub Release【如果使用定时触发或事件触发执行仓库管理任务，那么GitHub Actions同样也有能力实现该功能】
- **Stale Issues/PRs**：自动标记和关闭长时间未活动的 Issue 或 Pull Request，保持仓库整洁。
- **Dependency Upgrades**：自动更新依赖项。比如 Dependabot 定期检查依赖版本，提交 PR 更新，然后 CI 运行测试验证，合并后即完成升级。
这些工作流类型覆盖了软件交付生命周期的核心环节：从代码提交（验证、构建）到部署（部署），再到仓库维护（自动化）。它展示了如何将一个任务拆解成步骤，并将其映射到GitHub Actions的工作流中，同时学习如何迭代和优化这些工作流。至于Actions，可以从零手动编写，也可以将Marketplace的Actions作为起点。
### 2.7 开发者体验
优化团队的日志、密钥、环境和权限
### 2.8 最佳实践
利用安全性、可靠性和成本效益技术来强化工作流程
### 2.9 实战项目【重要】
#### 2.9.1 项目简介
该项目是一个最小化的三层Web应用，采用单仓库结构，包含以下组件：
* **前端**：React应用，负责页面展示
* **后端API**：提供两个不同语言实现的API，前端会调用它们
  - 一个使用Node.js编写
  - 一个使用Go编写
* **负载生成器**：用Python编写，会持续向其中一个或两个API发送请求，模拟真实用户流量
* **数据库**：PostgreSQL，用于记录每次页面访问的数据
* **部署配置**：包含Kubernetes清单文件，用于将应用部署到Kubernetes集群中
* **自动化脚本**：GitHub Actions工作流文件将存放在`.github/workflows`
所有应用代码按语言分类存放在`services/`子目录下：`services/go`、`services/node/`、`services/python/`
#### 2.9.2 六个工作流
##### 1) 测试工作流
- 为每个应用（前端、Node API、Go API、Python负载生成器等）运行单元测试。
- 目的：确保代码变更不会破坏已有功能，维持代码质量。 
##### 2)  **发布流程自动化**
- 基于对仓库的特定提交（例如打标签或推送到主分支）自动触发发布流程。     
- 可能包括生成版本号、创建Release、打包等操作。
##### 3)  **构建并推送容器镜像**    
- 当代码更新时，自动构建应用的Docker镜像，并将其推送到容器镜像仓库（如Docker Hub、GitHub Container Registry等）。
- 为后续部署准备最新的镜像。
##### 4) **识别并关闭陈旧的Issue和PR**
 - 定期扫描仓库中的Issue和Pull Request，自动关闭那些长时间无活动、已过时的条目。
 - 保持项目看板的整洁，减少维护负担。        
##### 5) **导出GitHub Actions计时数据到Honeycomb**
 - 收集GitHub Actions工作流的执行时间、延迟等指标，发送到Honeycomb（一个可观测性平台）。
 - 目的：监控CI/CD管道的性能，帮助发现瓶颈和优化点。        
##### 6) **自动更新Kubernetes清单**
 - 当有新镜像构建并推送到仓库后，自动修改Kubernetes部署文件（如Deployment中的镜像标签），使清单与最新镜像保持一致。
 - 结合GitOps实践，可以通过提交更新后的清单到仓库，触发集群的自动同步（例如使用ArgoCD），从而将应用部署到不同环境的Kubernetes集群中。
![[Pasted image 20260316181635.png]]
#### 2.9.3 实操
* 使用Devbox同步环境：在bash终端输入`devbox shell`，获得作用于本次窗口的环境`.devbox`和`.venv`。其背后的逻辑是：**Devbox构建了OS级的工具链**（见[[工具：Devbox]]），如`python312@latest`、`go-task@latest`等等，这些用于还原项目的元数据被保存在`.devbox`中（真正的工具存放在`nix/store`下），而对于Python这样的工具，**如果不使用虚拟环境将依赖管理妥当的话，依赖的影响则会扩散到项目外部**。所以**Devbox在提供Python解释器时，还自动地为每个项目创建了隔离的Python虚拟环境**，在`.devbox/virtenv/python312/`下，有名为`venvSheelHook.sh`的脚本文件，其中就在检查`venv`：
```bash
# Check that Python version supports venv
if ! python -c 'import venv' 1> /dev/null 2> /dev/null; then
    echo "WARNING: Python version must be > 3.3 to create a virtual environment."
    touch "$STATE_FILE"
    exit 1
fi

# Check if the directory exists
if [ -d "$VENV_DIR" ]; then
    if is_valid_venv "$VENV_DIR"; then
        # Check if we've already run this script
        if [ -f "$STATE_FILE" ]; then
            # "We've already run this script. Exiting..."
            exit 0
        fi
        if ! is_devbox_venv "$VENV_DIR"; then
            echo "WARNING: Virtual environment at $VENV_DIR doesn't use Devbox Python."
            echo "Do you want to overwrite it? (y/n)"
            read reply
            echo
            if [[ $reply =~ ^[Yy]$ ]]; then
                echo "Overwriting existing virtual environment..."
                create_venv
            elif [[ $reply =~ ^[Nn]$ ]]; then
                echo "Using your existing virtual environment. We recommend changing \$VENV_DIR to a different location"
                touch "$STATE_FILE"
                exit 0
            else
                echo "Invalid input. Exiting..."
                exit 1
            fi
        fi
    else
        echo "Directory exists but is not a valid virtual environment. Creating a new one..."
        create_venv
    fi
else
    echo "Virtual environment directory doesn't exist. Creating new one..."
    create_venv
fi
```
这段脚本先确认Devbox提供的Python是否含有标准库`venv`，然后检查`.env`（即`$VENV_DIR`）如果目录不存在或无效，则调用`create_venv()`创建虚拟环境，如果目录存在且有效就查看是否使用Devbox提供的Python（`is_devbox_venv`），如果使用其他的Python则会提示选择是否覆盖。
总之，Devbox会在`devbox shell`运行之后为项目配备一个**最基础的空白虚拟环境**，**而至于Python项目所需要的大量依赖，则由Python微服务中的`poetry.lock`和`pyproject.toml`来构建**。比如`pyproject.toml`中就指定了`pytest`版本：
```toml
[tool.poetry.group.dev.dependencies]
pytest = "^8.4.1"
```
而运行`devbox shell`后获得的`.venv`里面并不包含，这证明了这两个文件并未在此使用，此后需要执行`poetry install`或`task install`才能在`.venv`中包含这些依赖包

* 进入数据库迁移脚本所在目录，并查看当前任务列表：
```bash
cd services/other/api-golang-migrator/
task --list
```
然后就会输出这样的列表（Task笔记见[[Task]]）：
```text
task: Available tasks for this project:
* build-container-image:      Build container image
* run-postgres:               Start postgres container
* run-psql-init-script:       Execute psql commands
```
* 执行`run-postgres`命令，启动psql数据库。此时可能会出现Docker找不到的问题，这是因为Nix在构建隔离的开发环境时，**有时会把`/bin`或`/usr/bin`省略掉**。而执行`which docker`可以得到Docker位置在WSL的`usr/bin/docker`处（**此处并非WSL自带Docker**，而是一个自动安装的Docker客户端，**因为Windows下的Docker可能存在路径转换或权限问题，所以Docker在WSL中创建一个软链接脚本，实际指向Windows的`docker.exe`**），所以会显示找不到Docker，此时手动添加`export PATH="$PATH:/usr/bin"`（或在`devbox.json`的`init_hook`中添加，避免每次都手动添加）
![[Pasted image 20260318060110.png]]
![[Pasted image 20260318060129.png]]
* 在第二个终端窗口执行psql数据库初始化任务：`task run-psql-init-script`：
![[Pasted image 20260318060704.png]]
* 在第二个终端继续执行Node微服务的启动：① 进入`services/node/api-node/`目录；② 执行`task install`；③ 执行`task run`：
![[Pasted image 20260318061421.png]]
然后Node服务就会在本机的3000端口提供：
![[Pasted image 20260318125945.png]]
* 在第三个终端开启Go微服务：① 进入`services/go/api-golang`；② 运行`task install`；③ 运行`task run`
![[Pasted image 20260318130504.png]]
然后Go的微服务就会在本机的8000端口提供：
![[Pasted image 20260318130606.png]]
* 再启动第四个终端，执行Python负载模拟器微服务：① 进入`services/python/load-generator`；② 执行`task install`；③ 执行`task run`；④ 微服务会不断向本机8000端口发送请求（Go微服务）
* 最后启动第五个终端，执行React前端微服务：① 进入`services/react/client-react`；② 执行`task install`；③ 执行`task run`；
![[Pasted image 20260318131811.png]]
React前端微服务会在本机5173端口提供服务：
![[Pasted image 20260318131744.png]]
* 
* 
* *