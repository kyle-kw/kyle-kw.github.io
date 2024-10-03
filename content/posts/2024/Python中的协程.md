+++
title = 'Python中的协程'
date = 2024-08-18T11:23:17+08:00
tags = ["Python", "asyncio"]
+++

由于上周遇到了协程的问题，所以想着梳理一下Python中的协程。本文会提出两个问题：
1. 什么时候会切换协程？
2. 怎么可以进行非阻塞运行异步代码？

下面通过这两个问题介绍协程，先看一下基本介绍。

### 一、基本介绍

在Python中，协程是一种用于并发编程的高级抽象。协程允许你在执行过程中挂起和恢复函数的执行，这使得它们非常适合处理I/O密集型任务，如网络请求或文件操作。与线程相比，协程的开销更小，因为它们在同一线程内运行，不需要上下文切换。

#### 协程的基本概念

1. **定义**：
   协程是使用 `async def` 语法定义的函数。调用协程时不会立即执行，而是返回一个协程对象。

   ```python
   async def my_coroutine():
       print("Hello")
       await asyncio.sleep(1)
       print("World")
   ```

2. **调度**：
   协程通过事件循环（event loop）来调度。事件循环负责管理协程的执行，处理I/O操作并在需要时挂起和恢复协程。

3. **异步操作**：
   使用 `await` 关键字可以挂起协程的执行，等待某个异步操作完成。例如，`await asyncio.sleep(1)` 会挂起协程1秒钟。

#### 使用示例

以下是一个简单的示例，演示如何使用协程来处理异步任务：

```python
import asyncio

async def fetch_data():
    print("Fetching data...")
    await asyncio.sleep(2)  # 模拟I/O操作
    print("Data fetched!")

async def main():
    await asyncio.gather(fetch_data(), fetch_data(), fetch_data())

# 运行事件循环
asyncio.run(main())
```

#### 优势

1. **高效的I/O操作**：协程在等待I/O操作时不会阻塞线程，可以同时处理多个任务。
2. **轻量级**：协程的创建和切换比线程更轻便，适合大量并发任务。



### 二、抛出问题

#### 什么时候会切换协程？
简单的说，当协程执行到 `await` 表达式时，协程会挂起执行，控制权会返回给事件循环。事件循环会调度其他协程执行，直到 `await` 的操作完成。


#### 怎么可以在非阻塞运行异步代码？
当某一个协程遇到`await`时，怎么可以不切换协程？而是继续执行。这时就需要主动开启另一个协程来执行这段代码。

例如：
```python
import asyncio

async def fetch_data(i):
    print(f"Fetching data...{i}")
    await asyncio.sleep(i)  # 模拟I/O操作
    print(f"Data fetched{i}!")
    return i
    
async def demo1():
    await fetch_data(1)
    await fetch_data(2)

async def demo2():
    loop = asyncio.get_running_loop()
    tasks = [
        loop.create_task(fetch_data(1)),
        loop.create_task(fetch_data(2)),
    ]
    await asyncio.wait(tasks, return_when=asyncio.ALL_COMPLETED)

async def main():
    await demo1()  # 用时3s
    await demo2()  # 用时2s

# 运行事件循环
asyncio.run(main())
```

#### 如何实现一个双发任务？
双发任务意思为当执行一个任务超时时，启动另一个项目。基本思路是创建三个协程，分别为任务1、任务2和超时协程，当收到超时的信号时，启动另一个任务。

```python
import asyncio
from functools import partial

async def task1(aio_func, queue: asyncio.Queue):
    print('task1 init')

    message = await queue.get()
    if message == 'stop':
        print('task1 stop')
        return
    if message == 'start':
        print('task1 start')
        r = await aio_func()

        print('task1 over')
        return 1, r

async def task2(aio_func, queue: asyncio.Queue):
    print('task2 init')

    message = await queue.get()
    if message == 'stop':
        print('task2 stop')
        return
    if message == 'start':
        print('task2 start')
        r = await aio_func()
        print('task2 over')
        return 2, r

async def timeout_task(i):
    await asyncio.sleep(i)
    return -1, -1

async def double_run_task(aio_func1, aio_func2, timeout=10):
    """
    执行双发任务，当执行一个任务超时，会启动另外一个任务。
    """

    q1 = asyncio.Queue()
    q2 = asyncio.Queue()
    t1 = asyncio.create_task(task1(aio_func1, q1))
    t2 = asyncio.create_task(task2(aio_func2, q2))
    t_timeout = asyncio.create_task(timeout_task(timeout))

    tasks = [t1, t2, t_timeout]
    await_tasks = asyncio.as_completed(tasks)

    await q1.put('start')

    coro = next(await_tasks)
    idx, result = await coro

    if idx != -1:
        await q2.put('stop')
        return idx, result

    await q2.put('start')

    coro = next(await_tasks)
    idx, result = await coro

    # 尝试关闭另一个任务
    if idx == 1:
        t2.cancel()
        cancel_task = t2
    else:
        t1.cancel()
        cancel_task = t1

    try:
        await cancel_task
    except asyncio.CancelledError:
        pass

    return idx, result

async def aio_func_test(i):
    await asyncio.sleep(i)
    return 'aio_func_test finish'

async def aio_double_test():
    # 测试双发
    t1 = partial(aio_func_test, i=3)
    t2 = partial(aio_func_test, i=3)
    s = await double_run_task(t1, t2, timeout=1)
    print(s)

async def main():
    await aio_double_test()
    await asyncio.sleep(3)


if __name__ == '__main__':
    asyncio.run(main())
```


### 四、相关资料

asyncio [官方文档](https://docs.python.org/3/library/asyncio.htm)
