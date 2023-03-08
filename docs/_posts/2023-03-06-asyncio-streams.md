---
title: Python异步（三） asyncio 流式I/O操作实现
---

> 参考 [Python文档-asyncio-流](https://docs.python.org/zh-cn/3.10/library/asyncio-stream.html#streamwriter)


## `asyncio`的Transport与Protocol

### 1. 传输协议`asyncio.Protocol`

`asyncio`中对于Protocol对象的定义

```python
class BaseProtocol:
    """Common base class for protocol interfaces."""

    __slots__ = ()

    def connection_made(self, transport):
        """当连接已建立时被调用, 参数transport表示一个双向传输管道"""

    def connection_lost(self, exc):
        """当连接丢失/被关闭时被调用, 参数exc表示原因"""

    def pause_writing(self):
        """当transport缓存数据(待发送)过多时被调用."""

    def resume_writing(self):
        """当transport缓存基本消耗完成时被调用"""


class Protocol(BaseProtocol):
    """Interface for stream protocol.

    State machine of calls:

      start -> CM [-> DR*] [-> ER?] -> CL -> end

    * CM: connection_made()
    * DR: data_received()
    * ER: eof_received()
    * CL: connection_lost()
    """

    __slots__ = ()

    def data_received(self, data):
        """当收到数据时被调用, 参数data时bytes对象"""

    def eof_received(self):
        """当对端调用write_eof()的等效操作时被调用. 
        transport将根据该方法返回值选择直接关闭, 或等待protocol关闭"""

```


### 2. 传输端点`asyncio.Transport`

```python
class BaseTransport:
    """Base class for transports."""
    __slots__ = ('_extra',)

    def __init__(self, extra=None):
        self._extra = extra or {}

    def get_extra_info(self, name, default=None):
        return self._extra.get(name, default)

    def is_closing(self):
        """表示已关闭 / 正在关闭"""

    def close(self):
        """关闭transport. 以缓存的data将继续被传输, 当所有缓存data发送完毕后,
        protocol的connection_lost()方法被调用."""

    def set_protocol(self, protocol):
        """设置protocol"""

    def get_protocol(self):
        """当前的protocol"""


class ReadTransport(BaseTransport):
    """Interface for read-only transports."""
    __slots__ = ()

    def is_reading(self):
        """表示transport处于receiving."""

    def pause_reading(self):
        """暂停接收, 使protocol的data_received()暂时不会被调用"""

    def resume_reading(self):
        """恢复接收."""


class WriteTransport(BaseTransport):
    """Interface for write-only transports."""
    __slots__ = ()

    def set_write_buffer_limits(self, high=None, low=None):
        """设置流传输控制的缓存上下限.
        - 当缓存超过high, 调用protocol的pause_writing(). 
        - 当缓存降至low, 调用protocoal的resume_writing()."""

    def get_write_buffer_size(self):
        """写缓存区大小."""
        raise NotImplementedError

    def get_write_buffer_limits(self):
        """写缓存区的(low, high)."""
        raise NotImplementedError

    def write(self, data):
        """向transport写一些数据, 非阻塞, 数据会安排异步发送."""

    def writelines(self, list_of_data):
        data = b''.join(list_of_data)
        self.write(data)

    def write_eof(self):
        """先flush所有已缓存写数据, 然后关闭写端."""

    def can_write_eof(self):
        """表示该transport是否支持write_eof()方法"""

    def abort(self):
        """立即关闭transport, 缓存的数据会被丢弃, 不接收新数据.
        确保protocol的connection_lost()方法会被调用."""
        
        
class Transport(ReadTransport, WriteTransport):
    __slots__ = ()
```



## asyncio的流控制对象

### 1. `StreamReader`, `StreamWriter`流方法接口

> [asyncio.StreamReader](https://docs.python.org/zh-cn/3.10/library/asyncio-stream.html#streamreader)
>
> 这个类表示一个读取器对象，该对象提供api以便于从IO流中读取数据。推荐使用`open_connection()`和`start_server()`获取`StreamReader`实例

说明

- `IncompleteReadError`标识没有读到预期的数据, `IncompleteReadError.partial`属性包括了流结束前的bytes字符串
- `LimitOverrunError`是`readuntil()`方法读入数据超过容量限制引起

```python
class StreamReader:
    async def read(self, n=-1):
        """至多读n个byte, n=-1表示读至EOF"""
      
    async def readline(self):
        """读至'\n'结尾的bytes"""
      
    async def readexactly(self, n):
        """精确读n个bytes, 若提前读到EOF, 引发IncompleteReadError"""
      
    async def readuntil(self, separator=b'\n'):
        """从流读至separator字符, 若提前读至EOF, 引发IncompleteReadError"""
        
    def at_eof(self):
        """当缓冲为空且feed_eof()被调用, 则为True"""
        
    """inner methods"""
    def exception(self):
    
    def set_exception(self, exc):
        """当传输异常终止时被调用. 是Protocol的connection_lost(exc)的响应处理"""
      
    def set_transport(self, transport):
      
    def feed_eof(self):
        """
        当传输正常终止时被调用. 是Protocol的connection_lost(exc=None)的响应处理
        对Protocol的eof_received()时的响应处理
        """
    
    def feed_data(self, data):
        """对Protocol的data_received()的响应处理"""
```


> [asyncio.StreamWriter](https://docs.python.org/zh-cn/3.10/library/asyncio-stream.html#streamwriter)
>
> 这个类表示一个写入器对象，该对象提供api以便于写数据至IO流中。推荐使用`open_connection()`和`start_server()`获取`StreamWriter`实例

说明:
- `drain()`方法用于自定义流控制

```python
class StreamWriter:
    def write(data):
      
    def writelines(data):
      
    def close():
        # 应当随后调用wait_closed()

    def can_write_eof():
      
    def write_eof():
      
    async def drain():
        """用于流控制"""
      
    def is_closing():
      
    async def wait_closed():
    
```

`open_connection()`, `start_server()`是对EventLoop建立连接方法`create_connnection()`,`create_server()`的封装, 返回的是`StreamReader`,`StreamWriter`的流操作对象

```python
async def open_connection(host=None, port=None, *,
                          loop=None, limit=_DEFAULT_LIMIT, **kwds):
    """对EventLoop的create_connection()方法的封装, 获取(reader, writer)."""
    if loop is None:
        loop = events.get_event_loop()
        
    reader = StreamReader(limit=limit, loop=loop)

    protocol = StreamReaderProtocol(reader, loop=loop)
    transport, _ = await loop.create_connection(
        lambda: protocol, host, port, **kwds)
    
    writer = StreamWriter(transport, protocol, reader, loop)
    return reader, writer
    
    
async def start_server(client_connected_cb, host=None, port=None, *,
                       loop=None, limit=_DEFAULT_LIMIT, **kwds):
    if loop is None:
        loop = events.get_event_loop()

    def factory():
        reader = StreamReader(limit=limit, loop=loop)
        protocol = StreamReaderProtocol(reader, client_connected_cb,
                                        loop=loop)
        return protocol

    return await loop.create_server(factory, host, port, **kwds)
```

### 2. 与流Reader/Writer相关的传输协议

`FlowControlMixin`部分实现了`asyncio.Protocol`接口中与流控制相关的功能

- `connection_lost()`
- `pause_writing()`
- `resume_writing()`

未实现:

- `connection_made()`
- `data_received()`
- `eof_received()`

```python
class FlowControlMixin(protocols.Protocol):
    def __init__(self, loop=None):
          self._loop = loop or events.get_event_loop()
        """标识是否暂停"""
        self._paused = False
        self._drain_waiter = None
        """表示连接丢失"""
        self._connection_lost = False
    
    def pause_writing(self):
        assert not self._paused
        self._paused = True
    
    def resume_writing(self):
          """完成self._drain_waiter的等待"""
        assert self._paused
        self._paused = False
        
        waiter = self._drain_waiter
        if waiter is not None:
            self._drain_waiter = None
            if not waiter.done():
                waiter.set_result(None)
                
    def connection_lost(self, exc):
        """因为丢失连接, 与resume_writing相同完成self._drain_waiter等待"""
        self._connection_lost = True
        # Wake up the writer if currently paused.
        if not self._paused:
            return
        waiter = self._drain_waiter
        if waiter is None:
            return
        self._drain_waiter = None
        if waiter.done():
            return
        if exc is None:
            waiter.set_result(None)
        else:
            waiter.set_exception(exc)
            
    async def _drain_helper(self):
        """等待self._drain_waiter
        - 连接已关闭, 不能等待, 引发异常
        - 非暂停状态, 不用等待, 直接返回
        - 非关闭+暂停状态, await self._drain_waiter
        """
        if self._connection_lost:
            raise ConnectionResetError('Connection lost')
        if not self._paused:
            return
        waiter = self._drain_waiter
        assert waiter is None or waiter.cancelled()
        waiter = self._loop.create_future()
        self._drain_waiter = waiter
        await waiter

    def _get_close_waiter(self, stream):
        raise NotImplementedError
```



`StreamReaderProtocol`是对`asyncio.Protocol`的协议实现. 因为流操作对象`StreamReader`/`StreamWriter`只是传输协议的一部分, 所以通过`StreamReaderProtocol`进行适配转换

主要属性:

- `_reject_connection=False` 虽然用于connection_made()方法, 但值并无变化

- `_over_ssl=False` 若Transport设置了`sslcontext`属性则为`True`
- `_closed=loop.create_future()`, 
  - 该协议实例化时同步建立, 
  - 在 connection_lost() 设置结果 `set_result()` or `set_exception()`
  - 回收`__del__`时, 如果非正常关闭(`closed.done() and not closed.cancelled()`)，调用该Future的exception()方法



在`create_server()`时使用

- `_stream_writer` 在服务模式的 connection_made() 中建立`StreamWriter(transport, ...)`
- `_client_connected_cb` 在服务模式的 connection_mader() 中，为建立的`StreamWriter`调用



```python
class StreamReaderProtocol(FlowControlMixin, protocols.Protocol):
    """Helper class to adapt between Protocol and StreamReader."""

    def __init__(self, stream_reader, client_connected_cb=None, loop=None):
        super().__init__(loop=loop)
        if stream_reader is not None:
            self._stream_reader_wr = weakref.ref(stream_reader)
        else:
            self._stream_reader_wr = None
        if client_connected_cb is not None:
            self._strong_reader = stream_reader
        self._reject_connection = False
        self._stream_writer = None
        self._transport = None
        self._client_connected_cb = client_connected_cb
        self._over_ssl = False
        self._closed = self._loop.create_future()

    @property
    def _stream_reader(self):
        if self._stream_reader_wr is None:
            return None
        return self._stream_reader_wr()

    def connection_made(self, transport):
        if self._reject_connection:
            ...         # handle_exception, tranport.abort()
            return
          
        self._transport = transport
        reader = self._stream_reader
        if reader is not None:
            reader.set_transport(transport)
            
        self._over_ssl = transport.get_extra_info('sslcontext') is not None
                
        # 仅对于create_server()模式建立_stream_writer, 执行回调
        if self._client_connected_cb is not None:
              ...

    def connection_lost(self, exc):
        # 为StreamReader调用 feed_eof() / set_exception() 表示传输终止
        reader = self._stream_reader
        if reader is not None:
            if exc is None:
                reader.feed_eof()
            else:
                reader.set_exception(exc)

                if not self._closed.done():
            if exc is None:
                self._closed.set_result(None)
            else:
                self._closed.set_exception(exc)
        # FlowControlMixin 的流关闭, 确保缓冲数据正确传输完毕
        super().connection_lost(exc)
        self._stream_reader_wr = None
        self._stream_writer = None
        self._transport = None

    def data_received(self, data):
        reader = self._stream_reader
        if reader is not None:
            reader.feed_data(data)

    def eof_received(self):
        reader = self._stream_reader
        if reader is not None:
            reader.feed_eof()
        if self._over_ssl:
            # 返回False, 表示对于ssl连接, Transport 自行关闭连接
            return False
        # 返回True, 表示对于ssl连接, Transport由上层关闭连接
        return True

    def _get_close_waiter(self, stream):
        return self._closed

    def __del__(self):
        # Prevent reports about unhandled exceptions.
        # Better than self._closed._log_traceback = False hack
        closed = self._closed
        if closed.done() and not closed.cancelled():
            closed.exception()

```



### 3. `StreamReader`代码研究

#### 1）与Transport交互

- Protocol `connection_made(transport)`
  - StreamReader `set_transport(transport)`
- Protocol `connection_lost(exc)`
  - StreamReader `set_exception(exc)`
  - StreamReader `feed_eof()`
- Protocol `pause_writing()`/`resume_writing()`
  - FlowControlMixin
- Protocol `data_received(data)`
  - StreamReader `feed_data(data)`
- Protocol `eof_received()`
  - StreamReader `feed_eof()`

```python
class StreamReader:
    def set_transport(self, transport):
        """记录transport, 仅一次"""
    
    def set_exception(self, exc):
        """记录exception, 唤醒waiter"""
        self._exception = exc

        waiter = self._waiter
        if waiter is not None:
            self._waiter = None
            if not waiter.cancelled():
                waiter.set_exception(exc)
        
    def feed_eof(self):
        """记录eof标识位, 唤醒waiter"""
        self._eof = True
        self._wakeup_waiter()
      
    def feed_data(self, data: bytes):
        """收到eof标识位后不再继续接收"""
        assert not self._eof, 'feed_data after feed_eof'

        if not data:
            return
                
        """向buffer增加数据, 唤醒waiter"""
        self._buffer.extend(data)
        self._wakeup_waiter()
                
        """asyncio.ReadTransport读入控制条件"""
        if (self._transport is not None and 
                    not self._paused and 
                    len(self._buffer) > 2 * self._limit):
            try:
                self._transport.pause_reading()
            except NotImplementedError:
                """屏蔽transport, 不再尝试暂停..."""
                self._transport = None
            else:
                self._paused = True
                
    def at_eof(self):
        """收到eof标识位, 同时没有缓存数据"""
        return self._eof and not self._buffer

```

`self._waiter`逻辑, 初始化为`None`

- `_wait_for_data(func_name)`, 用于4个读接口方法
  - 只能在`_waiter=None`时等待数据, 
  - 新建`self._waiter = self._loop.create_future()`
  - 持续等待`await self._waiter`
- `set_exception(exc)`, 异常状态唤醒*waiter*, `waiter.set_exception(exc)`
- `_wakeup_waiter()`, 读取数据后唤醒`waiter.set_result(None)`
  - `feed_eof()`
  - `feed_data(data)`

#### 2) 与上层应用交互

```python
class StreamReader:
    async def readline(self):
        """对readunitl('\n')的封装
        1) 部分读可能 -> 返回部分读到的数据
        2) 超缓存容量时 -> 清除+丢弃缓存部分, _maybe_resume_transport()
        """
      
    async def readuntil(self, separator=b'\n'):
        """ 读至separator, 正常情况下data与separator会从缓冲区buffer中移除
        
        检查 len(separator)>0
        检查 若有self._exception, 则向上引发
        可能引发IncompleteReadError, LimitOverrunError, 超限异常时数据仍在buffer中
        """
      
    async def read(self, n=-1):  
        """读入n个bytes"""
      
    async def readexactly(self, n):
        """
        1) EOF, len(buffer)<n -> 清除buffer, IncompleteReadError
        2) len(buffer) < n -> 继续等待数据
        3) len(buffer) ==n -> data=bytes(buffer)复制, buffer.clear()
        4) len(buffer) > n -> data=bytes(buffer[:n]), del buffer[:n]
        
        检查读入transport状态. _maybe_resume_transport
        """
      
```



### 4. `StreamWriter`代码研究

写端点`WriteTransport`接口方法

- `abort()`
- `get_write_buffer_size()`
- `get_write_buffer_limits()`
- `set_write_buffer_limits(high, low)`


- `write(data)` <- StreamWriter `write(data)`
- `writelines(list_of_data)` <- StreamWriter `writelines()`
- `can_write_eof()` <- StreamWriter `can_write_eof()`
- `write_eof()` <- StreamWriter `write_eof()`
- `close()` <- StreamWriter `close()`
- `is_closing()` <- StreamWriter `is_closing()`

```python
class StreamWriter:
    async def wait_closed(self):
        """StreamReaderPotocol在响应connection_lost()时, 将Future类型的_closed变量设置为完成"""
        await self._protocol._get_close_waiter(self)
        
    async def drain(self):
        """reader的异常在writer的方法中引发？
        因为StreamReaderProtocol仅在connection_lost()时可能设置异常状态"""
        if self._reader is not None:
            exc = self._reader.exception()
            if exc is not None:
                raise exc
                
        if self._transport.is_closing():
            await sleep(0)
        """FlowControlMixin控制, 在resume_writing()和connection_lost()情况下完成等待"""
        await self._protocol._drain_helper()
```

