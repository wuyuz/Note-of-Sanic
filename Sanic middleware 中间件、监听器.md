## Sanic middleware 中间件、监听器

​	中间件是在服务器接受请求之前或之后执行的函数。它们用于修改传递给路由处理函数的`request`，或是由处理函数生成的`response`对象。



#### 中间件类型

​	中间件有两种类型：**request**和**response**，都是通过`@app.middleware`修饰器来声明的，以修饰器的字符串参数`request`或`response`来表示这两种类型。

- 请求中间件只接受`request`对象作为参数。

- 响应中间件同时接受`request`和`response`两个对象作为参数

  下面是一个最简单的中间件的例子，它没有改变request和response，只是打印了信息：

  ```python
  @app.middleware('request')
  async def print_on_request(request):
      print("I print when a request is received by the server")
  
  @app.middleware('response')
  async def print_on_response(request, response):
      print("I print when a response is returned by the server")
  ```

  

#### 修改request或response

- 中间件可以修改作为参数传递的request或response，但不需要返回它们，参见下面的例子：

  ```python
  app = Sanic(__name__)
  
  @app.middleware('request')
  async def add_key(request):
      # Add a key to request object like dict object
      request['foo'] = 'bar'
  
  @app.middleware('response')
  async def custom_banner(request, response):
      response.headers["Server"] = "Fake-Server"
  
  @app.middleware('response')
  async def prevent_xss(request, response):
      response.headers["x-xss-protection"] = "1; mode=block"
  
  @app.route('/')
  async def home(request):
      return response.text(request['foo'])
  ```

  ​       上面的代码将按顺序应用3个中间件。第一个中间件**add_key**给`request`对象增加了一个新的键`foo`，这样可以工作是因为`request`对象可以像字典那样被操作。

  第二个中间件**custom_banner**修改了HTTP响应的头，把`Server`设置成`Fake-Server`。最后一个中间件**prevent_xss**添加了响应头以防止跨站点脚本（XSS）攻击。response类型的中间件在路由处理函数（比如，本例中的`home()`返回`response`后被调用。

  ```
  http://127.0.0.1:8888
  
  #结果：
  HTTP/1.1 200 OK
  Connection: keep-alive
  Keep-Alive: 5
  x-xss-protection: 1; mode=block
  Server: Fake-Server
  Content-Length: 3
  Content-Type: text/plain; charset=utf-8
  
  bar
  ```

- 注意：提前响应

  ​	这里的“提前”是指中间件直接返回`HTTPResponse`对象，这时请求将停止处理并返回response。如果这发生在request类型的中间件，路由处理函数将不会被调用。返回response将阻止后续的中间件继续执行。

  ```python
  @app.middleware('request')
  async def halt_request(request):
      return text('I halted the request')
  
  @app.middleware('response')
  async def halt_response(request, response):
      return text('I halted the response')
      
  #因为中间件halt_request返回了Response对象，其后续的中间件halt_response就不会被执行。
  ```

  

#### 监听器

​      Sanic提供的监听器（listener）允许我们在应用程序生命周期内的多个时间点运行一些代码。类似与Flask、Django的信号

##### 监听器分类：

​	如果你想在Server开始时执行一些初始化代码，或者是在Server结束时执行一些清除代码，你就可以使用下面这些监听器：（和中间件一样，监听器的类型也是通过字符串参数来分类的）

- before_server_start

- after_server_start

- before_server_stop

- after_server_stop

  这些中间件通过修饰器`@app.listener`修饰接受`app`和`loop`参数的函数来实现。比如：

  ```python
  @app.listener('before_server_start')
  async def setup_db(app, loop):
      app.db = await db_setup()
  
  @app.listener('after_server_start')
  async def notify_server_started(app, loop):
      print('Server successfully started!')
  
  @app.listener('before_server_stop')
  async def notify_server_stopping(app, loop):
      print('Server shutting down!')
  
  @app.listener('after_server_stop')
  async def close_db(app, loop):
      await app.db.close()
  ```

  

##### 监听器注册方法：register_listener

​	除了使用修饰器还可以通过`register_listener`方法来注册监听器。这个方法的用处是，方便你在其它模块定义监视器函数，并在app所在文件进行注册。

```python
# file: my_listener.py
async def setup_db(app, loop):
    app.db = await db_setup()
    
    
# file: app.py
from my_listener import setup_db
app = Sanic()
app.register_listener(setup_db, 'before_server_start')
```



##### Sanic `add_task`方法

如果你想安排一个后台任务在事件循环开始后执行，Sanic提供了`add_task`方法来帮你轻松实现。

```
async def notify_server_started_after_five_seconds():
    await asyncio.sleep(5)
    print('Server successfully started!')

app.add_task(notify_server_started_after_five_seconds())
```

Sanic将试图自动注入`app`对象，可以作为一个参数传递给任务函数：

```
async def notify_server_started_after_five_seconds(app):
    await asyncio.sleep(5)
    print('Server successfully started!')

app.add_task(notify_server_started_after_five_seconds)
```

或者可以显式地传递`app`可以起到同样的效果：

```
async def notify_server_started_after_five_seconds(app):
    await asyncio.sleep(5)
    print('Server successfully started!')

app.add_task(notify_server_started_after_five_seconds(app))
```



#### Sanic WebSocket使用

​	WebSocket使得客户端和服务器之间的数据交换变得更加简单，允许服务端主动向客户端推送数据。Sanic 提供了非常简洁的 websockets 抽象，让我们开发基于 WebSocket 的web应用非常容易。

##### WebSocket 实例

- 下面是一个简单的建立 WebSocket 的例子：

  ```python
  from sanic import Sanic
  from sanic.response import json
  from sanic.websocket import WebSocketProtocol
  
  app = Sanic()
  
  @app.websocket('/feed')
  async def feed(request, ws):
      while True:
          data = 'hello!'
          print('Sending: ' + data)
          await ws.send(data)
          data = await ws.recv()
          print('Received: ' + data)
  
  if __name__ == "__main__":
      app.run(host="0.0.0.0", port=8888, protocol=WebSocketProtocol)
  ```

  上面这个`app`建立一个 WebSocket 的服务。也可以用 `app.add_websocket_route` 方法替换装饰器：

  ```python
  async def feed(request, ws):
      pass
  
  app.add_websocket_route(feed, '/feed')
  ```

  ​	WebSocket 路由的处理函数需要两个参数，第一个是`request`对象，第二个是 WebSocket 协议对象，这个协议对象有两个方法`send`和`recv`用来发送和接收数据。

##### WebSocket 的配置

通过`app.config`来配置 WebSocket，比如：

```
app.config.WEBSOCKET_MAX_SIZE = 2 ** 20
app.config.WEBSOCKET_MAX_QUEUE = 32
app.config.WEBSOCKET_READ_LIMIT = 2 ** 16
app.config.WEBSOCKET_WRITE_LIMIT = 2 ** 16
```