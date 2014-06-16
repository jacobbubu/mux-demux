# MuxDemux

在 _任何_ 文本流上多路复用其他对象流。

[![Build Status](https://travis-ci.org/dominictarr/mux-demux.png)]
  (https://travis-ci.org/dominictarr/mux-demux)
  
本 repo 不仅仅是对原 [repo](https://github.com/dominictarr/mux-demux) redeme 的翻译，也试图将协议的细节分析出来。

## 稳定程度

稳定: 期待补丁或者添加可能的新功能。

## 例子

``` js
var MuxDemux = require('mux-demux')
var net = require('net')

net.createServer(function (con) {
  con.pipe(MuxDemux(function (stream) {
    stream.on('data', console.log.bind(console))
  })).pipe(con)
}).listen(8642, function () {
  var con = net.connect(8642), mx
  con.pipe(mx = MuxDemux()).pipe(con)

  var ds = mx.createWriteStream('times')

  setInterval(function () {
    ds.write(new Date().toString())
  }, 1e3)
})
```

上例中，通过 `net.createServer...listen...` 创建了一个`tcp stream`, `con`。随后，创建了一个 `MuxDemux` stream，`mx`。`con.pipe(mx = MuxDemux()).pipe(con)`， 使得所有 tcp stream 上的往来消息都会被传递给 mx，反之亦然。

随后，通过 `var ds = mx.createWriteStream('times')`创建了一个命名为 `times` 的 stream 专门用于周期性的写入时间见戳（见 `setInterval` 部分）。

这样，每一个 tcp 的client都会每隔1秒收到一个 server 的时间戳。

通过为  `mx.createWriteStream(name)` 传入不同的名字，我们可以在 MuxDemux 流对象上创建多个不同目的的 stream，它们都利用底层的 `tcp stream`（`con`）来传输。

由此可以看到，承担不同职责、不同功能的对象，只要支持`node stream`规范，就可以通过 MuxDemux 复用底层的通信管道（当然也是 stream），使得 client/server 见可以创建丰富的使用场景。

例如，通过 MuxDemux 你可以把 [dnoe](https://github.com/substack/dnode) 和 [Scuttlebutt](https://github.com/dominictarr/scuttlebutt) 集成到你的 WebSocket 上，使得 client/server 即支持 rpc 风格的调用，也支持基于向量钟的状态同步。

## 二进制支持

用 msgpack 替代 JSON 编码方式，即可使用二进制协议。

只需要 将 require `mux-demux/msgpack` 改为 `mux-demux`。

``` js
var MuxDemux = requrie('mux-demux/msgpack')
```

## 要小心：

为每一个底层连接建立一个 `MuxDemux`。不要将多个连接连接到一个 `MuxDemux'。

### 正确的方式

``` js
net.createServer(function (stream) {
  stream.pipe(MuxDemux(function (_stream) { 

  }).pipe(stream)
}).listen(port)
```

### 错误!
``` js
var mx = MuxDemux()
net.createServer(function (stream) {
  //this will connect many streams to the OUTER MuxDemux Stream!
  stream.pipe(mx).pipe(stream)
}).listen(port)
```

每一个客户端的连接都会连到同一个 `MuxDemux`，这就乱了。

### 错误处理, 生产环境的用法

`mux-demux` 解析 `JSON` 协议，所以你必须处理任何错误。无效的数据可能来自于任何一个连接者。

``` js
net.createServer(function (stream) {
  var mx = MuxDemux()
  stream.pipe(mx).pipe(stream)
  mx.on('error', function () {
    stream.destroy()
  })
  stream.on('error', function () {
    mx.destroy()
  })
}).listen(9999)
```

当 mx 无法解析数据时，会 emit error，此时你是无法恢复的，通常的做法是关闭底层连接。反之，底层连接也可能出错，mx 也没有必要的知识去容忍这个错误。

#API

the API [browser-stream](http://github.com/dominictarr/browser-stream#api)

这个 repo 是把 socket.io 的client端包装为 stream 的方式。类似的库挺多的，有的会包装 socks，也有的包装 websocket-stream。它们都是通过 browserify 是的浏览器也可以包含 node.js 的核心库，尤其是 stream。

``` js

var MuxDemux = require('mux-demux')
var a = MuxDemux()
var b = MuxDemux()

a.pipe(b).pipe(a)

b.on('connection', function (stream) {
  // inspect stream.meta to decide what this stream is.
})

a.createWriteStream(meta)
a.createReadStream(meta)
a.createStream(meta)

```

如果 client 和 server 都是通过监听 `on('connection',...)` 来创建连接，那么本质上，client 和 server 的代码没什么区别。随后 client/server 每一方都可以通过 `create{Write,Read,}Stream(meta)` 来创建新的 stream。

### MuxDemux(options, onConnection)

创建 MuxDemux stream. 最为可选，可以传入一个 options hash 

    {
        error: Boolean,
        wrapper: function (stream) {...}
    }

如果 error option 是 true，那么 MuxDemux 遇到非预期的错误时将在 stream 上发出 error 事件，否则，MuxDemux 将直接在 stream 上发出 'end' 事件。

`wrapper` 被用来改变缺省的序列化方式。缺省方式是行分割的JSON字符串（JSON消息之间以换行符分割），参见[下面](#wrapper_examples)的例子。mux-demux 的两端必须使用相同的 `wrapper`。

`options` 是可选的。`MuxDemux(onConnection)` 是 `MuxDemux().on('connection', onConnection)` 的快捷方式。

### createReadStream (meta)

为连接的另一端打开一个 `ReadableStream`。
该 API 返回一个 `ReadableStream`。
连接的另一端也将生成一个 `WritableStream`和这个 `ReadableStream` 对接。

### createWriteStream (meta)

为连接的另一端打开一个`WritableStream`。
该API返回一个 `WritableStream`。
连接的另一端也会生成一个 `ReadableStream` 和这个 `WritableStream` 对接。

### createStream (meta, opts)

为连接的另一段创建一个 readable and writable stream。
API 返回一个 `Stream`，连接的另一端也会生成一个 `Stream` 于此对接。

opts 的值可能为 `{allowHalfOpen: true}`；如果没有这样设置，当调用 stream 的 `end()` 方法时，stream 将发出 `'end'` 事件，这可能造成 stream 丢失对端发来的部分数据。如果 `allowHalfOpen` 设置为 `true`，则远端必须自己调用`end()`。

> 注意，当引用类 (`Stream`) 时，必须首字母大写，并且用``括起来。
> 当说明是一个 `Stream` 的实例的时候，则用小写，并且不使用 `` 括起来，除非是用来指代例子中的特定变量。

### close(cb)

调用 `close(cb)`，将使得 mux-demux 在所有 sub-streams 关闭后发出 end 事件，并且调用毁掉函数。这个方法会等待所有 sub-streams 结束或者出错，其行为类似[`net.Server#close`](http://nodejs.org/api/net.html#net_server_close_cb)。

也就是说，close 方法并不会帮你关闭所有 sub-streams。

callback 是可选的。

### Wrapper Examples

A stream of plain old js objects.

``` js
new MuxDemux({wrapper: function (stream) { return stream } })
```

A stream of msgpack.

``` js
var es = require('event-stream')
var ms = require('msgpack-stream')

new MuxDemux({wrapper: function (stream) { 
  return es.pipeline(ms.createDecodeStream(), stream, ms.createEncodeStream()) 
}})

```

### MuxDemuxStream#error

这里有一个附加的方法。调用 `stream.error(err)` 将在对端的 stream 上触发一个 error 事件。这可能用于 server 向 client 发送类似 404-Not Found 之类的错误消息。