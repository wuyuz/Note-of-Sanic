## Sanic基于类的视图（CBV）



​	基于类的视图只是实现对请求的响应行为的类。它们提供了一种在同一路由节点处分别处理不同HTTP请求类型的方法。不是为每个路由节点支持的请求类型定义和装饰三个不同的处理函数，而是为路由节点分配基于类的视图。这非常类似于Tornado的类视图。



#### 定义视图

​	基于类的视图应该继承自 `HTTPMethodView`。然后就可以为每个 HTTP 请求类型实现类方法。如果收到的请求没有对应的方法，就会产生一个`405: Method not allowed`的响应。为了注册一个类视图到一个路由节点，可以使用`app.add_route`方法。它的第一个参数应该是定义的类对`as_view`方法的调用，第二个应该是URL节点。可用的方法是get，post，put，patch和delete。 使用所有这些方法的类看起来如下所示

- 简单示例

  ```python
  from sanic import Sanic
  from sanic.views import HTTPMethodView
  from sanic.response import text
  
  app = Sanic('some_name')
  
  class SimpleView(HTTPMethodView):
  
      def get(self, request):
          return text('I am get method')
  
      def post(self, request):
          return text('I am post method')
  
      def put(self, request):
          return text('I am put method')
  
      def patch(self, request):
          return text('I am patch method')
  
      def delete(self, request):
          return text('I am delete method')
  
  app.add_route(SimpleView.as_view(), '/')
  ```

  对类视图的方法也可以使用`async`语法：

  ```python
  class SimpleAsyncView(HTTPMethodView):
  
    async def get(self, request):
          return text('I am async get method')
  
  app.add_route(SimpleAsyncView.as_view(), '/')
  ```

  

#### URL参数

- 如果需要任何 URL 参数，可以像Sanic 路由中讲的那样，把它们当做类方法的参数。

  ```python
  class NameView(HTTPMethodView):
  
    def get(self, request, name):
        return text('Hello {}'.format(name))
  
  app.add_route(NameView.as_view(), '/<name>')
  ```

  

#### 修饰器

- 如果需要给类添加任何修饰器，可以设置`decorators`类变量。它们会在调用`as_view`是被应用。

  ```python
  class ViewWithDecorator(HTTPMethodView):
    decorators = [some_decorator_here]
  
    def get(self, request, name):
      return text('Hello I have a decorator')
  
    def post(self, request, name):
      return text("Hello I also have a decorator")
  
  app.add_route(ViewWithDecorator.as_view(), '/url')
  ```

  但是，如果只是想装饰某些方法而不是所有的，可以这样操作：

  ```python
  class ViewWithSomeDecorator(HTTPMethodView):
  
      @staticmethod
      @some_decorator_here
      def get(request, name):
          return text("Hello I have a decorator")
  
      def post(self, request, name):
          return text("Hello I don't have any decorators")
  ```

  

#### URL建立

- 如果为一个`HTTPMethodView`建立URL，类的名称就是传递给`url_for`的路由节点。比如：

  ```python
  @app.route('/')
  def index(request):
      url = app.url_for('SpecialClassView')
      return redirect(url)
  
  class SpecialClassView(HTTPMethodView):
      def get(self, request):
          return text('Hello from the Special Class View!')
  
  app.add_route(SpecialClassView.as_view(), '/special_class_view')
  ```

  

#### 使用CompositionView

​	作为`HTTPMethodView`的替代，我们可以使用`CompositionView`把处理函数移到视图类的外面。每个 HTTP 方法的处理函数定义在代码的其它地方，并通过`CompositionView.add`方法添加到视图中。该方法的第一个参数是 HTTP 方法的列表（比如，`['GET', 'POST']`），第二个参数是处理函数。下面的例子展示了`CompositionView`用法：外部函数和内联的lambda函数：

```python
from sanic import Sanic
from sanic.views import CompositionView
from sanic.response import text

app = Sanic(__name__)

def get_handler(request):
    return text('I am a get method')

view = CompositionView()
view.add(['GET'], get_handler)
view.add(['POST', 'PUT'], lambda request: text('I am a post/put method'))

# Use the new view to handle requests to the base URL
app.add_route(view, '/')
```

需要注意的是，目前不能使用`url_for`为`CompositionView`创建URL。



#### Sanic Streaming – 流式传输

- 请求流下面的例子，当请求结束，`await request.stream.read()` 返回 `None`。只有`post, put, patch`修饰器有流参数。已经学习了响应的流式传输。实际上，Sanic还支持请求的流式传输。

  ```python
  from sanic import Sanic
  from sanic.views import CompositionView
  from sanic.views import HTTPMethodView
  from sanic.views import stream as stream_decorator
  from sanic.blueprints import Blueprint
  from sanic.response import stream, text
  
  bp = Blueprint('blueprint_request_stream')
  app = Sanic('request_stream')
  
  
  class SimpleView(HTTPMethodView):
  
      @stream_decorator
      async def post(self, request):
          result = ''
          while True:
              body = await request.stream.read()
              if body is None:
                  break
              result += body.decode('utf-8')
          return text(result)
  
  
  @app.post('/stream', stream=True)
  async def handler(request):
      async def streaming(response):
          while True:
              body = await request.stream.read()
              if body is None:
                  break
              body = body.decode('utf-8').replace('1', 'A')
              await response.write(body)
      return stream(streaming)
  
  
  @bp.put('/bp_stream', stream=True)
  async def bp_put_handler(request):
      result = ''
      while True:
          body = await request.stream.read()
          if body is None:
              break
          result += body.decode('utf-8').replace('1', 'A')
      return text(result)
  
  
  # You can also use `bp.add_route()` with stream argument
  async def bp_post_handler(request):
      result = ''
      while True:
          body = await request.stream.read()
          if body is None:
              break
          result += body.decode('utf-8').replace('1', 'A')
      return text(result)
  
  bp.add_route(bp_post_handler, '/bp_stream', methods=['POST'], stream=True)
  
  
  async def post_handler(request):
      result = ''
      while True:
          body = await request.stream.read()
          if body is None:
              break
          result += body.decode('utf-8')
      return text(result)
  
  app.blueprint(bp)
  app.add_route(SimpleView.as_view(), '/method_view')
  view = CompositionView()
  view.add(['POST'], post_handler, stream=True)
  app.add_route(view, '/composition_view')
  
  
  if __name__ == '__main__':
      app.run(host='127.0.0.1', port=8000)
  ```

  

#### 响应流

- Sanic 允许我们使用`stream`方法向客户端流式传输内容。该方法接受一个带有`StreamingHTTPResponse`对象参数用以写操作的协程回调函数。比如下面的例子：

  ```python
  from sanic import Sanic
  from sanic.response import stream
  
  app = Sanic(__name__)
  
  @app.route("/")
  async def test(request):
      async def sample_streaming_fn(response):
          await response.write('foo,')
          await response.write('bar')
  
      return stream(sample_streaming_fn, content_type='text/csv')
  ```

  这在将源自外部服务（如数据库）的内容流式传输给客户端的情况下非常有用。例如，我们可以使用`asyncpg`提供的异步游标将数据库记录流式传输到客户端：

  ```python
  @app.route("/")
  async def index(request):
      async def stream_from_db(response):
          conn = await asyncpg.connect(database='test')
          async with conn.transaction():
              async for record in conn.cursor('SELECT generate_series(0, 10)'):
                  await response.write(record[0])
  
      return stream(stream_from_db)
  ```

  

#### Sanic处理函数修饰器

​	因为Sanic处理函数就是普通的 Python 函数，所以我们可以想 Flask 那样对它们使用修饰器。比较典型的应用场景是，我们希望在运行处理函数之前执行一些代码。

##### 授权修饰器

- web应用中经常需要特定用户权限才能访问某些路径。我们可以创建修饰器来包装处理函数，检查一个请求的客户端是否被授权访问，并返回适当的响应。

  ```python
  from functools import wraps
  from sanic.response import json
  
  def authorized():
      def decorator(f):
          @wraps(f)
          async def decorated_function(request, *args, **kwargs):
              # run some method that checks the request
              # for the client's authorization status
              is_authorized = check_request_for_authorization_status(request)
  
              if is_authorized:
                  # the user is authorized.
                  # run the handler method and return the response
                  response = await f(request, *args, **kwargs)
                  return response
              else:
                  # the user is not authorized. 
                  return json({'status': 'not_authorized'}, 403)
          return decorated_function
      return decorator
  
  
  @app.route("/")
  @authorized()
  async def test(request):
      return json({'status': 'authorized'})
  ```

  用户授权检查是处理函数修饰器应用的一个典型场景，任何需要用户授权的路由节点都可以使用该修饰器进行修饰。这个功能也可以通过 Sanic 中间件来实现，感兴趣的可以尝试用中间件来实现授权检查。

