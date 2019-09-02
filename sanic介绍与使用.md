## Sanic介绍与使用

**版本要求：**在开始前请确保你同时安装了 [pip](https://pip.pypa.io/en/stable/installing/) 和至少 3.5 以上版本的 Python 。Sanic 使用了新的 `async`/`await` 语法，所以之前的版本无法工作。

##### 安装介绍：

​	Sanic是一个支持 `async/await` 语法的异步无阻塞框架，这意味着我们可以依靠其处理异步请求的新特性来提升服务性能，如果你有`Flask`框架的使用经验，那么你可以迅速地使用`Sanic`来构建出心中想要的应用，并且性能会提升不少，我将同一服务分别用Flask和Sanic编写，再将压测的结果进行对比，发现Sanic编写的服务大概是`Falsk`的1.5倍。

​	仅仅是Sanic的异步特性就让它的速度得到这么大的提升么？是的，但这个答案并不标准，更为关键的是Sanic使用了`uvloop`作为`asyncio`的事件循环，`uvloop`由Cython编写，它的出现让`asyncio`更快，快到什么程度？[这篇](https://links.jianshu.com/go?to=http%3A%2F%2Fcodingpy.com%2Farticle%2Fuvloop-blazing-fast-networking-with-python%2F)文章中有介绍，其中提出速度至少比 nodejs、gevent 和其他Python异步框架要快两倍，并且性能接近于用Go编写的程序，顺便一提，Sanic的作者就是受这篇文章影响，这才有了Sanic。

##### 安装Sanic：

- 保证电脑安装了 Visual C++ 14
- 执行：`python3 -m pip install sanic`

##### 第一个Sanic程序

- 新建一个main.py文件

  ```python
  from sanic import Sanic
  from sanic.response import json
  
  app = Sanic()
  
  @app.route('/')
  #async def语法来定义Sanic处理函数，因为它们是异步函数，当然可以不使用async，那将和Flask无区别
  async def test(request):
      return json({"hello":"world"})
  
  if __name__ == '__main__':
      app.run(host='0.0.0.0',port=8000)
  ```

  

#### 路由

路由允许用户为不同的URL端点指定处理程序函数。

- 比如第一个程序中写的

  ```python
  from sanic.response import json
  # 学习了Flask，也就知道这个路由怎么写了
  @app.route("/")
  async def index(request):
      return json({ "hello": "world" })
  ```

- ##### 请求参数

  要指定一个参数，可以用像这样的角引号<PARAM>包围它。请求参数将作为关键字参数传递给路线处理程序函数

  - 类型一：不指定类型

    ```python
    from sanic import Sanic
    # 与Flask相异，Sanic做了细分，将response相关的功能分化出来
    from sanic.response import json,text
    
    app = Sanic()
    # 开启debug调试
    app.debug = True
    
    @app.route('/tag/<tag>')
    async def tag_handler(request, tag):
        return text('Tag - {}'.format(tag))
    
    if __name__ == '__main__':
        app.run(host='0.0.0.0',port=8000)
     #http://127.0.0.1:8000/tag/python，返回Tag - python
    ```

  - 类型二：为参数指定类型，在参数名后面添加（：类型）。如果参数不匹配指定的类型，Sanic将抛出一个不存在的异常，导致一个404页面

    ```PYTHON
    from sanic import Sanic
    from sanic.response import json,text
    
    app = Sanic()
    app.debug = True
    
    #http://127.0.0.1:9000/number/1
    @app.route('/number/<integer_arg:int>')
    async def integer_handler(request, integer_arg):
        return text('Integer - {}'.format(integer_arg))
    
    #http://127.0.0.1:9000/number/12.0
    @app.route('/number/<number_arg:number>')
    async def number_handler(request, number_arg):
        return text('Number - {}'.format(number_arg))
    
    #http://127.0.0.1:9000/person/junxi
    @app.route('/person/<name:[A-z]+>')
    async def person_handler(request, name):
        return text('Person - {}'.format(name))
    
    #http://127.0.0.1:9000/folder/img1
    @app.route('/folder/<folder_id:[A-z0-9]{0,4}>')
    async def folder_handler(request, folder_id):
        return text('Folder - {}'.format(folder_id))
    
    if __name__ == '__main__':
        app.run(host='0.0.0.0',port=8000)
    ```

    详细讲解：

    ```python
    如上面的代码所示，:type中的type支持的形式有:
        int – int整数
        number – float浮点数
        alpha – str只含有大小写字母的字符串
        path – str 匹配正则r"[^/].*?"的字符串
        uuid – uuid.UUID
        string – string字符串，不指定:type时的默认类型
        任何形式的正则表达式
    
    其实，:type的type就是通过正则表达式匹配的，具体可以看Sanic源码中的router.py的REGEX_TYPES的定义：
        REGEX_TYPES = {
            "string": (str, r"[^/]+"),
            "int": (int, r"\d+"),
            "number": (float, r"[0-9\\.]+"),
            "alpha": (str, r"[A-Za-z]+"),
            "path": (str, r"[^/].*?"),
            "uuid": (
                uuid.UUID,
                r"[A-Fa-f0-9]{8}-[A-Fa-f0-9]{4}-"
                r"[A-Fa-f0-9]{4}-[A-Fa-f0-9]{4}-[A-Fa-f0-9]{12}",
            ),  
        }
    ```



- ##### 请求类型

  ​	路由装饰器接受一个可选的参数、方法，它允许处理程序函数与列表中的任何HTTP方法一起工作，默认情况下，URL上定义一个路由只会对GET请求有效，我们可以通过methods来指定HTTP的请求方式

  - 通过route/请求方式来匹配视图函数

    ```python
    from sanic import Sanic
    from sanic.response import json,text
    
    app = Sanic()
    app.debug = True
    
    #方式一： 使用methods参数设置
    #使用post定制headers=application/json
    #http://127.0.0.1:8000/post, raw数据：{'name': 'wang'}
    @app.route('/post', methods=['POST'])
    async def post_handler(request):
        #POST request - {'name': 'wang'}
        return text('POST request - {}'.format(request.json))
    
    #http://127.0.0.1:8000/get?name=wang&xing=li
    @app.route('/get', methods=['GET'])
    async def get_handler(request):
        #GET request - {'name': ['wang'], 'xing': ['li']}
        return text('GET request - {}'.format(request.args))
    
    #方式二：配置路由后自动判断请求方式，通过app.post/get...来设计，满足了restful的根据请求方式分发路径
    #http://127.0.0.1:8000/post2 和方式一的方式和结果一致
    @app.post('/post2')
    async def posts_handlers(request):
        return text('POST request - {}'.format(request.json))
    
    #http://127.0.0.1:8000/get2?name=wang
    @app.get('/get2')
    async def get_handler(request):
        return text('GET request - {}'.format(request.args))
    
    if __name__ == '__main__':
        app.run(host='0.0.0.0',port=8000)
    ```

    详细讲解：`@app.route`还有一个可选参数`host`，它可以是字符串或列表。它限定了路由对应的host。如果还有不带host的路由，则该路由是默认的。

    ```python
    @app.route('/get', methods=['GET'], host='yuanrenxue.com')
    async def get_handler(request):
        return text('GET request - {}'.format(request.args))
    
    
    #如果header中的host不匹配yuanrenxue.com，该路由将任意被使用
    @app.route('/get', methods=['GET'])
    async def get_handler(request):
        return text('GET request in default - {}'.format(request.args))
    ```

    

- ##### 添加路由

  通过`app.add_route()`添加路由和视图函数之间的映射关系，类似于Flask

  ```python
  from sanic import Sanic
  from sanic.response import json,text
  
  app = Sanic()
  app.debug = True
  
  # http://127.0.0.1:9000/test1
  async def handler1(request):
      return text('ok')
  
  #http://127.0.0.1:9000/folder2/aaa
  async def handler2(request, name):
      return text('Folder - {}'.format(name))
  
  #http://127.0.0.1:9000/personal2/a
  async def personal_handler2(request, name):
      return text('Person - {}'.format(name))
  
  
  app.add_route(handler1, '/test1')
  app.add_route(handler2, '/folder2/<name>')
  app.add_route(personal_handler2, '/personal2/<name:[A-z]>', methods=['GET'])
  
  if __name__ == '__main__':
      app.run(host='0.0.0.0',port=8000)
  ```



#### 反向路由解析

Sanic 提供了一个 `url_for` 方法来生成一组基于这个处理方法名称的 URL。如果你想在 app 中避免硬编码 url，这是有用的。

- 反向路由总结

  ```python
  from sanic import Sanic
  from sanic.response import text,redirect
  
  app = Sanic()
  app.debug = True
  
  @app.route("/")
  async def index(request):
      url = app.url_for('post_handler', post_id=5)
      return redirect(url)
  
  @app.route('posts/<post_id>')
  async def post_handler(request, post_id):
      return text('Post - {}'.format(post_id))
  
  #区别于Flask的url_for,Sanic写在函数外面
  #方式一：关键字传参，如果传递的参数不是url路径名参数，则会以url参数的形式传递
  url = app.url_for('post_handler', post_id=5, arg_one='one', arg_two='two')
  # /posts/5?arg_one=one&arg_two=two
  
  #方式二：多元的参数也会被 url_for 传递。例如：
  url = app.url_for('post_handler', post_id=5, arg_one=['one', 'two'])
  # /posts/5?arg_one=one&arg_one=two
  
  #特殊传参：还有一些特殊的参数 (_anchor, _external, _scheme, _method, _server)
  # 传递到 url_for 会有一个指定的 url 建立 (_method 现在不被支持并且会被忽略)。例如：
  #_anchor 称为锚部分
  url = app.url_for('post_handler', post_id=5, arg_one='one', _anchor='anchor')
  # /posts/5?arg_one=one#anchor
  
  url = app.url_for('post_handler', post_id=5, arg_one='one', _external=True)
  # //server/posts/5?arg_one=one
  # _external requires passed argument _server or SERVER_NAME in app.config or
  # url will be same as no _external
  
  
  if __name__ == '__main__':
      app.run(host='0.0.0.0',port=8000)
  
  ```



#### Sanic 支持 WebSocket

我们知道Tonado是支持WebSocket的，但是Flask/Django都是不支持的，对于 WebSocket 协议的路由可以使用 `@app.websocket` 装饰器定义

- 简单使用

  ```python
  from sanic import Sanic
  from sanic.response import text,redirect
  
  app = Sanic()
  app.debug = True
  
  #方式一：通过装饰器
  @app.websocket('/feed')
  async def feed(request, ws):
      while True:
          data = 'hello!'
          print('Sending：' + data)
          # 在async 的函数中，只要涉及到i/o操作的时候都要使用await关键字，发送数据涉及i/o
          await ws.send(data)
          # 监听连接，涉及i/o
          data = await ws.recv()
          print('Received：', data)
  
  #方式二：
  async def feed(request, ws):
      pass
  #my_websocket_handler 是我们创建的websocket句柄
  app.add_websocket_route(my_websocket_handler, '/feed')
  
  if __name__ == '__main__':
      app.run(host='0.0.0.0',port=8000)
  
   #一个 WebSocket 路由的处理程序作为第一个参数传递请求，同时一个 WebSocket 协议对象作为第二个参数。这个协议对象有 send 和 recv 方法分别发送和接受数据。
  ```

- 设置路径的严格模式

  ```python
  from sanic import Sanic,Blueprint
  from sanic.response import text,redirect
  
  app = Sanic()
  app.debug = True
  
  # provide default strict_slashes value for all routes,对所有的路径都遵循严格路径匹配
  app = Sanic('test_route_strict_slash', strict_slashes=True)
  
  # you can also overwrite strict_slashes value for specific route
  #必须访问/get/才能访问
  @app.get('/get/', strict_slashes=True)
  def handler(request):
      return text('OK')
  
  # It also works for blueprints
  bp = Blueprint('test_bp_strict_slash', strict_slashes=True)
  
  #都可以访问加不加/
  @bp.get('/bp/get', strict_slashes=False)
  def handler(request):
      return text('OK')
  
  #注册蓝图
  app.blueprint(bp)
  
  if __name__ == '__main__':
      app.run(host='0.0.0.0',port=8000)
  
  ```

  详细讲解：通过传递参数`name`给路由装饰器就可以自定义路由名称，该参数会覆盖默认使用`hanler.__name__`属性的路由名称。这个路由名称是给`url_for`使用的。

  ```python
  app = Sanic('test_named_route')
  
  @app.get('/get', name='get_handler')
  def handler(request):
      return text('OK')
  
  # 当为上面的路由使用`url_for`时，
  # 应该使用 `app.url_for('get_handler')`
  # 而不是`app.url_for('handler')`
  
  
  # 同样适用于blueprints
  bp = Blueprint('test_named_bp')
  
  @bp.get('/bp/get', name='get_handler')
  def handler(request):
      return text('OK')
  
  app.blueprint(bp)
  
  # 应该使用 `app.url_for('test_named_bp.get_handler')`
  # 而不是 `app.url_for('test_named_bp.handler')`
  
  
  # 同一个url的不同方法可以使用不同的名字，但是对应`url_for`来说，它们都是对应同一个url：
  
  @app.get('/test', name='route_test')
  def handler(request):
      return text('OK')
  
  @app.post('/test', name='route_post')
  def handler2(request):
      return text('OK POST')
  
  @app.put('/test', name='route_put')
  def handler3(request):
      return text('OK PUT')
  
  # 下面生成的url都相同，可以使用它们中的任何一个：
  # '/test'
  app.url_for('route_test')
  # app.url_for('route_post')
  # app.url_for('route_put')
  
  # 同一个处理函数名对应不同方法时，
  # 就需要知道`name` (为了`url_for`)
  @app.get('/get')
  def handler(request):
      return text('OK')
  
  @app.post('/post', name='post_handler')
  def handler(request):
      return text('OK')
  
  # 因此：
  # app.url_for('handler') == '/get'
  # app.url_for('post_handler') == '/post'
  ```

  

#### 用户定义路径别名

- 与Flask路径和视图函数名的区别

  ```python
  @app.route('/get',name='s')
  def handler(request):
      return text('OK')
  
  @app.route('/get1',name='s')
  def handler(request):
      return text('OK1')
   #这样的定义试图函数和别名，在Sanic中是不会报错的，也就是说我们的函数名可以不一样，那么怎么绑定函数那？所以必须要区别开，所以name尽量不要重复，虽然也不会报错
  ```

- 通过路径别名来获取相应的路径

  ```python
  from sanic import Sanic,Blueprint
  from sanic.response import text,redirect
  app = Sanic()
  app.debug = True
  app = Sanic('test_named_route')
  
  @app.get('/get', name='get_handler')
  def handler(request):
      return text('OK')
  
  # then you need use `app.url_for('get_handler')`,instead of # `app.url_for('handler')`
  
  # 不再使用函数名来绑定，且不会重名
  app.url_for('get_handler')
  
  # It also works for blueprints
  bp = Blueprint('test_named_bp')
  
  @bp.get('/bp/get', name='get_handler')
  def handler(request):
      return text('OK')
  
  app.blueprint(bp)
  
  # then you need use `app.url_for('test_named_bp.get_handler')`，instead of `app.url_for('test_named_bp.handler')`
  #通过蓝图别名.函数别名区分
  app.url_for('test_named_bp.get_handler')
  
  # different names can be used for same url with different methods
  @app.get('/test', name='route_test')
  def handler(request):
      return text('OK')
  
  @app.post('/test', name='route_post')
  def handler2(request):
      return text('OK POST')
  
  @app.put('/test', name='route_put')
  def handler3(request):
      return text('OK PUT')
  
  # below url are the same, you can use any of them '/test'
  app.url_for('route_test')
  app.url_for('route_post')
  app.url_for('route_put')
  
  # for same handler name with different methods
  # you need specify the name (it's url_for issue)
  @app.get('/get')
  def handler(request):
      return text('OK')
  
  @app.post('/post', name='post_handler')
  def handler(request):
      return text('OK')
  
  # app.url_for('handler') == '/get'
  # app.url_for('post_handler') == '/post'
  
  if __name__ == '__main__':
      app.run(host='0.0.0.0',port=8000)
  ```

  

#### 为静态文件路径建立URL

​	现在你可以使用 `url_for` 给静态资源建立 url。如果是为文件直传的， `filename` 会被忽略。和其他框架一样，为了隐藏本地静态文件路径，统一标识为同意的别名如'/static'，我们需要使用url来配置

- 简单了解

  ```python
  from sanic import Sanic,Blueprint
  from sanic.response import text,redirect
  
  app = Sanic()
  app.debug = True
  
  app = Sanic('test_static')
  app.static('/static', './static')
  app.static('/uploads', './uploads', name='uploads')
  app.static('/the_best.png', '/home/ubuntu/test.png', name='best_png')
  
  #url_prefix 表示必须添加bp为url的前缀
  bp = Blueprint('bp', url_prefix='bp')
  bp.static('/static', './static')
  bp.static('/uploads', './uploads', name='uploads')
  bp.static('/the_best.png', '/home/ubuntu/test.png', name='best_png')
  app.blueprint(bp)
  
  
  #url_for中的第一个参数是'static'表示该路径是静态路径，第二个参数name是相对于的路径别名，我们可以通过它找到xx.static()中对应的路径，
  #注意： 蓝图中加了url_prefix='bp'，表示静态文件也要添加bp路径，但是对应的本地都是'./static'
  print(app.url_for('static', filename='file.txt'))  #/static/file1.txt
  print(app.url_for('static', name='static', filename='file.txt'))  #/static/file.txt
  print(app.url_for('static', name='static', filename='file.txt')) #/static/file.txt
  print(app.url_for('static', name='best_png',filename='xxx'))  #/the_best.png/xxx
  
  # then build the url
  app.url_for('static', filename='file.txt') == '/static/file.txt'
  app.url_for('static', name='static', filename='file.txt') == '/static/file.txt'
  app.url_for('static', name='uploads', filename='file.txt') == '/uploads/file.txt'
  app.url_for('static', name='best_png',filename='xxx') == '/the_best.png'
  
  # # blueprint url building
  app.url_for('static', name='bp.static', filename='file.txt') == '/bp/static/file.txt'
  app.url_for('static', name='bp.uploads', filename='file.txt') == '/bp/uploads/file.txt'
  # app.url_for('static', name='bp.best_png') == '/bp/static/the_best.png'
  
  if __name__ == '__main__':
      app.run(host='0.0.0.0',port=8000)
  
  ```

  

- 静态路径补充

  ```python
  from sanic import Sanic,Blueprint
  from sanic.response import text,redirect
  
  app = Sanic()
  app.debug = True
  
  #这里注册了static url为/static 实际物理路径为./static
  app = Sanic('test_static')
  app.static('/static', './static')
  app.static('/uploads', './static/uploads', name='uploads')
  
  #这样：http://127.0.0.1:8000/static/xx.txt 就能访问项目目录下的static/xx.txt了
  @app.route("/")
  async def index(request):
      url = app.url_for('static',name='uploads',filename='xx')
      return redirect(url)
  
  @app.route("/uploads/xx")
  async def index(request):
      return text('ok!')
  
  #url_prefix 表示必须添加bp为url的前缀,不添加url_prefix,会报错出现重复函数名
  bp = Blueprint('bp',url_prefix='bp')
  
  
  #这里注册了static url为/static 实际物理路径为./static
  bp.static('/static', './static')
  #http://127.0.0.1:8000/static/uploads/xxx.txt可以访问文件
  bp.static('/uploads', './uploads', name='uploads')
  
  #http://127.0.0.1:8000/bp/uploads/xxx，会跳转到bp的/uploads/xxx函数中
  @bp.route("/bp")
  async def index(request):
      url = app.url_for('static',name='bp.uploads',filename='xxx')
      print(url)
      return redirect(url)
  
  @bp.route("/uploads/xxx")
  async def index(request):
      return text('bp:ok!')
  
  app.blueprint(bp)
  
  if __name__ == '__main__':
      app.run(host='0.0.0.0',port=8000)
  
  ```

  