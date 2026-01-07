asyncio、异步编程等术语都是由协程这个概念所发展出来的，所以学习路线：协程 -> 使用asyncio模块进行异步编程 -> 实战案例

# 异步编程

## 1. 协程`Coroutine`

协程不是计算机中存在的，计算机只提供了进程和线程，而是由程序员人为创造的，它也被称作微线程，是一种用户态内的上下文切换技术：用一个线程去实现在代码之间来回切换地运行，即在一个线程中如果遇到IO等待时间，线程不会死等，而是会利用空闲时间执行其他任务，例如：
```python
def func1():
	print(1)
	...
	print(2)
	
def func2():
	print(3)
	...
	print(4)
	
func1()
func2()
```
实现协程有以下方法：
* 使用第三方库，如`greenlet`这种早期模块。
* `yield`关键字
* python 3.4+引入的`asyncio`装饰器
* python 3.5+引入的`async`、`await`关键字（最主流，且官方也最推荐）
### 1.1 通过`greenlet`实现协程

```python
from greenlet import greenlet  
  
def func1():  
    print(1)  # 第2步：输出 1    
    gr2.switch()  # 第3步：切换到 func2 函数  
    print(2)  # 第6步：输出 2    
    gr2.switch()  # 第7步：切换到 func2 函数，从上一次执行的位置继续向后执行  
  
  
def func2():  
    print(3)    # 第4步：输出3  
    gr1.switch()  # 第5步：切换到 func1 函数，从上一次执行的位置继续向后执行
    print(4)  # 第8步：输出 4  
  
gr1 = greenlet(func1)  
gr2 = greenlet(func2)  
gr1.switch()  # 第1步：去执行 func1 函数
```
### 1.2 通过`yield`关键字实现协程（不会使用）
```python
def func1():  
    yield 1  
    yield from func2()  
    yield 2  
  
  
def func2():  
    yield 3  
    yield 4  
  
  
f1 = func1()  # 执行生成器函数返回了一个生成器，被f1接收
for item in f1:  
    print(item)
```
### 1.3 通过`asyncio`装饰器实现协程
```python
import asyncio  
  
  
@asyncio.coroutine  
def func1():  
    print(1)  
    # 遇到IO耗时操作，自动化切换到tasks中的其他任务  
    yield from asyncio.sleep(2)  
    print(2)  
  
  
@asyncio.coroutine  
def func2():  
    print(3)  
    # 遇到IO耗时操作，自动化切换到tasks中的其他任务  
    yield from asyncio.sleep(2)  
    print(4)  
  
tasks = [  
    asyncio.ensure_future(func1),  
    asyncio.ensure_future(func2)  
]  
  
loop = asyncio.get_event_loop()  
loop.run_until_complete(asyncio.wait(tasks))
```
前面两种方法是手动切换，而这种方法是自动切换上下文的。
但是在Python 3.10及以上版本中，`asyncio.coroutine`装饰器已经被移除，建议使用`async def`和`await`来定义协程；同时`asyncio.ensure_future`在Python 3.10中仍然可用，但推荐使用`asyncio.create_task`；`loop.run_until_complete()`也改为了`asyncio.run()`；`yield from`改为了`await`。
**在新版本中：始终使用`async def`定义异步函数；用`asyncio.gather()`并行执行多个任务；用`async with`管理异步资源；用`await`调用异步函数**
### 1.4 通过`async`、`await`关键字实现协程（最常用）
```python
import asyncio

async def func1():
    print(1)
    # 使用await等待异步操作
    await asyncio.sleep(2)
    print(2)

async def func2():
    print(3)
    await asyncio.sleep(2)
    print(4)

async def main():
    # 现代写法：创建任务并等待
    task1 = asyncio.create_task(func1())
    task2 = asyncio.create_task(func2())
    
    # 等待两个任务都完成
    await task1
    await task2

# Python 3.7+推荐使用asyncio.run()
asyncio.run(main())
```
## 2. 事件循环

理解成是一个死循环，会去检测并执行某些代码，不断地循环检测任务列表，找到可执行和已完成的任务返回到可执行列表和已完成列表中。然后处理可执行列表中的任务，并移除已完成的任务。
通过`async def`定义协程函数，调用协程函数返回一个协程对象。
```python
async def func():
	print(1)
	
result = func()
```
但是调用协程函数创建协程对象，函数内部代码不会执行（类似`yield`关键字定义生成器函数，返回不会立刻执行的生成器对象）
**如果想要运行协程函数内部代码，必须要将协程对象交给事件循环来处理。**
```python
import asyncio

async def func():
	print("111")
	
result = func()

# 早期版本，显示获取loop
#	loop = asyncio.get_event_loop()
#	loop.run_until_complete(result)

# 3.7版本后有了更简便的写法，上下两种写法等价
asyncio.run(result)
```
## 3. `await`
`await`后面要跟可等待对象：`Awaitable`，如协程函数返回的协程对象就是一个可等待对象。`Awaitable`主要有三种：协程`Ciroutine`、任务`Task`和`Future`。
协程就是协程函数返回的协程对象；任务`Task`是通过`asyncio.create_task()`等函数将协程包装成的任务，也是可等待的；`Future`是一种底层级的可等待对象，通常无需直接创建。
```python
import asyncio  

async def func():  
    print('hello')  
    response = await asyncio.sleep(2)  
    print("结束：", response)  
  
asyncio.run(func())
```
在`asyncio.run(func())`把`func()`返回的协程对象加入到事件循环列表中，内部会先执行`print('hello')`，然后遇到等待（假设是I/O，因为只有**等待的对象`awaitable`尚未完成**时才会主动挂起并将控制权交换给事件循环），事件循环就不会继续执行这个函数了，转而去执行其他任务。当等待结束，事件循环就会将该任务放入就绪队列，在适当时机恢复执行。
```python
import asyncio  
  
async def others():  
    print("start")  
    await asyncio.sleep(2)  
    print("end")  
    return "返回值"  
  
async def func():  
    print("执行协程函数内部代码")  
    # 遇到IO操作挂起当前协程，等IO操作完成后在继续往下执行  
    # 当前协程挂起时，事件循环可以去执行其他协程  
    response = await others()  
    print("I/O请求结束，结果为：", response)  
  
asyncio.run(func())
```
## 4. `Task`对象
`Task`的官方文档的介绍，它是用于协程并发调度的类：
```text
Tasks are used to schedule coroutines concurrently.

When a coroutine is wrapped into a Task with functions like asyncio.create_task() the coroutine is automatically scheduled to run soon.
```
总之它就是能够在事件循环中，添加多个任务。通过`asyncio.create_task(Coroutine)`的方式创建`Task`对象。这样可以让协程加入事件循环中等待被调度执行。除了使用`asyncio.create_task()`函数外，还可以用底层级`loop.create_task()`或`ensure_future()`函数。另外不建议手动实例化`Task`对象。
示例1（不太常用）：
```python
import asyncio  

async def func():  
    print(1)  
    await asyncio.sleep(2)  
    print(2)  
    return "return"  
  
async def main():  
    print("main开始")  
    # 将协程封装到Task对象中，并添加进事件循环任务列表，等待执行（默认就绪）  
    task1 = asyncio.create_task(func())  
    # 将协程封装到Task对象中，并添加进事件循环任务列表，等待执行（默认就绪）  
    task2 = asyncio.create_task(func())  
    print("main结束")  
      
    # 当执行协程遇到I/O操作时，会自动化切换执行其他任务。  
    # 此处的await是等待相对应的协程全部执行完毕并获取结果  
    ret1 = await task1  
    ret2 = await task2  
    print(ret1, ret2)
    
"""输出结果为：
main开始
main结束
1
1
2
2
return return
"""
```
示例2（常用，因为不用多次写`await`，并用集合接收结果）：
```python
import asyncio  
  
async def func():  
    print(1)  
    await asyncio.sleep(2)  
    print(2)  
    return "return"  
  
async def main():  
    print("main开始")  
    # 创建协程，将协程封装到Task对象中，并添加进事件循环任务列表，等待执行（默认就绪）  
    task_list = [  
        asyncio.create_task(func()),  
        asyncio.create_task(func(), name="n1")  
    ]  
    print("main结束")  
  
    # pending是还未完成的中间结果（没什么用），返回结果都放在done集合里面  
    # 如timeout=1（默认为None）但是完成需要2秒，done就是空的，中间结果在pending中  
    done, pending = await asyncio.wait(task_list, timeout=None)  
    print(done)  
  
asyncio.run(main())
"""输出结果为：
main开始
main结束
1
1
2
2
{<Task finished name='Task-2' coro=<func() done, defined at E:\Code\asyncProject\task_test2.py:3> result='return'>, <Task finished name='n1' coro=<func() done, defined at E:\Code\asyncProject\task_test2.py:3> result='return'>}
"""
```
其中可以看到`done`是一个有两个元素的集合（因为`task_list`定义了两个任务），里面有一个`Task finished name`字段，两个元素的值分别为`Task-2`和`n1`，而`Task-1`就是`asyncio.run(main())`这个主协程。
示例3，在事件循环创建前调用`create_task()`：
```python
import asyncio  
  
async def func():  
    print(1)  
    await asyncio.sleep(2)  
    print(2)  
    return "return"  

task_list = [  
	asyncio.create_task(func()),  
	asyncio.create_task(func(), name="n1")  
]  
  
done, pending = asyncio.run(asyncio.wait(task_list))
print(done)
"""输出结果为：
Traceback (most recent call last):
  File "E:\Code\asyncProject\task_test3.py", line 12, in <module>
    asyncio.create_task(func()),
    ^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "D:\Anaconda\envs\asyncProject\Lib\asyncio\tasks.py", line 381, in create_task
    loop = events.get_running_loop()
           ^^^^^^^^^^^^^^^^^^^^^^^^^
RuntimeError: no running event loop
sys:1: RuntimeWarning: coroutine 'func' was never awaited
"""
```
示例4，上例的修改（`task_list`不再放`Task`而是`Coroutine`）**旧版本适用**：
```python
import asyncio  
  
async def func():  
    print(1)  
    await asyncio.sleep(2)  
    print(2)  
    return "return"  

task_list = [  
	func(), 
	func()
]  
  
done, pending = asyncio.run(asyncio.wait(task_list))
print(done)
"""输出结果为：
Traceback (most recent call last):
  File "E:\Code\asyncProject\task_test3.py", line 34, in <module>
    done, pending = asyncio.run(asyncio.wait(task_list))
                    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "D:\Anaconda\envs\asyncProject\Lib\asyncio\runners.py", line 190, in run
    return runner.run(main)
           ^^^^^^^^^^^^^^^^
  File "D:\Anaconda\envs\asyncProject\Lib\asyncio\runners.py", line 118, in run
    return self._loop.run_until_complete(task)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "D:\Anaconda\envs\asyncProject\Lib\asyncio\base_events.py", line 654, in run_until_complete
    return future.result()
           ^^^^^^^^^^^^^^^
  File "D:\Anaconda\envs\asyncProject\Lib\asyncio\tasks.py", line 425, in wait
    raise TypeError("Passing coroutines is forbidden, use tasks explicitly.")
TypeError: Passing coroutines is forbidden, use tasks explicitly.
sys:1: RuntimeWarning: coroutine 'func' was never awaited
"""
```
问题根源在于Python 3.8开始弃用`asyncio.wait()`直接传入协程对象，直接传递会引发`DeprecationWarning`，到Python 3.11正式禁止该行为，必须显式地将`Corouttine`包装为`Task`，传递协程对象会明确地抛出`TypeError`报错。
## 5. `asyncio.Future`对象
`Future`对象是异步编程中底层的，一般不会直接使用的对象，它是`Task`对象的基类，它代表异步操作的最终结果。官网对它的解释为：
```texr
A `Future` is a special **low-level** awaitable object that represents an **eventual result** of an asynchronous operation.
```
对于`await`，`Future`内部维护了一个状态变量`_state`，里面有不同的值`FINISHED`、`PENDING`、`CANCELLED`等，它们用于识别任务状态，比如如果`_state`值为`FINISHED`那么代表任务完成，`await`就拿到结果继续往下执行了（这是简化形容，其实内部状态由标志位组合来管理，在判断`Future`是否完成时还需要结合`_result`、`_exception`等属性判断）。
`Future`和`Task`的区别：`Future`是一个通用的”空容器“或”协议“，它什么都不做，而`Task`是一个专为驱动协程设计的”自动执行容器“。
## 6. `concurrent.futures.Future`对象
`concurrent.futures.Future`与`asyncio.Future`没有关系，它是在未来需要使用线程池、进程池来实现异步操作时会使用的。P9











