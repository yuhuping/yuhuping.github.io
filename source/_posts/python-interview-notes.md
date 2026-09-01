---
title: "Python 八股"
date: 2026-09-01 09:08:35
tags:
  - Python
  - Python基础
  - 面试八股
  - 迭代器
  - asyncio
categories:
  - 学习笔记
---

### **-Python 浅拷贝与深拷贝：**
区别：[[0] * n] 与 [[0] for _ in range(n)]的区别
前者是浅拷贝共用一个内存地址一改都改，后者深拷贝

### **-Python 之迭代器、生成器：**

迭代器：一个控制数据流的**对象**，注意不是可迭代对象（如 list）—迭代器 __next__一下后无法回到之前数据；使用方法-一般用 iter(list)这样子创建，实际上用户一般不明显使用，而是通过使用**内置高阶函数**-map() filter() reverse()间接使用到。

生成器：带 yield 的函数, 调用一下返回一下
```
def stream_chat(prompt):
    _#_ 模拟调用 _LLM_ 流式 _API_
    for chunk in ["Hello", " ", "World", "!"]:  _#_ 实际是网络 _IO_
        yield chunk
_#_ ✅ 正常用法：立即消费，边拿边打印
def main():
    print("🤖 AI: ", end="", flush=True)
    for chunk in stream_chat("Hi")
        print(chunk, end="", flush=True)  _# CPU_ 在这里极速运行，写完立即进入下一次迭代
        _#_ 如果 _chunk_ 还没从网络到达（假设是网络_IO_），
        _# for_ 循环会阻塞在迭代器的 ___next__()_ 上，线程挂起，_CPU_ 彻底空闲。
    print("\n✅ 输出完毕")
if __name__ == "__main__":
    main()
```

### **-Python之async与asyncio.run**
async def定义的对象是协程对象，
```
async def task_a():
    await asyncio.sleep(2)  _#_ 模拟耗时 _IO_（比如请求 _API_）
    return "A的结果"

async def task_b():
    await asyncio.sleep(1)  _#_ 模拟耗时 _IO_
    return "B的结果"

async def main():
    _#_ 并发运行 _task_a_ 和 _task_b_（两个协程同时被事件循环管理）
    results = await asyncio.gather(task_a(), task_b())
    print(f"最终结果: {results}")

asyncio.run(main())
```
这样子使用的目的是让CPU知道遇到await可以去执行其他**协程（不是线程！）**

### **-Python Other****：
- 垃圾回收机制：引用计数，当对象引用数为0就释放内存
- GIL全局锁：导致使用多线程同一时刻只有一个在CPU运行，可用multiprocessing绕过全局锁（如我们MinerU pdf处理）
- 协程 是一种比线程更加轻量级的用户态并发设计模式
- 依赖注入：一种设计模式，被调用方要求调用方把需要的依赖对象传过来
- 装饰器：本质是一种特别函数，使不修改原函数代码下拓展新功能，如日志打印、超时重试等等