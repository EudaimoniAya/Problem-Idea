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
配置可以改变pytest默认的规则