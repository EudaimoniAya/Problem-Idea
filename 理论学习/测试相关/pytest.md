# 一、快速上手

## 1.1 使用
pytest有三种启动方式：命令、代码（`pytest.main()`方法启动测试框架）、鼠标
以命令为例，一个简易用例文件`test_base.py`：
```python
def test_ok():  
    open("main.py")
    """
    pytest在简单的基础上，对断言进行了高级封装（AST）
    对python数据结构断言非常友好，并鼓励开发者积极使用断言
    """
    assert 1 == 1  
  
  
def test_fail():  
    assert False
```
在终端输入`pytest`后：
```cmd
(pytest_env) PS E:\Code\pytest_project> pytest
============================================== test session starts ==============================================
platform win32 -- Python 3.11.14, pytest-9.0.2, pluggy-1.6.0
rootdir: E:\Code\pytest_project
collected 2 items                                                                                                 

test_base.py FF                                                                                            [100%]

=================================================== FAILURES ==================================================== 
____________________________________________________ test_ok ____________________________________________________ 

    def test_ok():
>       open("main.py")
E       FileNotFoundError: [Errno 2] No such file or directory: 'main.py'

test_base.py:2: FileNotFoundError
___________________________________________________ test_fail ___________________________________________________ 

    def test_fail():
>       assert False
E       assert False

test_base.py:6: AssertionError
============================================ short test summary info ============================================ 
FAILED test_base.py::test_ok - FileNotFoundError: [Errno 2] No such file or directory: 'main.py'
FAILED test_base.py::test_fail - assert False
=============================================== 2 failed in 0.15s =============================================== 
(
```
## 1.2 看懂结果
```bash
============================= test session starts =============================
platform win32 -- Python 3.11.14, pytest-9.0.2, pluggy-1.6.0
rootdir: E:\Code\pytest_project
collected 6 items

test_base.py .F                                                          [ 33%]
test_data.py FFFF                                                        [100%]

================================== FAILURES ===================================
__________________________________ test_fail __________________________________

    def test_fail():
>       assert False
E       assert False

test_base.py:7: AssertionError
__________________________________ test_int ___________________________________

    def test_int():
        a = 1
        b = 2
>       assert a == b
E       assert 1 == 2

test_data.py:4: AssertionError
__________________________________ test_str ___________________________________

    def test_str():
        a = "aaaaaa"
        b = "aaaabc"
    
>       assert a == b
E       AssertionError: assert 'aaaaaa' == 'aaaabc'
E         
E         - aaaabc
E         + aaaaaa

test_data.py:11: AssertionError
__________________________________ test_list __________________________________

    def test_list():
        a = list('aaaaaa')
        b = list('aaaabc')
    
>       assert a == b
E       AssertionError: assert ['a', 'a', 'a', 'a', 'a', 'a'] == ['a', 'a', 'a', 'a', 'b', 'c']
E         
E         At index 4 diff: 'a' != 'b'
E         Use -v to get more diff

test_data.py:18: AssertionError
__________________________________ test_dict __________________________________

    def test_dict():
        a = {"a": 1, "b": 2, "c": 3}
        b = {"a": 2, "b": 2, "c": 3}
    
>       assert a == b
E       AssertionError: assert {'a': 1, 'b': 2, 'c': 3} == {'a': 2, 'b': 2, 'c': 3}
E         
E         Omitting 2 identical items, use -vv to show
E         Differing items:
E         {'a': 1} != {'a': 2}
E         Use -v to get more diff

test_data.py:25: AssertionError
=========================== short test summary info ===========================
FAILED test_base.py::test_fail - assert False
FAILED test_data.py::test_int - assert 1 == 2
FAILED test_data.py::test_str - AssertionError: assert 'aaaaaa' == 'aaaabc'
FAILED test_data.py::test_list - AssertionError: assert ['a', 'a', 'a', 'a', ...
FAILED test_data.py::test_dict - AssertionError: assert {'a': 1, 'b': 2, 'c':...
========================= 5 failed, 1 passed in 0.12s =========================
```
整个输出结果可以分为以下几个部分：
1. 执行环境：版本、根目录、用例数量
```bash
platform win32 -- Python 3.11.14, pytest-9.0.2, pluggy-1.6.0
rootdir: E:\Code\pytest_project
collected 6 items
```
2. 执行过程：文件名称、用例结果、执行进度
```bash
test_base.py .F                                                          [ 33%]
test_data.py FFFF                                                        [100%]
```
3. 失败详情：用例内容、断言提示
```bash
__________________________________ test_int ___________________________________

    def test_int():
        a = 1
        b = 2
>       assert a == b
E       assert 1 == 2

test_data.py:4: AssertionError
```
4. 整体摘要： 结果情况、结果数量、花费时间
```bash
=========================== short test summary info ===========================
FAILED test_base.py::test_fail - assert False
FAILED test_data.py::test_int - assert 1 == 2
FAILED test_data.py::test_str - AssertionError: assert 'aaaaaa' == 'aaaabc'
FAILED test_data.py::test_list - AssertionError: assert ['a', 'a', 'a', 'a', ...
FAILED test_data.py::test_dict - AssertionError: assert {'a': 1, 'b': 2, 'c':...
========================= 5 failed, 1 passed in 0.12s =========================
```

**用例结果缩写**：

| 缩写  | 单词      | 含义              |
| --- | ------- | --------------- |
| .   | passed  | 通过              |
| F   | failed  | 失败（用例执行时报错）     |
| E   | error   | 出错（fixture执行报错） |
| s   | skipped | 跳过              |
| X   | xpassed | 预期外的通过（不符合预期）   |
| x   | xfailed | 预期内的失败（符合预期）    |

## 1.3 用例规则
### 1.3.1 用例的发现规则
测试框架在识别、加载用例的过程，称之为：**用例发现**。
pytest的用例发现步骤：
1. 遍历所有的目录，例外：`venv`和`.`**开头**的目录（Linux和MAC规定`.`开头为隐藏文件）
2. 打开所有`test_`**开头**或是`_test`**结尾**的python文件
3. 遍历所有的`Test`**开头**的类（类不是用例，但是类中的方法可以是，类本身用作用例分组）
4. 收集所有的`test_`**开头**的**函数 或者 方法**
```python
def add(a, b):  
    return a+b  
  
# 如果不使用类，那么就会有三个零散的方法：test_add_int、test_add_str、test_add_list
class TestAdd:  
    def test_int(self):  
        res = add(1, 2)  
        assert res == 3  
  
    def test_str(self):  
        res = add("1", "3")  
        assert res == "13"  
  
    def test_list(self):  
        res = add([1, 2, 3], [4, 5, 6])  
        assert res == [1, 2, 3, 4, 5, 6]
```
### 1.3.2 用例的内容规则
pytest对用例的要求：
1. 可调用的（函数、方法、类、对象）
2. 名字`test_`开头
3. 没有参数（参数有另外含义）
4. 没有返回值（默认为None），如果返回了非None的值则会警告：
```bash
============================== warnings summary ===============================
test_result.py::test_xfail
  D:\Anaconda\envs\pytest_env\Lib\site-packages\_pytest\python.py:170: PytestReturnNotNoneWarning: Test functions should return None, but test_result.py::test_xfail returned <class 'int'>.
  Did you mean to use `assert` instead of `return`?
  See https://docs.pytest.org/en/stable/how-to/assert.html#return-not-none for more information.
```
即用例只需要是一个符合命名规则的`Callable`对象（任何可以通过括号`()`调用的对象）就能够使用，即使有返回值也只是警告。**因为真正的测试逻辑是`assert`，pytest并不关心返回值，只关心`assert`是否失败**。

## 1.4 配置框架
配置可以改变pytest默认的规则，但是一般无需更改（约定优于配置）：
1. 命令参数
2. `.ini`配置文件
3. 环境变量（很少用）
所有的配置方式，可以一键获取
```bash
pytest -h
```
* 有哪些配置
* 分别有哪些方式
	* `-`开头：参数
	* 小写字母开头：`.ini`文件配置
	* 大写字母开头：环境变量
* 配置文件：`pytest.ini`（位置在根目录，文件名固定）
### 1.4.1 常用参数
* `-v`，以用例为单位，能获取到更详细的信息（用例名、所属类等）：
```bash
test/test_add.py::TestAdd::test_int PASSED                         [ 33%] 
test/test_add.py::TestAdd::test_str PASSED                         [ 66%] 
test/test_add.py::TestAdd::test_list FAILED                        [100%]
```
而不是以文件为单位：
```bash
test\test_add.py ..F                                               [100%]
```
* `-s`：在用例中正常地使用输入和输出（**默认情况下输入和输出被拦截**，因为在自动化测试中，不需要出现手动的输入输出）：
```python
def test_input():  
    a = input("请输入手机号")
```
如果使用`pytest`命令则会测试不通过，而加上`-s`参数后就可以正常输入信息
* `-x`：快速退出，即一旦出现失败用例，整个测试活动提前终止（冒烟测试）
* `-m`：用例筛选
#### 标记mark
标记可以让用例与众不同，进而可以让用例被区别对待
##### a. 用户的自定义标记
用户自定义标记：只能实现用例筛选，步骤：注册、标记、筛选
**① 注册**：参数/配置文件
```ini
# pytest.ini
[pytest]  
markers =  
 api: 接口测试  
 web: UI测试  
 ut: 单元测试  
 login: 登录相关  
 pay: 支付相关
```
注册好后，在终端输入`pytest --markers`，就可以显示自定义标记：
```bash
(pytest_env) PS E:\Code\pytest_project> pytest --markers
@pytest.mark.api: 接口测试

@pytest.mark.web: UI测试

@pytest.mark.ut: 单元测试

@pytest.mark.login: 登录相关

@pytest.mark.pay: 支付相关

```
**② 标记（使用）**：利用装饰器（命名与`--markers`参数显示的一致）
```python
class TestAdd:  
    @pytest.mark.api  
    def test_int(self):  
        res = add(1, 2)  
        assert res == 3  
  
    @pytest.mark.web
    def test_str(self):  
        res = add("1", "3")  
        assert res == "13"  
  
    @pytest.mark.pay  
    def test_list(self):  
        res = add([1, 2, 3], [4, 5])  
        assert res == [1, 2, 3, 4, 5, 6]
```
**③ 筛选**：使用参数`-m <markName>`实现筛选
```bash
(pytest_env) PS E:\Code\pytest_project> pytest -m pay
=========================== test session starts ==========================
platform win32 -- Python 3.11.14, pytest-9.0.2, pluggy-1.6.0
rootdir: E:\Code\pytest_project
configfile: pytest.ini
collected 4 items / 3 deselected / 1 selected                                                     

test\test_add.py F                                              [100%]
============================== FAILURES ================================== 
___________________________ TestAdd.test_list ____________________________
self = <test_add.TestAdd object at 0x000001CD93FA42D0>

    @pytest.mark.pay
    def test_list(self):
        res = add([1, 2, 3], [4, 5])
>       assert res == [1, 2, 3, 4, 5, 6]
E       assert [1, 2, 3, 4, 5] == [1, 2, 3, 4, 5, 6]
E
E         Right contains one more item: 6
E         Use -v to get more diff

test\test_add.py:22: AssertionError
========================= short test summary info =============================
FAILED test/test_add.py::TestAdd::test_list - assert [1, 2, 3, 4, 5] == [1, 2, 3, 4, 5, 6]
===================== 1 failed, 3 deselected in 0.30s ======================== 

```
一共四个用例，其中只执行了标记为`pay`的用例，其余三个处于`deselected`状态。
##### b. 框架的内置标记
框架内置标记能够为用例增加特殊执行效果，它和用户自定义标记的区别在于：
* 无需注册，直接使用
* 不仅可以筛选，还可以增加特殊效果
* 不同的标记，增加不同的特殊效果：
  ① `skip`：无条件跳过
  ② `skipif`：有条件跳过
  ③ `xfail`：预期失败
  ④ `paramentrize`：参数化，把一个用例变成多个用例（数据驱动测试）
  ⑤ `usefixtures`：使用`fixtures`

## 1.5 数据驱动测试
**数据驱动测试 = 参数化测试 + 数据文件**
**根据数据文件的内容，动态地决定用例的数量、内容**
```python
# test_data.py
def add(a, b):  
    return a+b
    
    
def read_csv(path):  
    f = open(path)  
    reader = csv.reader(f)  
    return list(reader)[1:]
    
    
@pytest.mark.data  
@pytest.mark.parametrize(  
    "a, b, c",  
    read_csv("data.csv")  
)  
def test_ddt(self, a, b, c):  
    res = add(int(a), int(b))  
    assert res == int(c)
```
其中数据来源于`data.csv`：
```csv
a, b, c  
1, 1, 2  
2, 3, 5  
3, 3, 6  
4, 4, 7
```
在`@pytest.mark.parametrize`中，定义了该参数化用例需要的参数，以及数据获取的方法。所以数据有4条，那么用例就有四个：
```bash
test/test_add.py::TestAdd::test_ddt[1- 1- 2] PASSED                         [ 25%]
test/test_add.py::TestAdd::test_ddt[2- 3- 5] PASSED                         [ 50%] 
test/test_add.py::TestAdd::test_ddt[3- 3- 6] PASSED                         [ 75%] 
test/test_add.py::TestAdd::test_ddt[4- 4- 7] FAILED                         [100%]

==================================== FAILURES ==================================== 
___________________________ TestAdd.test_ddt[4- 4- 7] ____________________________ 

self = <test_add.TestAdd object at 0x0000025A68A56B10>, a = '4', b = ' 4', c = ' 7'

    @pytest.mark.data
    @pytest.mark.parametrize(
        "a, b, c",
        read_csv("data.csv")
    )
    def test_ddt(self, a, b, c):
        res = add(int(a), int(b))
>       assert res == int(c)
E       AssertionError: assert 8 == 7
E        +  where 7 = int(' 7')

test\test_add.py:41: AssertionError
============================ short test summary info ============================= 
FAILED test/test_add.py::TestAdd::test_ddt[4- 4- 7] - AssertionError: assert 8 == 7
=================== 1 failed, 3 passed, 4 deselected in 0.17s ==================== 
```

## 1.6 夹具fixture
夹具：在用例执行之前、执行之后，自动运行代码
场景：
* 之前：加密参数 / 之后：解密结果
* 之前：启动浏览器 / 之后：关闭浏览器
* 之前：注册、登录账号 / 之后：删除账号
> [!问题]
> **Q：**  pytest的夹具fixture作用于用例前后，它是一种钩子机制。而这种机制在很多框架中都有体现：像FastAPI的中间件分别作用在HTTP报文的处理响应前后，LangChain的中间件作用于智能体一段会话的前后，这体现了一种怎样的开发共性？
> **A：** 这是软件开发中的**拦截/包装模式**或**AOP（面向切面编程）**，这种模式的本质特征在于横切关注点，处理与主业务逻辑正交的功能（如日志、鉴权、资源管理等），并且能够实现可插拔。
### 1.6.1 创建fixture
```python
@pytest.fixture  
def f():  
    # 前置操作  
    yield  # 开始执行用例，此处可以实现夹具和用例的信息传递
    # 后置操作（用例没有结果，所以无需接收信息）
```
1. 创建函数
2. 添加装饰器
3. 添加`yield`关键字（虽然python允许函数中存在多个`yield`，但是pytest框架严格要求只能有一个）
### 1.6.2 使用fixture
1. 在用例的参数列表中，加入fixture名字
2. 给**用例**加上`usefixtures`标记（仅适合少量用例需要夹具的情况）
```python
def test_1(f):  
    pass  

@pytest.mark.usefixtures("f")  
def test_2():  
    pass
```
### 1.6.3 fixture高级用法
1. 自动使用：`pytest.fxture(autouse=True)`
2. 依赖使用：把fixture要使用的fixture放入参数列表
```python
@pytest.fixture  
def f(ff):  
    print(datetime.now(), "用例开始执行")  
    # 前置操作  
    yield  
    # 后置操作  
    print(datetime.now(), "用例执行完毕")  
  
  
@pytest.fixture  
def ff():  
    print("ff被f使用")
```
3. 返回内容：接口自动化封装，接口关联
4. 范围共享：如果对于Web自动化测试，一个用例就要启动一次浏览器太耗时，所以可以通过共享，让多个用例使用同一浏览器。对于其他场景的资源处理也类似。
   - 默认范围：function，一个用例就是一个函数，所以默认不会共享
   - 全局范围：session，所有用例执行开始前，及所有用例结束后只执行一次fixture。但是python不同文件的命名空间独立，fixture在另一个文件中则pytest框架找不到。要实现真正的全局共享，需要把共享的fixture统一存放进`conftest.py`（建议在根目录）中。

## 1.7 插件管理
pytest插件生态是pytest特别的优势之处。插件分为两类：内置插件、第三方插件。而对于内置插件，因为无法卸载，所以为了更灵活地实现测试，pytest框架提供了插件的启用管理：
- 启用：`-p abc`
- 禁用：`-p no:abc`
插件的使用方式：
1. 参数
2. 配置文件
3. fixture
4. mark
### 常用的第三方插件
#### a. pytest-html
用途：生成HTML测试报告
安装：
```bash
pip install pytest-html
```
使用：
```bash
# 法一：使用命令参数
pytest --html=report.html --self-contained-html
```
每次写命令行参数太繁琐，所以可以使用配置文件
```ini
[pytest]

addopts = --html=report.html --self-contained-html
```
#### b. pytest-xdist
用途：分布式执行，通过多进程实现（有资源竞争、同步等问题的用例不能使用并发）
安装：
```bash
pip install pytest-xdist
```
使用：
```bash
pytest -n <进程数>
```
【注】只有在任务本身耗时较长，超出调用成本很多时才有意义。
#### c. pytest-returnfailures
用途：用例失败之后，重新执行，如果通过则也为通过（常用于和Web相关的测试）
安装：
```bash
pip install pytest-returnfailures
```
使用：
```bash
pytest -reruns <重新执行次数> --returns-delay <重新执行等待时间>
```
#### d. pytest-result-log
用途：把用例的执行结果记录到日志文件中
安装：
```bash
pip install pytest-result-log
```
使用：
```ini
log_file = ./logs/pytest.log
log_file_level = info
log_file_format = %(levelname)-8s %(asctime)s [%(name)s:&(lineno)s] : %(message)s 
log_file_date_format = %Y-%m-%d %H:%M:%S

; 记录用例执行结果
result_log_enable = 1
; 记录用例分割线
result_log_separator = 1
; 分割线等级
result_log_level_separator = warning
; 异常信息等级
result_log_level_verbose = info
```

# 二、实战
## 2.1 企业级测试报告生成框架——allure
pytest-html能够生成测试报告，但是不够专业，一般企业级测试报告使用allure生成（allure是一个测试报告的框架，能支持很多语言，所以安装时allure写在前，表示是allure的插件）
```bash
pip install allure-pytest
```
配置：
```bash
--alluredir=temps --clean-alluredir
```
生成报告：
```bash
allure generate -o <输出目录> -c <加载数据目录>
```
但是在命令行使用`allure`之前，需要安装allure命令行工具：
```bash
npm config set registry https://registry.npmmirror.com
npm install -g allure-commandline

allure --version
# 如果仍出现找不到的情况，就需要配置环境变量，把allure.cmd的目录添加到Path
```
![[Snipaste_2026-02-06_19-05-03.png]]
![[Snipaste_2026-02-06_19-14-10.png]]
代码自动生成测试报告：
```ini
[pytest]  
 
addopts = --alluredir=temps --clean-alluredir
```
在配置文件中生成测试报告所需的元数据，然后在`main.py`中将终端命令写进代码：
```python
import pytest  
import os  
  
pytest.main()  
# 执行命令  
os.system("allure generate -o report -c temps")
```
所以开始`pytest.main()`让pytest测试框架开启，并对所有用例测试，配置文件中的`addopts`先生成了必要元数据，然后下一行的`os.system`命令完成allure的测试报告生成。
## 2.2 allure的定制
allure支持对用例进行分组和关联（敏捷开发术语）
```python
@allure.epic     # 史诗 项目最大的模块测试，表示项目
@allure.feature  # 功能 项目中某种需求的测试，表示模块
@allure.story    # 故事 项目中某个场景的测试，表示场景
@allure.title    # 具体验证点 项目某个功能的用例
```
使用相同装饰器的用例，自动并入同一组。
**示例：** 4个用例在allure中的分组：
```python
@allure.epic("自动化测试")  
@allure.feature("pytest框架训练")  
@allure.story("mark标记和筛选")  
@allure.title("实现筛选的用例1")  
@pytest.mark.login  
def test_a():  
    pass  
  
  
@allure.epic("自动化测试")  
@allure.feature("pytest框架训练")  
@allure.story("fixture前置和后置")  
@allure.title("使用fixture用例")  
@pytest.mark.login  
def test_b():  
    pass  
  
  
@allure.epic("自动化测试")  
@allure.feature("pytest框架训练")  
@allure.story("mark标记和筛选")  
@allure.title("实现标记的用例2")  
@pytest.mark.login  
def test_c():  
    pass
    
@pytest.mark.login  
def test_rerun():  
    x = random.randint(0, 9)  
    assert x >= 9
```
其中用例`test_a`和`test_c`在同一story中，它们和`test_b`在同一feature中，`rest_rerun`和这三个用例独立：
![[Snipaste_2026-02-06_20-13-38.png]]
## 2.3 Web自动化测试实战
