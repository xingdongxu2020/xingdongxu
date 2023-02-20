---
title:  "asyncio Futures"
---

# asyncion.Future

类Future, 是连接底层回调式(low-level callback-based)代码和上层异步/等待(high-level async/await)代码.

注: 在futures.py中
```python
try:
    import _asyncio
except ImportError:
    pass
else:
    # 实际的Future是底层_CFuture
    Future = _CFuture = _asyncio.Future
```

## Future概述

> PEP 3156

在asyncio中的`Future`类型有意与`concurrent.futures.Future`(PEP 3148)类型尽量相似, 但两者有所不同。

asyncion中`Future`接口方法:

- 状态
  - `cancelled()`
  - `done()`
  - `result()`: 1)返回set_result()结果; 2)引起set_exception()异常; 3)被取消时,引起CaneclledError; 4)尚未结束时,引起其它异常.
  - `exception()`: 1)set_result()时,返回None; 2)返回set_exception()异常; 3)被取消时,引起CaneclledError; 4)尚未结束时,引起其它异常.

- 自定义回调关系<sup>[1]</sup>
  - `add_done_callback(fn)`
  - `remove_done_callback(fn)`

- 驱动Future回调
  - `cancel()`当Future已经结束/取消时, 立即返回False; 此外, 尝试(Task可能对取消有额外处理)取消Future并返回True。
  - `set_result(result)`:
  - `set_exception(exception)`


> 注[1]: 回调`fn`将以Future对象为参数进行回调, 即`fn(self)`



讨论:

`Future`约等于(回调函数)+(状态管理)。`Future`本身不能实现异步功能。(实际的异步回调逻辑由`EventLoop`完成)



## Future代码研究

> asyncio.futures.Future

**对于Future的使用: `await Future()`或`yield from Future()`, 不可以`yield Future()`**

```python
class Future:
    _state = _PENDING   # _PENDING, _CANCELLED, _FINISHED

    _asyncio_future_blocking = False

    def __init__(self, *, loop=None):
        self._loop = loop or events.get_vent_loop()
        # 是一组callback+context的列表
        self._callbacks = []
        ...

    def __await__(self):
        if not self.done():
            self._asyncio_future_blocking = True
            yield self  # This tells Task to wait for completion.
        if not self.done():
            raise RuntimeError("await wasn't used with future")
        return self.result()  # May raise too.

    __iter__ = __await__  # make compatible with 'yield from'.

    def cancel(self, msg=None):
        if self._state != _PENDING:
            return False
        # 对于PENDING状态的future设置取消标识, 记录msg
        self._state = _CANCELLED
        self._cancel_message = msg

        self.__schedule_callbacks()
        return True

    def __schedule_callbacks(self):
        callbacks = self._callbacks[:]
        if not callbacks:
            return
        self._callbacks[:] = []
        for callback, ctx in callbacks:
            self._loop.call_soon(callback, self, context=ctx)

    def cancelled(self):
        return self._state == _CANCELLED

    def done(self):
        return self._state != _PENDING

    def result(self):
        ...

    def exception(self):
        ...

    def add_done_callback(self, fn, *, context=None):
        if self._state != _PENDING:
            self._loop.call_soon(fn, self, context=context)
        else:
            if context is None:
                context = contextvars.copy_context()
            self._callbacks.append((fn, context))

    def remove_done_callback(self, fn):
        # 过滤器的使用, 可以更加高效吗?
        filtered_callbacks = [(f, ctx)
                              for (f, ctx) in self._callbacks
                              if f != fn]
        removed_count = len(self._callbacks) - len(filtered_callbacks)
        if removed_count:
            self._callbacks[:] = filtered_callbacks
        return removed_count

    def set_result(self, result):
        ...

    def set_exception(self, exception):
        ...
```



## await 语句 (从Generator至await语法的变化)

### PEP 255-SimpleGenerators 关键字yield (2001-05)

```python
def g():
    i = 1
    yield i
    i += 1

a = g()		# a is a generator
next(a)		# out 1
next(a)		# raise StopIteration
```

- 首次调用函数返回的是`Generator g`对象
- 使用`next(g)`时. (此时, `a.next()`语句似乎时允许的, 现在已经不允许了)
  - 运行至`yield x`语句时, `return x`
  - 运行函数结尾时, `raise StopIteration`

### PEP 342 – Coroutines via Enhanced Generators (2005-05)

> [PEP 342](https://peps.python.org/pep-0342/):
>
> 协程是表达许多算法的自然方式，例如模拟、游戏、异步 I/O 和其他形式的事件驱动编程或协作式多任务处理。 Python的生成器函数几乎是协程——但不完全是——因为它们允许暂停执行以产生一个值，但不提供在执行恢复时传入的值或异常。它们也不允许在 try/finally 块的 try 部分暂停执行，因此使中止的协同程序难以自行清理

此时，generators尚存在的问题: 

- `yield`语句虽然可以实现暂停、实现向外传值, 但无法接收外部的新值. 另外, 无法在`try/finally`语句中执行
- `yield`无法穿过非generator的普通函数向上传递, 即使将嵌套的所有函数均以`yield`语句定义为generator, 上层函数也无法将值/异常直接传递给里层`yield`语句, 必须经过外层`yield`语句的处理判断。

该PEP主要工作

- `yield`扩展为表达式, 而不仅仅是一个声明. (为`yield`语句增加返回值)
- 为generator增加`send(value=None)`方法
  - 首次调用`send()`数值必须时`None`表示生成器的首次启动
  - 后续调用`send()`数值可以为任意值, 作为`yield`语句的返回值
- 为generator增加`throw(type,value=None,tb=None)`方法, 将在`yield`语句处引发异常`type`
- 为generator增加`close()`方法, 在`yield`语句中引发`GeneratorExit`异常
  - 生成器向上抛出`GeneratorExit`等效为`StopIteration`, 均表示生成器已经结束
- 底层支持所有生成器在gc之前`close()`方法会被调用。那么, `yield`语句可用于`try/finally`语句块中, 因为`yield`会被`close()`方法确保执行



### PEP 380 – Syntax for Delegating to a Subgenerator

> [PEP 380](https://peps.python.org/pep-0380/)
>
> Python生成器的局限性在于它只能向其直接调用者 yield。这意味着一段包含 yield 的代码不能像其他代码一样被提取出来并放入一个单独的函数中。执行这样的因式分解会导致被调用函数本身成为生成器，并且必需显对第二级生成器再次 yield

考虑当前generator再内外分层时, 处理`send()`/`throw()`等方法逻辑复杂

该PEP主要工作，定义`yield from <expr>`表达式语句:

- 表达式`expr`是一个迭代器iterator, 在该语句中将持续迭代直至产生最终结果. 在这个过程中, 如果`expr`内部的`yield`将直接与外层调用函数直接交互, 包含`yield from`语句的生成器(被称为"delegating generator")不参与交互.
- `expr`的返回值`return value`将作为`yield from <expr>`的返回值
  - 如果`expr`是一个迭代器, 返回的将是`StopIteration(value)`。 扩展StopIteration以承载返回数据(对于所有迭代器均适用)







```python
RESULT = yield from EXPR

# yield from expression is semantically equivalent to the followings:
_i = iter(EXPR)
try:
    _y = next(_i)
except StopIteration as _e:
    _r = _e.value
else:
    while 1:
        try:
            _s = yield _y
        except GeneratorExit as _e:
            try:
                _m = _i.close
            except AttributeError:
                pass
            else:
                _m()
            raise _e
        except BaseException as _e:
            _x = sys.exc_info()
            try:
                _m = _i.throw
            except AttributeError:
                raise _e
            else:
                try:
                    _y = _m(*_x)
                except StopIteration as _e:
                    _r = _e.value
                    break
        else:
            try:
                if _s is None:
                    _y = next(_i)
                else:
                    _y = _i.send(_s)
            except StopIteration as _e:
                _r = _e.value
                break
RESULT = _r
```



### PEP 492 – Coroutines with async and await syntax（2015-04）

> [PEP 3152 - Cofunctions (2009-02)](https://peps.python.org/pep-3152/)
>
> [PEP 492](https://peps.python.org/pep-0492/)









### PEP 3156 – Asynchronous IO Support Rebooted: the “asyncio” Module(2012-12)

> [PEP 3153 – Asynchronous IO support (2011-05)](https://peps.python.org/pep-3153/)
>
> [PEP 3156](https://peps.python.org/pep-3156/)



