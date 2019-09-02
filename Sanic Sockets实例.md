## Sanic Sockets实例

- Sanic 可以使用 Python 的 socket 模块来容纳非 IPv4 的 sockets。比如下面的 IPv6 的例子：

  ```python
  from sanic import Sanic
  from sanic.response import json
  import socket
  
  sock = socket.socket(socket.AF_INET6, socket.SOCK_STREAM)
  sock.bind(('::', 7777))
  
  app = Sanic()
  
  @app.route("/")
  async def test(request):
      return json({"hello": "world"})
  
  if __name__ == "__main__":
      app.run(sock=sock)
  ```

  可以通过 `curl` 来测试这个 IPv6 的应用：



#### 设置调试模式

- 通过设置`debug=True`来启用调试模式：

  ```python
  from sanic import Sanic
  from sanic.response import json
  
  app = Sanic()
  
  @app.route('/')
  async def hello_world(request):
      return json({"hello": "world"})
  
  if __name__ == '__main__':
      app.run(host="0.0.0.0", port=8000, debug=True)
  ```

  

#### 手动设置自动重新加载

- 可以通过`auto_reload`设置来启动自动重新加载模式：

  ```python
  from sanic import Sanic
  from sanic.response import json
  
  app = Sanic()
  
  @app.route('/')
  async def hello_world(request):
      return json({"hello": "world"})
  
  if __name__ == '__main__':
      app.run(host="0.0.0.0", port=8000, auto_reload=True)
  ```

  

#### Sanic 测试

Sanic 路由节点测试可以通过`test_client`对象进行，它依赖于`aiohttp`库。

##### test_client测试

- 这个`test_client`对象提供了`get`, `post`, `put`, `delete`, `patch`, `head`和`options` 方法来测试我们的web 应用。

  ```python
  # Import the Sanic app, usually created with Sanic(__name__)
  from external_server import app
  
  def test_index_returns_200():
      request, response = app.test_client.get('/')
      assert response.status == 200
  
  def test_index_put_not_allowed():
      request, response = app.test_client.put('/')
      assert response.status == 405
  ```

  





