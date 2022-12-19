
# EventLoop如何实现

## EventLoop Methods概述

参考[PEP-3156](https://peps.python.org/pep-3156/#event-loop-methods-overview)

**标准Loop方法**

- 启动(starting), 停止(stopping), 关闭(closing): 
  - `run_forever()`
  - `run_until_complete()`
  - `stop()`
  - `is_running()`
  - `is_closed()`
  - `close()`
  - 其它补充：
    - `async shutdown_asyncgens()`
    - `async shutdown_default_executor()`
- 基础callback方法: 
  - `call_soon()` -> 返回类型**Handle**
  - `call_later()` -> 返回类型**TimerHandle**
  - `call_at()` -> 返回类型**TimerHandle**
  - `time()`
- 线程交互方法: 
  - `call_soon_threadsafe()`
  - `run_in_executor()`
  - `set_default_executor()`
- Future与Task:
  - `create_future()` -> 返回**Future**类型(更丰富的Handle)
  - `create_task()` -> 返回**Task**类型
  - `set_task_factory()`
  - `get_task_factory()`
- 异常处理: 
  - `get_exception_handler()`
  - `set_exception_handler()`
  - `default_exception_handler()`
  - `call_exception_handler()`
- Debug模式: 
  - `get_debug()`
  - `set_debug()`

- Network I/O域名解析方法: 
  - `async getaddrinfo()`
  - `async getnameinfo()`
- Network I/O连接方法: 
  - `async create_connection()`
  - `async create_server()` -> 返回类型**AbstractServer**
  - `async create_datagram_endpoint()`
  - 其它补充：
    - `async sendfile()`
    - `async start_tls()`
    - `async create_unix_connection()`
    - `async create_unix_server()`
- socket封装方法: 
  - `async sock_recv()`(`async sock_recv_into()`)
  - `async sock_sendall()`
  - `async sock_connect()`
  - `async sock_accept()`
  - `async sock_sendfile()`

**部分支持功能, UNIX支持**

- I/O 补充操作: 
  - `add_reader()`
  - `remove_reader()`
  - `add_writer()`
  - `remove_writer()`
- 信号处理: 
  - `add_signal_handler()`
  - `remove_signal_handler()`
- 管道与子进程: 
  - `async connect_read_pipe()`
  - `async connect_write_pipe()`
  - `async subprocess_shell()`, 
  - `async subprocess_exec()`
  - 

## 抽象Handle, TimerHandle

### Handle

处理回调callback的结构

1. `Handle(callback, args, loop, context=None)`, 属性
   - `_context`
   - `_loop`
   - `_callback`
   - `_args`
   - `_cancelled`
   - `_source_traceback`: 在debug模式下记录调用栈信息
2. 方法
   - `cancel()`
   - `canceled()`
   - `_run()`: 供EventLoop使用的内部方法
   

### TimerHandle(Handle)

1. `TimerHandle(when, callback, args, loop, context=None)`
   - 额外的`_when`
   - 额外的`_scheduled`
2. 补充方法
   - `when()`计划执行时间


## BaseEventLoop(AbstractEventLoop)

### running

#### 运行大循环

```python
class BaseEventLoop:
    def run_forever(self):
        """Run until stop() is called."""
        self._check_closed()
        self._check_running()
        self._set_coroutine_origin_tracking(self._debug)
        self._thread_id = threading.get_ident()

        # 为当前loop设置异步生成器的firstiter, finalizer
        old_agen_hooks = sys.get_asyncgen_hooks()
        sys.set_asyncgen_hooks(firstiter=self._asyncgen_firstiter_hook,
                                finalizer=self._asyncgen_finalizer_hook)
        try:
            events._set_running_loop(self)
            while True:
                # 循环执行
                self._run_once()
                if self._stopping:
                    break
        finally:
            self._stopping = False
            self._thread_id = None
            events._set_running_loop(None)
            self._set_coroutine_origin_tracking(False)
            sys.set_asyncgen_hooks(*old_agen_hooks)
```

#### 单步执行动作

```python
class BaseEventLoop:
    def _run_once(self):
        # 清理定时任务(堆)
        # 1. 定时任务取消数量>50%, 且大于>50个时, 清理重建堆
        # 2. 清理堆顶已取消的任务
        sched_count = len(self._scheduled)
        if (sched_count > _MIN_SCHEDULED_TIMER_HANDLES and
            self._timer_cancelled_count / sched_count >
                _MIN_CANCELLED_TIMER_HANDLES_FRACTION):
            new_scheduled = []
            for handle in self._scheduled:
                if handle._cancelled:
                    handle._scheduled = False
                else:
                    new_scheduled.append(handle)

            heapq.heapify(new_scheduled)
            self._scheduled = new_scheduled
            self._timer_cancelled_count = 0
        else:
            while self._scheduled and self._scheduled[0]._cancelled:
                self._timer_cancelled_count -= 1
                handle = heapq.heappop(self._scheduled)
                handle._scheduled = False

        # ready是已完成的非定时任务队列
        timeout = None
        if self._ready or self._stopping:
            timeout = 0
        elif self._scheduled:
            # 依据定时任务计算合适的timeout时长, 不小于0, 不大于24*3600
            when = self._scheduled[0]._when
            timeout = min(max(0, when - self.time()), MAXIMUM_SELECT_TIMEOUT)

        # _selector留个loop的具体实现定义, select
        # _process_events同样留给loop的具体实现定义
        event_list = self._selector.select(timeout)
        # select得到的event_list向self._ready添加了相应的callback
        self._process_events(event_list)

        # Handle 'later' callbacks that are ready.
        end_time = self.time() + self._clock_resolution
        while self._scheduled:
            handle = self._scheduled[0]
            if handle._when >= end_time:
                break
            handle = heapq.heappop(self._scheduled)
            handle._scheduled = False
            self._ready.append(handle)

        # ～～此处是callback被真正调用的部分～～.
        ntodo = len(self._ready)
        for i in range(ntodo):
            handle = self._ready.popleft()
            if handle._cancelled:
                continue
            if self._debug:
                try:
                    self._current_handle = handle
                    t0 = self.time()
                    # ～～调用callback～～, 所有callback都是Handle的形式
                    handle._run()
                    dt = self.time() - t0
                    if dt >= self.slow_callback_duration:
                        logger.warning('Executing %s took %.3f seconds',
                                        _format_handle(handle), dt)
                finally:
                    self._current_handle = None
            else:
                # ～～调用callback～～, 所有callback都是Handle的形式
                handle._run()
        handle = None  # Needed to break cycles when an exception occurs.
```

#### SelectorEventLoop实现

```python



# 处理事件方法
class BaseSelectorEventLoop(BaseEventLoop):
    def __init__(self)
        # _selector具体实现
        self._selector = selectors.SelectSelector()
        ...

    def _process_events(self, event_list):
        for key, mask in event_list:
            # key是SelectorKey对象, key.data是非透明的自定义数据
            fileobj, (reader, writer) = key.fileobj, key.data
            if mask & selectors.EVENT_READ and reader is not None:
                if reader._cancelled:
                    self._remove_reader(fileobj)
                else:
                    # 如果文件对象的读就绪, 那么调用一次reader的callback
                    self._add_callback(reader)
            if mask & selectors.EVENT_WRITE and writer is not None:
                if writer._cancelled:
                    self._remove_writer(fileobj)
                else:
                    self._add_callback(writer)
```

### 基本的callback

call_soon(), call_later(), call_at()

```python
class BaseEventLoop:
    def _call_soon(self, callback, args, context):
        # 为callback创建Handle
        handle = events.Handle(callback, args, self, context)
        ...
        # 将callback添加至ready队列
        self._ready.append(handle)
        return handle

    def call_soon(self, callback, *args, context=None):
        self._check_closed()
        ...
        handle = self._call_soon(callback, args, context)
        ...
        return handle

    def call_at(self, when, callback, *args, context=None):
        self._check_closed()
        ...
        timer = events.TimerHandle(when, callback, args, self, context)
        ...
        # 将定时任务的handle添加到scheduled队列
        heapq.heappush(self._scheduled, timer)
        timer._scheduled = True
        return timer
```

### task与futures

create_future(), create_task(), set_task_factory(), get_task_factory()


```python
class BaseEventLoop:
    def create_future(self):
        return futures.Future(loop=self)

    def create_task(self, coro, *, name=None):
        self._check_closed()
        if self._task_factory is None:
            task = tasks.Task(coro, loop=self, name=name)
            ...
        else:
            task = self._task_factory(self, coro)
            tasks._set_task_name(task, name)
        return task
```

### Network I/O方法

- Network I/O连接方法: 
  - `async create_connection()`
  - `async create_server()` -> 返回类型**AbstractServer**
  - `async create_datagram_endpoint()`
  - 其它补充：
    - `async sendfile()`
    - `async start_tls()`
    - `async create_unix_connection()`
    - `async create_unix_server()`

#### create_connection

```python
async def create_connection(
    self, protocol_factory, host=None, port=None,
            *, ssl=None, family=0,
            proto=0, flags=0, sock=None,
            local_addr=None, server_hostname=None,
            ssl_handshake_timeout=None,
            happy_eyeballs_delay=None, interleave=None):
```

方法说明
- 与一个TCP server建立连接
- `protocol_factory`用于产生协议的工厂调用方法
- 成功建立时, 返回(传输Transport, 协议Protocol)元组


参数含义
- `ssl`: 指定ssl时会创建一个SSL/TLS
- `server_hostname`: 设置证书将要匹配的主机名, 只有ssl不是None时需要, 默认使用host值
- `ssl_handshake_timeout`, 放弃前等待TLS握手的时长, 默认60
- `sock`, 给定sock时则必须是一个已存在已连接并被传输使用的socket对象, 此时下面参数不能指定
- `family`, `proto`, `flags`是可选的地址族, 协议和标志. 会被传入`getaddrinfo()`对host进行解析, 指定是必须是socket模块常量
- `happy_eyeballs_delay`, 用以启动Happy Eyeballs地址选择算法
- `interleave`, 控制主机名解析为多个IP地址时的重排序.
- `local_addr`给定时, 是用以本地绑定的(local_host, local_port)元组


处理逻辑

1. 解析地址, `await self.run_in_executor()`调用socket模块的`getaddrinfo()`方法
   - `getaddrinfo(host, port, family=0, type=0, proto=0, flags=0)`
   - 指定的family, type, proto参数用以缩小地址列表
   - 输出5元组`(family, type, proto, canonname, sockaddr)`
   - 输出的sockaddr对于AF_INET是`(address, port)`, 对于AF_INET6是`(address, port, flowinfo, scope_id)`用于调用`socket.connect()`

2. 建立sock, `await self._connect_sock(exceptions, addrinfo, laddr_infos)`
    ```python
        async def _connect_sock(self, exceptions, addr_info, local_addr_infos=None):
        """Create, bind and connect one socket."""
        my_exceptions = []
        exceptions.append(my_exceptions)
        family, type_, proto, _, address = addr_info
        sock = None
        try:
            # 创建socket
            sock = socket.socket(family=family, type=type_, proto=proto)
            # 非阻塞模式
            sock.setblocking(False)
            if local_addr_infos is not None:
                for _, _, _, _, laddr in local_addr_infos:
                    try:
                        # 指定本地地址时进行bind操作
                        sock.bind(laddr)
                        break
                    except OSError as exc:
                        msg = (
                            f'error while attempting to bind on '
                            f'address {laddr!r}: '
                            f'{exc.strerror.lower()}'
                        )
                        exc = OSError(exc.errno, msg)
                        my_exceptions.append(exc)
                else:  # all bind attempts failed
                    raise my_exceptions.pop()
            # 调用loop方法, 使用sock与address连接
            await self.sock_connect(sock, address)
            return sock
        except OSError as exc:
            my_exceptions.append(exc)
            if sock is not None:
                sock.close()
            raise
        except:
            if sock is not None:
                sock.close()
            raise
    ```

3. `sock_connect`方法的`BaseSelectorEventLoop`实现
    ```python
    class BaseSelectorEventLoop(BaseEventLoop):
        async def sock_connect(self, sock, address):
            _check_ssl_socket(sock)
            if self._debug and sock.gettimeout() != 0:
                raise ValueError("the socket must be non-blocking")

            if not hasattr(socket, 'AF_UNIX') or sock.family != socket.AF_UNIX:
                resolved = await self._ensure_resolved(
                    address, family=sock.family, proto=sock.proto, loop=self)
                _, _, _, _, address = resolved[0]

            fut = self.create_future()
            self._sock_connect(fut, sock, address)
            return await fut

        def _sock_connect(self, fut, sock, address):
            fd = sock.fileno()
            try:
                sock.connect(address)
            except (BlockingIOError, InterruptedError):
                # Issue #23618: When the C function connect() fails with EINTR, the
                # connection runs in background. We have to wait until the socket
                # becomes writable to be notified when the connection succeed or
                # fails.
                self._ensure_fd_no_transport(fd)
                handle = self._add_writer(
                    fd, self._sock_connect_cb, fut, sock, address)
                fut.add_done_callback(
                    functools.partial(self._sock_write_done, fd, handle=handle))
            except (SystemExit, KeyboardInterrupt):
                raise
            except BaseException as exc:
                fut.set_exception(exc)
            else:
                fut.set_result(None)
    ```

4. 建立(传输, 协议)关系`self._create_connection_transport()`. 指定ssl时`self._make_ssl_transport()`, 不指定时`self._make_socket_transport()`.

   - **BaseSelectorEventLoop实现**, `_SelectorSocketTransport`传输以及`sslproto.SSLProtocol`


- 注1: getaddrinfo调用返回`(family, type, proto, canonname, sockaddr)`
- 注2: socket创建时根据目标解析地址的family, type, proto创建
- 注3: socket建立后设置非阻塞模式`sock.setblocking(False)`; 执行连接远端的操作`sock.connect()`, 连接中可能存在中断, 封装为future





# Futures

Futures是链接底层回调式(low-level callback-based)代码和上层异步/等待(high-level async/await)代码。

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

## asyncio.Future说明

> PEP 3156

在asyncio中的`Future`类型有意与`concurrent.futures.Future`(PEP 3148)类型尽量相似, 但两者有所不同, Future接口方法:

- `cancel()`当Future已经结束/取消时, 立即返回False; 此外, 尝试(Task可能对取消有额外处理)取消Future并返回True。
- `cancelled()`
- `done()`
- `result()`1)返回set_result()结果; 2)引起set_exception()异常; 3)被取消时,引起CaneclledError; 4)尚未结束时,引起其它异常.
- `exception()`1)set_result()时,返回None; 2)返回set_exception()异常; 3)被取消时,引起CaneclledError; 4)尚未结束时,引起其它异常.
- `add_done_callback(fn)`
- `remove_done_callback(fn)`
- `set_result(result)`
- `set_exception(exception)`

注1: 回调`fn`将以Future对象为参数进行回调, 即`fn(self)`
注2: `CancelledError`, `TimeoutError`, `InvalidStateError`曾经都使用了`concurrent.futures`模块下的定义, 后来在asyncio中重新定义
- `CancelledError(BaseException)`: 继承自**BaseException**
- `TimeoutError(Exception)`
- `InvalidStateError(Exception)`  



## Future类型

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



## ensure_future()方法 

> **注**: 3.10为非Future-like对象且无loop的情况标记为移除

`ensure_future(obj, *, loop=None)`

1. obj是Future, Task或者Future-like对象(通过`isfuture()`判断)
   - `return obj`
2. obj是协程coroutine(通过`iscoroutine()`判断), 则封装为Task对象
   - `return loop.create_task(obj)`
3. obj是可等待对象(通过`inspect.isawaitalbe()`判断), 则封装为等待obj的Task对象
   - `return ensure_future(_wrap_awaitable(obj), loop=loop)`


### `isfuture()`判断方法

> asyncio.futures(asyncio.base_fuatures)模块方法

obj是Future, 或者以duck-type方式声明(advertising)自身可用于Future(以特殊属性`_asyncio_future_blocking`)

```python
def isfuture(obj):
    return (hasattr(obj.__class__, '_asyncio_future_blocking') and
            obj._asyncio_future_blocking is not None)
```


### `iscoroutine()`判断方法

> asyncio.coroutines模块方法

1. 是`types.CoroutineType`的实例
2. 是`types.GeneratorType`, `collections.abc.Coroutine`的实例
3. 是asyncio.coroutines定义类型`CoroWrapper`的实例
4. 判断通过后的缓存类型

在inspect模块中, `iscoroutine()`指符合**条件1**的实例

```python
_COROUTINE_TYPES = (types.CoroutineType, types.GeneratorType,
                    collections.abc.Coroutine, CoroWrapper)
_iscoroutine_typecache = set()

def iscoroutine(obj):
    if type(obj) in _iscoroutine_typecache:
        return True

    if isinstance(obj, _COROUTINE_TYPES):
        if len(_iscoroutine_typecache) < 100:
            _iscoroutine_typecache.add(type(obj))
        return True
    else:
        return False
```

### `isawaitable()`判断方法

> inspect模块方法

1. 是`types.CoroutineType`实例
2. 是`types.GeneratorType`且*object.gi_code.co_flags*符合特殊标识
3. 是`collections.abc.Awaitable`实例

```python
def isawaitable(object):
    return (isinstance(object, types.CoroutineType) or
            isinstance(object, types.GeneratorType) and
                bool(object.gi_code.co_flags & CO_ITERABLE_COROUTINE) or
            isinstance(object, collections.abc.Awaitable))
```

### `_wrap_awaitable()`封装方法

- 为func添加**`_is_coroutine`**属性
- `types.coroutine()`将生成器函数转换为协程函数。该函数仍是是生成器(generator iteratior), 又是协程(coroutine)对象, 也是可等待对象(awaitable)

``` python
@types.coroutine
def _wrap_awaitable(awaitable):
    return (yield from awaitable.__await__())

_wrap_awaitable._is_coroutine = _is_coroutine
```



# Tasks

> asyncio.tasks.Task

## Task类型定义

`Task`继承于futures定义的`Futures`类型(_PyFuture = Future)

`Task`处于两种可能的状态之一: 1) 没有等待的`_fut_waiter`, 则计划调用`_step()`, 2) 正在等待`_fut_waiter`, 则不计划调用`_step()`


```python
_all_tasks = weakref.WeakSet()
_current_tasks = {}


def _register_task(task):
    _all_tasks.add(task)

def _unregister_task(task):
    _all_tasks.discard(task)

def _enter_task(loop, task):
    current_task = _current_tasks.get(loop)
    if current_task is not None:
        raise RuntimeError(f"Cannot enter into task {task!r} while another "
                           f"task {current_task!r} is being executed.")
    _current_tasks[loop] = task

def _leave_task(loop, task):
    current_task = _current_tasks.get(loop)
    if current_task is not task:
        raise RuntimeError(f"Leaving task {task!r} does not match "
                           f"the current task {current_task!r}.")
    del _current_tasks[loop]


class Task(futures._PyFuture)
    def __init__(self, coro, *, loop=None, name=None):
        super().__init__(loop=loop)
        ...
        if name is None:
            self._name = f'Task-{_task_name_counter()}'
        else:
            self._name = str(name)

        self._must_cancel = False
        self._fut_waiter = None
        self._coro = coro
        self._context = contextvars.copy_context()

        # 调用__step方法
        self._loop.call_soon(self.__step, context=self._context)
        # 注册记录该Task
        _register_task(self)

    def cancel(self, msg=None):
        if self.done():
            return False
        if self._fut_waiter is not None:
            if self._fut_waiter.cancel(msg=msg):
                return True
        
        self._must_cancel = True
        self._cancel_message = msg
        return True

    def __step(self, exc=None):
        if self.done():
            raise ...
        if self._must_cancel:
            if not isinstance(exc, exceptions.CancelledError):
                exc = self._make_cancelled_error()
            self._must_cancel = False
        coro = self._coro
        self._fut_waiter = None
        
        _enter_task(self._loop, self) # 状态管理
        try:
            # 启动协程生成器(coroutine generator)
            if exc is None:
                result = coro.send(None)
            else:
                result = coro.throw(exc)
        except StopIteration as exc:
            if self._must_cancel:
                # Task在刚好在结束前被取消
                self._must_cancel = False
                super().cancel(msg=self._cancel_message)
            else:
                super().set_result(exc.value)
        except exceptions.CancelledError as exc:
            self._cancelled_exc = exc
            super().cancel()  # I.e., Future.cancel(self).
        except (KeyboardInterrupt, SystemExit) as exc:
            super().set_exception(exc)
            raise
        except BaseException as exc:
            super().set_exception(exc)
        else:
            # coro 在启动后实际yield...
            blocking = getattr(result, '_asyncio_future_blocking', None)
            if blocking is not None:
                # coro 内部 yield from Future()???

                # Yielded Future must come from Future.__iter__().
                if futures._get_loop(result) is not self._loop:
                    # 判断loop的一致性
                    new_exc = RuntimeError(
                        f'Task {self!r} got Future '
                        f'{result!r} attached to a different loop')
                    self._loop.call_soon(
                        self.__step, new_exc, context=self._context)
                elif blocking:
                    # Future yield状态是blocking
                    if result is self:
                        new_exc = RuntimeError(
                            f'Task cannot await on itself: {self!r}')
                        self._loop.call_soon(
                            self.__step, new_exc, context=self._context)
                    else:
                        # 1) 修改Future yield的blocking状态为False
                        # 2) 等待Future完毕后调用该Task.__wakeup
                        # 3) 记录Task等待的Future对象
                        result._asyncio_future_blocking = False
                        result.add_done_callback(
                            self.__wakeup, context=self._context)
                        self._fut_waiter = result
                        if self._must_cancel:
                            if self._fut_waiter.cancel(
                                    msg=self._cancel_message):
                                self._must_cancel = False
                else:
                    new_exc = RuntimeError(
                        f'yield was used instead of yield from '
                        f'in task {self!r} with {result!r}')
                    self._loop.call_soon(
                        self.__step, new_exc, context=self._context)

            elif result is None:
                # Bare yield relinquishes control for one event loop iteration.
                self._loop.call_soon(self.__step, context=self._context)
            elif inspect.isgenerator(result):
                # Yielding a generator is just wrong.
                new_exc = RuntimeError(
                    f'yield was used instead of yield from for '
                    f'generator in task {self!r} with {result!r}')
                self._loop.call_soon(
                    self.__step, new_exc, context=self._context)
            else:
                # Yielding something else is an error.
                new_exc = RuntimeError(f'Task got bad yield: {result!r}')
                self._loop.call_soon(
                    self.__step, new_exc, context=self._context)
        finally:
            _leave_task(self._loop, self)
            self = None  # Needed to break cycles when an exception occurs.
```

## 场景: 如何等待Task呢?

示例代码

```python
import asyncio

async def main():
    await asyncio.sleep(1)

asyncio.run(main())
```

1. main()通过async def声明, 是一个协程. 在asyncio的启动中被封装为Task(Futures)
2. Task初始化后向loop添加Task的`__step`方法, 即启动协程(生成器), 直至`yield`,
3. 当Task运行生成器获得yield值是Future对象时, 向Future对象添加`__wakeup()`的回调

```python
async def sleep(delay, result=None, *, loop=None):
    ...
    if loop is None:
        loop = events.get_running_loop()
    else:
        ...
    future = loop.create_future()
    h = loop.call_later(delay,
                        futures._set_result_unless_cancelled,
                        future, result)
    try:
        return await future
    finally:
        h.cancel()
```

3. 协程启动后逐步向内调用(`await`=`yield from`), 进入sleep生成器, sleep再进入future生成器
4. 在`await future`前(在Future类实例yield中断执行前), sleep使用loop添加了**回调函数**, 在指定条件下为future设置结果
5. future设置结果的动作将唤醒上层Task, 恢复执行生成器执行逻辑, 即从future内部的`yield`处恢复运行至结束`StopIteration`




# 高级I/O复用Selectors

## select I/O操作系统模块

该模块是对select(), poll()的处理. Linux系统可用epoll(), BSD系统上可用kqueue().

1. `select.devpoll()`solaris系统
2. `select.epoll(sizehint=-1, flags=0)`支持Linux, 返回edge poll对象, 用于I/O事件边缘/水平触发
3. `select.poll()`返回poll对象, 支持注册与注销文件描述符, 对描述符轮询获取I/O事件
4. `select.kqueue()`仅BSD, 返回内核队列对象
5. `select.kevent()`仅BSD, 返回内核事件对象
6. `select.select(rlist, wlist, xlist[, timeout])`Unix系统的select的调用接口

### epoll边缘触发和水平触发的轮询

> https://linux.die.net/man/4/epoll

eventmask列表, 例如: EPOLLIN(可读), EPOLLOUT(可写)...

- `epoll.close()`
- `epoll.closed`
- `epoll.fileno()`返回文件描述符用于控制该epoll对象
- `epoll.fromfd(fd)`从给定fd创建epoll对象
- `epoll.register(fd[, eventmask])`向epoll注册一个fd
- `epoll.modify(fd, eventmask)`修改fd的mask
- `epoll.unregister(fd)`从epoll中删除fd
- `epoll.poll(timeout=None, maxevents=-1)`等待事件发生


### Kqueue, Kevent对象

> https://www.freebsd.org/cgi/man.cgi?query=kqueue&sektion=2

- `kqueue.close()`
- `kqueue.closed`
- `kqueue.fileno()`
- `kqueue.fromfd(fd)`
- `kqueue.control(changelist, max_events[, timeout]) -> eventlist`
  - `changelist`是可迭代对象, 迭代生成kevent, 否则为None

- `kevent.ident`标识值, 通常是文件描述符
- `kevent.filter`内核筛选器, 例如: `KQ_FILTER_READ`(获取描述符,有数据可读时返回)
- `kevent.flag`筛选器操作, 利润`KQ_EV_ADD`添加/修改事件, `KQ_EV_DELETE`从队列中删除事件
- `kevent.fflags`筛选特点标识
- `kevent.data`


## selectors描述

模块建立在select模块之上, 定义了`BaseSelector`抽象基类, 以及多个实现
- `SelectSelector`
- `PollSelector`
- `KqueueSelector`
- `EpollSelector`
- `DevpollSelector`

`DefaultSelector`是指向当前平台上最高效实现的别名

### 具体说明

位掩码`EVENT_READ`可读, `EVENT_WRITE`可写

类型定义: `SelectorKey`

- fileobj: 文件对象
- fd: 底层文件描述符
- events: 在此文件上被等待的时间
- data: 可选的关联到此对象上的不透明数据

类型定义: `BaseSelector`

- `register(fileobj, events, data=None) -> SelectorKey`注册一个文件对象, 在其上监视I/O事件
- `unregister(fileobj) -> SelectorKey`
- `modify(fileobj, events, data=None) -> SelectorKey`返回新的SelectorKey
- `select(timeout=None) -> List`等待注册的文件对象就绪
  - 返回(key, events)构成的列表, key对应SelectorKey实例, events是该文件对象等待的位掩码
- `close()`
- `get_key(fileobj)`返回关联的SelectorKey
- `get_map()`返回fileobj至SelectorKey的映射


### 示例

```python
import selectors
import socket

sel = selectors.DefaultSelector()

def accept(sock, mask):
    conn, addr = sock.accept()  # Should be ready
    print('accepted', conn, 'from', addr)
    conn.setblocking(False)
    sel.register(conn, selectors.EVENT_READ, read)

def read(conn, mask):
    data = conn.recv(1000)  # Should be ready
    if data:
        print('echoing', repr(data), 'to', conn)
        conn.send(data)  # Hope it won't block
    else:
        print('closing', conn)
        sel.unregister(conn)
        conn.close()

sock = socket.socket()
sock.bind(('localhost', 1234))
sock.listen(100)
sock.setblocking(False)
# 将callback函数accept预先存储
sel.register(sock, selectors.EVENT_READ, accept)

while True:
    events = sel.select()
    for key, mask in events:
        callback = key.data
        callback(key.fileobj, mask)
```




# Transports

> PEP 3156, 传输与协议内容接受了Twisted与PEP 3153的很多想法, 用户很少自己实现/初始化一个传输, 经常是利用事件循环建立传输

最常用的传输是双向流传输(bidirectional stream transport), 还有单向流传输(管道使用), 数据报传输. `get_extra_info(name, default=None)`适用于所有传输, 返回传输的具体实现信息, 相当于从传输中查找名为name的信息.

## Bidirectional Stream Transports

双向流式传输是对socket, 一对UNIX pipes, 或SSL/TLS链接的上层抽象.

具体方法:

- `write(data)`, 参数data是bytes对象
- `writelines(iterable)`, 这等价于`write(data) for data in iterable`
- `write_eof()`, 关闭当前链接的写端
- `can_write_eof()`, 如果底层传输协议支持write_eof()操作则返回True.
  - 例如对HTTP协议, 发送一个长度不确定的数据可以在结尾处发送eof; 如果传输协议(如SSL/TLS)不支持eof, 那么HTTP实现应使用 **chunked** 传输编码. 如果长度可以提前确定, 在HTTP头中标识长度更好.
- `get_write_buffer_size()`
- `set_write_buffer_limits(high=None, low=None)`为传输设置流传输的上下限, 与传输调用pause_writing()和resume_writing()方法相关
- `pause_reading()`, 在暂停读的期间, 传输不应该调用协议层的data_received()方法
- `resume_reading()`
- `close()`切断连接, 在切断连接前保持与对端的连接直到`write()`写入的缓存数据全被传输完毕
  - 当所有缓存的数据被flushed后, 应该调用协议的`connection_lost(None)`方法
- `abort()`切断连接, 已缓存的数据被丢弃, 然后调用协议的`connection_lost(None)`方法


## Unidirectional Stream Transports

单向写端传输支持`write()`, `writelines()`, `write_eof()`, `can_write_eof()`, `close()`, `abort()`方法. 

单向读端传输支持`pause_reading()`, `resume_reading()`和`close()`方法

在与协议方法交互方面
- 写端传输只能调用协议的`connection_made()`与`connection_lost()`方法
- 读端传输还可以调用`data_received()`与`eof_received()`方法


## Datagram Transports

实现传输方法
- `sendto(data, addr=None)`
- `abort()`
- `close()`


## Subprocess Transports

实现传输方法
- `get_pid()`
- `get_returncode()`
- `get_pipe_transport(fd)`根据参数fd(应该是0/1/2)返回对应的pipe传输, 应该使用该方法获取向子进程输出的端口(stdin)
- `send_signal(signal)`
- `terminate()`
- `kill()`
- `close()`terminate方法别名



# Protocols

> PEP 3156

协议总是与传输(transports)一同使用. 在asyncio中只提供少量(如HTTP协议的实现), 其它留给用户/三方实现

## Stream Protocols

流式协议(双向的)是最常用的协议. 该协议需要实现如下供传输使用的方法.

- `connection_made(transport)`. 表示传输已经就绪完全和对端建立了连接
  - **必须调用一次, 只能调用一次**
- `data_received(data)`. 传输从连接中读取了数据. data必须式非空的**bytes**对象.
- `eof_received()`. 当对端执行`write_eof()`等效的操作时. 返回false值(包括None)时, 传输将关闭; 返回true值时, 传输将保留由协议关闭. 
  - **最多调用一次**
  - 例外情况SSL/TLS连接将立即关闭
- `pause_writing()`. 使协议暂停写入数据
- `resume_writing()`. 使协议可以开始写入数据.
- `connection_lost(exc)`. 表示传输被关闭/终止/异常情况.
  - **必须调用一次, 只能调用一次**

## Datagram Protocols

数据报协议, `connection_made(transport)`与`connection_lost(exc)`与流式协议一致. 此外

- `datagram_received(data, addr)`. 表示数据报data(bytes对象)从远端addr接收到
- `error_received(exc)`. 表示发送/接收操作引起了`OSError`异常, 由于数据报协议的异常可能是暂时性的, 因此由上层协议决定关闭传输操作`close()`


## Subprocess Protocol

子进程协议, `connection_made(transport)`与`connection_lost(exc)`, `pause_writing()`, `resume_writing()`与流式协议一致. 此外

- `pipe_data_received(fd, data)`, 当子进程向标准输出/标准异常发送数据时, fd必须为1(stdout)/2(stderr), data是bytes对象
- `pipe_connetion_lost(fd, exc)`当子进程关闭stdin/stdout/stderr时
- `process_exited()`当子进程退出时, 协议通过传输的`get_returncode()`方法跟踪退出状态

