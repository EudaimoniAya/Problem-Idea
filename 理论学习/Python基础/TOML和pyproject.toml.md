# 一、TOML语法
比起JSON和YAML，TOML更加简洁且更适合Python项目，因为语法相似且类型对应（比如注释用`#`、支持单引号`'`和双引号`"`表示字符串）。
Python 3.11引入了标准模块`tomllib`能够读取TOML文件：
```toml
# This is a toml file, named "config.toml"
name = 'Aya'
age = 24
gender = "\"Male\""
hobbies = ['reading', 'coding']
birthdate = 2026-02-28T00:00:00Z

shape = {height = 1, weight = 2}
```
Python读取：
```Python
# read_toml.py
import tomllib
from pprint import pp 

with open('config.toml', 'br') as f:  # 仅二进制读取
	config: dict = tomllib.load(f)
pp(config)
```
除此之外，为了让经常出现的嵌套表结构美观，TOML使用了特殊的表示方法：① 用方括号`[]`括起table的名称
```toml
[account]
QQ = "A"
google = "B"
X = "C"
# 如果要实现嵌套表，那就用.来引出变量
bilibili.username = "AAA"
bilibili.password = "123456"

# 如果嵌套表的字段多了（比如`bilibili`还需要添加`url`、`star`等信息），就会显得很杂乱，此时可以将上层嵌套写进方括号内
[acount.youtube]
username = "DDDD"
password = "123456"
star = ["X", "Y", "Z"]
url = "xxxxxxxxx"

# 如果需要给每个平台都添加类似的字段，可以使用JSON风格的
# account = [
#     {name = "wechat", username = "ASD", password = "123"},
#     {name = "weibo", username = "ASD", password = "123"}
# ]
# 也可以使用TOML特有的table组：
[[account]]
name = "wechat"
username = "ASD"
password = "123"

[[account]]
name = "weibo"
username = "ASD"
password = "123"
```
# 二、Python的工程化与工具链：项目配置与打包分发
工程化是一套**让软件项目可重复、可协作、可交付、可维护的实践方法**；工具链则是实现这些实践的**具体工具**。其中包括：
* **项目配置与依赖管理（本节内容）**：定义项目元数据、依赖、环境要求，统一入口
* **构建与打包（本节内容）**：将源码转换为可分发的格式（sdist、wheel），便于安装和发布。
* **测试**：保证代码质量，使用`pytest`等测试框架编写单元/集成测试
* **代码质量（Lint）**：统一代码风格、发现潜在错误，比如使用`ruff`做Lint和格式化
* **类型检查**：提前发现类型错误，增强代码可读性，比如使用`mypy`、`pyright`进行检查
* **版本控制与协作**：管理代码历史，支持多人协同。比如使用Git分支模型，使用PR、tag管理版本演进等
* **持续集成/持续部署**：自动化构建、测试、部署，保证每次提交质量
* **环境管理**：隔离项目依赖，避免冲突。比如使用`conda`或`.venv`创建虚拟环境，通过`pyproject.toml`的依赖列表保证环境可复现
* **文档**：记录项目用途、安装、使用说明。
* **容器化与部署**：保证运行环境一致，简化部署
## 2.1 项目配置与依赖管理
一个项目不应该只是一堆代码，它还需要告诉其他人或工具：
* 项目的名字、版本、作者是谁（**元数据**）
* 项目需要哪些第三方库运行（**依赖**）
* 最低的Python版本是多少
* 提供哪些命令行入口
### 2.1.1 传统配置文件
过去这些信息分散在多个文件中：
* `requirements.txt`：列出项目依赖的第三方包及其版本，用于安装。但是它**仅记录顶层依赖，不记录间接依赖**，导致环境不可重现
```text
# 格式为`package==version`的列表
PyMySQL==1.1.2
pyperclip==1.11.0
pytest==9.0.2
pytest-asyncio==1.3.0
python-dateutil==2.9.0.post0
python-dotenv==1.2.1
```
* `setup.py`：用于定义项目元数据和依赖，用于**打包发布**（生成`.whl`或`.tar.gz`），以及`pip insatall -e .`可编辑安装。但是**因为配置是可执行脚本，可能引入安全问题**（如果其中包含恶意代码，如`os.system`、文件操作、网络请求等，代码就会在用户执行`pip install`或`python setup.py install`时无警告地执行），并且维护两个文件（`setup.py`+`requirements.txt`）会造成依赖重复
```python
# 内容为Python脚本，其中`setuptools.setup()`将在用户直接运行
# `python setup.py install`或`pip install`时执行
from setuptools import setup, find_packages
setup(
    name="myproject",
    version="0.1.0",
    packages=find_packages(),
    install_requires=[
        "fastapi>=0.115.0",
        "sqlalchemy>=2.0.23",
    ],
)
```
* `setup.cfg`：以声明式方式替代`setup.py`，将元数据与逻辑分离。在传统`setuptools`体系下，`setup.cfg`只能写键值对（安全），而`setup.py`支持动态逻辑，如根据环境变量动态生成`version`、动态读取`README.md`并进行复杂处理、安装`setup_requires` 前置依赖等场景下，仍然需要能够执行代码的`setup.py`来完成。
  而如果没有上述动态逻辑，那么最佳实践就是让`setup.py`变成一个”引导文件“，把配置全部放在`setup.cfg`中
```ini
# setup.cfg 为 .ini 格式文件
[metadata]
name = myproject
version = 0.1.0
[options]
packages = find:
install_requires =
    fastapi>=0.115.0
    sqlalchemy>=2.0.23
```
### 2.1.2 现代统一配置：pyproject.toml
`pyproject.toml` 是一个**配置文件**（无脚本执行风险），用于打包工具以及其他工具，如 LINTER、类型检查器等，它统一管理：
- **项目元数据**（名称、版本、作者、描述等）
- **依赖声明**（`dependencies`、`optional-dependencies`）
- **构建系统**（`[build-system]`）
- **工具配置**（`[tool.isort]`、`[tool.mypy]`等）
该文件中有三种可能的 TOML 表：
* `[build-system]`：声明使用哪个构建系统，以及构建项目需要的其他依赖，该表应该始终存在。它包含一个名为`build-backend`的键，用来指定要使用的构建系统，另外包含一个名为`requires`的键，它是构建项目所需的依赖列表
  其中build在Python生态中指“构建”过程（生成源码包、wheel包等），system指一套工具链（如`setuptools`、`flit`、`poetry`等）
  ![[Pasted image 20260329003510.png]]
* `[project]`：用于指定项目基本元数据的格式，比如项目名、依赖、作者名等，如果只是某个特定功能需要一些包，就用`[project.optional-dependencies]`将其设置为可选，它将一组依赖打包成一个组，比如有`dev`、`test`、`mysql`组，每组中有不同的依赖，在安装时通过`pip install -e .[组名1,组名2,···]`，如果不带方括号，**可选依赖就不会被安装**，只有`dependencies`的必装依赖安装
  ![[Pasted image 20260329003556.png]]
* `[tool]`：它包含了所有和Python项目相关的工具，可以让用户指定配置数据，只要在`[tool]`内使用特定工具的子表，例如`[tool.hatch]`、`[tool.mypy]`、`[tool.black]`
  ![[Pasted image 20260329030436.png]]
## 2.2 构建与打包
构建是指将源代码（`.py` 文件、资源文件、可能的 C 扩展）转换成**可分发的格式**的过程。Python 有两种主流分发格式：
- **sdist（源码分发包）**：一个 `.tar.gz` 压缩包，包含原始源码、`pyproject.toml`、`README` 等。安装时需要解压并在目标机器上执行构建步骤（若有 C 扩展则需编译）。它就是GitHub上对应版本的release下的assets
  ![[Pasted image 20260329034234.png]]
- **wheel（预构建分发包）**：一个 `.whl` 文件，本质是 zip 压缩包，里面包含了已经整理好的文件结构（**纯 Python 文件或编译好的 C 扩展，如`.so`/`.pyd`**）。安装时 pip 只需解压复制，**无需编译**，所以安装速度极快。但是Python程序运行时**仍然需要解释器来执行`.py`文件**，所以wheel不是最终的可执行文件，不同于Go的二进制镜像是可直接运行的机器码
上面的打包和分发工作需要像`setuptools`这样的构建工具来完成，解释器（CPython）的工作是运行`.py`文件，不负责打包、分发、处理依赖。像`setuptools`、`flit`、`poetry`等工具，它们做的是读取`pyproject.toml`中的元数据和依赖，将项目代码打包成sdist或wheel，并处理依赖关系，生成安装时需要的元数据。
同样是构建（build），它可以**和Docker镜像构建类比**：
* Python打包输入的是源代码和`pyproject.toml`，通过`pip`/`build`生成sdist（`.tar.gz`）或wheel（`.whl`），然后上传到PyPI，其他人通过`pip install`直接下载安装，安装后成为Python环境的一部分
* Docker镜像构建输入的是源代码和Dockerfile，通过`docker build`构建出镜像，然后推送到DockerHub或其他Registry，其他人通过`docker pull`下载，安装后用来创建容器。
* 两种构建操作背后都基于相同的工程理念，只是抽象层次不同：**Python打包是代码（库）级的，目的是代码的复用和分发，在开发期完成；而Docker镜像是系统（应用）级，目的是应用的交付与运行，在发布期完成**
在`setuptools`构建或安装Python时会自动生成一个名为`.egg-info`的目录，它是Python包的元数据目录，**用于开发时识别包，而不参与分发，随时可删（临时）**（而`.egg`文件则另外包含所有代码的压缩包，用于分发/安装完整包，**这种方法已经被淘汰**，被`wheel`取代）。当运行下面代码就会产生（但是现代更推荐使用新方法构建，它们生成的是`dist-info`）：
```bash
# 1. 传统方法（setup.py）
python setup.py egg_info      # 专门生成 .egg-info
python setup.py develop     # 可编辑安装（生成 .egg-info）
python setup.py install     # 安装时生成

# 2. 现代工具（pip/poetry/pdm）
pip install -e .            # 可编辑安装（旧行为）
```
它包含六个文件
* `PKG-INFO`：包含包的基本信息（名称、版本、描述、作者等），还包括依赖项和`README.md`的内容
  ![[Pasted image 20260330093835.png]]
* `SOURCES.txt`：包含项目源代码文件列表
* `dependency_links.txt`：依赖链接，没有则为空文件
* `requires.txt`：项目运行时依赖
* `top_level.txt`：顶层模块名
  ![[Pasted image 20260330095201.png]]
* `entry_points.txt`：命令行入口点，如果没有则不会生成
  ![[Pasted image 20260330093617.png]]

> [!问题] 为什么需要`.egg-info`/`.dist-info`？
> 因为Python的`sys.path`默认不包含当前项目目录，它不知道该目录是一个可导入包。而这两个文件会提供包发现功能，它能够告诉Python存在一个`<模块名>`的模块，入口在`./<模块名>/`

## 2.3 可编辑安装与虚拟环境
### 2.3.1 虚拟环境
虚拟环境能够保证项目运行的环境可隔离，使用venv、conda、devbox等工具创建虚拟环境后，需要pip、conda、nix等包管理器，将下载的包管理好。不同的工具有不同的管理情况：
* venv：会在每个虚拟环境中创建独立的`site-packages/`（在`lib/`目录下），**不同项目之间不共享包，完全隔离**。
  ![[Pasted image 20260330113745.png]]
* conda：所有的包会下载到Conda的中央缓存`~/miniconda3/pkgs/`下，这些是下载的原始包，**跨环境共享**。虚拟环境在激活后仍然能够安装或卸载包来修改它，**并且一个环境内，同一个包只能有一个版本**
  ![[Pasted image 20260330105300.png]]
  然后`miniconda/envs/`下各个环境中的`lib/site-packages`中存放包的硬链接或复制（默认为硬链接），最终以成对的方式管理这些包（**venv也一样，只是venv无法共享，每个环境存放的是完整源代码**）：
  * `<包名>/`：存放指向真实位置（`pkgs/<包名>`）的硬链接
  * `<包名>-<版本>.dist-info/`：类似`.egg-info`，存放包的元数据，必须通过这种方式将在包`site-packages/`中注册，Python才能发现，pip才能够管理它。
  ![[Pasted image 20260330110031.png]]
* Nix：类似于Conda的管理思路，但是它要更为严格，使用哈希确保内容锁定，且`/nix/store`全局只读存储，**一旦激活环境就无法修改它。并且Nix支持同一个包的不同版本共存，因为包路径包含依赖哈希，不同版本使用会去完全不同的目录找**。
### 2.3.2 可编辑安装
在远程获取了代码之后，一般就需要构建虚拟环境，根据项目声明的依赖项还原运行环境。并且还**需要让项目本身能够被Python解释器找到**
#### 传统方法：流程
在传统方法中，一般使用`pip install .`**构建并安装当前项目+依赖**：
* **读取构建配置**：pip会读取`pyproject.toml`，解析`[build-system]`执行构建系统
* **创建隔离构建环境&构建项目**：创建**临时的隔离构建环境**`/tmp/pip-build-env-xxx/`（构建过程可能需要特定版本的工具，不应该让它污染开发环境），构建完当前项目后（构建过程见2.2，**源代码和元数据会被复制进wheel**），将构建产物（本项目的`.whl`）安装到目标环境后就会**自动删除**
> [!问题] 构建是用于打包和分发的，为什么还需要在开发时使用构建工具把正在开发的项目给打包？
> 其实开发时不需要真正地打包，但是`pip install .`的设计迫使构建发生，这是Python包管理的一个历史遗留问题和设计妥协。
> **pip强制要求一切情况都要复用生产安装的流程，即使开发时不需要**。比如开发时不需要`wheel`包文件，不需要把源码复制到`site-packages/`因为需要未来开发面临着大量更改。**开发过程中，构建只需要生成元数据让pip进行包管理**

* **安装wheel**：然后wheel就会被解压到虚拟环境中，得到从wheel解压的源码副本（就是`site-packages/`下的`<项目名>`目录，**但是在可编辑安装时不会生成它**），和项目的**元数据**（`site-packages/`下的`<项目名>-<版本>.dist-info`目录）。**此步骤中依赖也会被安装，并被成对地管理起来**
  ![[Pasted image 20260330103806.png]]
* **记录安装状态**：pip会将安装的依赖记录，如果执行`pip list`可以确认pip已安装的包（即`site-packages/`下有对应`.dist-info`的包）
#### 传统方法：缺陷
传统方法最终会让`site-packages/`下有一份项目的代码文件A，而当前在编写的则是另外一份文件B。所以修改的是B，而导入的是A：
```bash
# 安装后
python -c "import my_package; print(my_package.__file__)"
# /home/user/.venv/lib/python3.11/site-packages/my_package/__init__.py
# ↑ 不是项目目录！

# 修改项目目录的 src/my_package/core.py
echo "# new feature" >> src/my_package/core.py

# 再次运行
python -c "import my_package"
# 没有变化。因为 site-packages/ 里还是旧代码
```
因此每次修改完代码后，都需要执行`pip install .`重新执行完整的构建流程。
#### 可编辑安装
可编辑安装让源代码的位置始终处于正在编辑的项目的目录，因此它是动态的无需重新安装，但是除了`.dist-info/`文件记录元数据，还需要`.pth`文件执行源代码的位置（因此缺少`<项目名>/`目录）。
![[Pasted image 20260330124844.png]]
但是在现代可编辑安装的严格模式（PEP 660）中，不再通过文本文件`.pth`把项目路径加入`sys.path`（Python的模块搜索路径列表），而是**使用`finder.py`模式**，自定义`MetaPathFinder`，精确控制导入：
![[Pasted image 20260330125809.png]]
这个文件（Python的自定义导入钩子）会在Python启动时注册自己，拦截导入需求，并将其直接指向项目目录。比如第一行有写项目中存在一个顶层模块`app/`，所以之后需要`import app`时，`finder.py`就会查映射表`MAPPING`，找到`app`模块的位置在`E:\Code\backendProject\app`，然后检查该模块是否为包（有没有`__init__.py`），如果是包就完成加载。

> [!总结] 【重要】关于依赖管理、配置、构建的总结
> 开发者之所以可以在开发项目时git clone远程仓库的代码，并创建完虚拟环境后，使用`pip install -e .`完成项目的构建。是因为Python会读取项目中的配置文件（setup.py、setup.cfg）或（pyproject.toml），其中如果定义了动态版本、入口脚本等可执行内容就仍然需要setup.py。在执行完该命令，依赖安装成功之后，就可以进行开发。 
> 而对于安装时需要构建这个过程，它是pip开发过程中基于”希望复用生产构建流程“的目标，但对于开发需求存在冗余的设计。因为生产构建，无需更改代码，把代码放进`site-packages`，构建好wheel，安装好依赖，就能让它跑起来。而不像开发，未来需要面临大量更改。 
> 而在这个背景下，为了解决Python项目根目录不在`sys.path`中的问题，传统方法选择直接将代码和第三方库用一种方法管理，项目/包同名目录存放代码（venv，conda就是存硬链接），.dist-info或.egg-info目录则是存储项目/包的元数据，以注册进site-packages，保证能够被Python识别。 
> 而为了解决传统方法两份代码的问题，新版的可编辑安装方法将项目同名目录改成了.pth文件，存放项目根目录。以及更专业的`.pth`+`finder.py`精确导入的方法。
