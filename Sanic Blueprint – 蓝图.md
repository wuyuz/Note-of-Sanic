## Sanic Blueprint – 蓝图

​	Blueprint 是用于应用程序的子路由的对象。它定义了跟Sanic类相同的添加路由的方法，然后通过灵活的方式注册到应用程序。Blueprint 尤其对大型应用非常有用，可以把应用分成不同的逻辑模块，每个模块就是一个blueprint。比如，一个博客系统，有处理用户的blueprint，有处理文章的blueprint，也有处理留言的blueprint，等等。



#### 第一个Blueprint（蓝图）

​	下面代码是一个非常简单的Blueprint，定义了一个根路径`/`的处理函数，我们把它保存为`my_blueprint.py`，然后就可以在主应用里面导入了

```python
from sanic.response import json
from sanic import Blueprint

bp = Blueprint('my_blueprint')

@bp.route('/')
async def bp_root(request):
    return json({'my': 'blueprint'})
```

#### 注册Blueprint

Blueprint必须注册到应用程序（Sanic类的实例）才能起作用。

```python
from sanic import Sanic
from my_blueprint import bp

app = Sanic(__name__)
app.blueprint(bp)

app.run(host='127.0.0.1', port=8888, debug=True)
```



#### Blueprint（蓝图）的分组和嵌套

​	蓝图也可以作为列表或元组的一部分进行注册，其中注册器将递归循环遍历任何蓝图的子序列并相应地注册它们。 Blueprint.group方法用于简化此过程，允许“模拟”后端目录结构模仿从前端看到的内容。 考虑这个（非常人为的）示例：

```
 #树状结构
api/
├──content/
│  ├──authors.py
│  ├──static.py
│  └──__init__.py
├──info.py
└──__init__.py
app.py
```

分别初始化这个blueprint的嵌套结构里面的`.py`文件：

- api/content/authors.py

  ```python
  from sanic import Blueprint
  from sanic import response
  
  authors = Blueprint('content_authors', url_prefix='/authors')
  
  @authors.route('/')
  async def home(request):
      return response.text(request.path)
  ```

- api/content/static.py

  ```python
  from sanic import Blueprint
  from sanic import response
  
  static = Blueprint('content_static', url_prefix='/static')
  
  @static.route('/')
  async def home(request):
      return response.text(request.path)
  ```

-  `api/content/__init__.py`

  ```python
  from sanic import Blueprint
  
  from .static import static
  from .authors import authors
  
  content = Blueprint.group(static, authors, url_prefix='/content')
  ```

- api/info.py

  ```python
  from sanic import Blueprint
  from sanic import response
  
  info = Blueprint('info', url_prefix='/info')
  
  @info.route('/')
  async def home(request):
      return response.text(request.path)
  ```

- `api/__init__.py`

  ```
  from sanic import Blueprint
  
  from .content import content
  from .info import info
  
  api = Blueprint.group(content, info, url_prefix='/api')
  ```

  上面这组blueprint定义了一组api的路径，它们通过`url_prefix`这个参数，把路径的嵌套关系组织起来。把这组blueprint注册到应用程序：

  ```python
  # file: app.py
  from sanic import Sanic
  from api import api
  
  app = Sanic(__name__)
  app.blueprint(api)
  
  
  if __name__ == '__main__':
      app.run(host='127.0.0.1', port=8888, debug=True)
  ```

  运行`app.py`，就可以访问以下这些路径：

  ```
  http://127.0.0.1:8888/api/info
  http://127.0.0.1:8888/api/content/authors
  http://127.0.0.1:8888/api/content/static
  ```

  

#### Blueprint 组中间件

- 组中间件可以把一个普通中间件应用到该组下所有的blueprint：

  ```python
  bp1 = Blueprint('bp1', url_prefix='/bp1')
  bp2 = Blueprint('bp2', url_prefix='/bp2')
  
  @bp1.middleware('request')
  async def bp1_only_middleware(request):
      print('applied on Blueprint : bp1 Only')
  
  @bp1.route('/')
  async def bp1_route(request):
      return text('bp1')
  
  @bp2.route('/<param>')
  async def bp2_route(request, param):
      return text(param)
  
  group = Blueprint.group(bp1, bp2)
  
  @group.middleware('request')
  async def group_middleware(request):
      print('common middleware applied for both bp1 and bp2')
  
  # Register Blueprint group under the app
  app.blueprint(group)
  ```

  

#### Blueprint 中间件（middleware）

- 使用blueprint还可以全局注册中间件。

  ```python
  @bp.middleware
  async def print_on_request(request):
      print("I am a spy")
  
  @bp.middleware('request')
  async def halt_request(request):
      return text('I halted the request')
  
  @bp.middleware('response')
  async def halt_response(request, response):
      return text('I halted the response')
  ```

  

#### 异常

- 可以为blueprint使用全局异常：

  ```python
  @bp.exception(NotFound)
  def ignore_404s(request, exception):
      return text("Yep, I totally found the page: {}".format(request.url))
  ```

  

#### 启动和停止

​	Blueprint可以在server的启动和停止过程中运行函数。如果在多进程模式（多余1个worker），则在worker创建后触发。
可用的事件有：

- before_server_start：在server开始接受连接之前执行；

- after_server_start： 在Server开始接受连接之后执行；

- before_server_stop： 在Server停止接受连接之前执行；

- after_server_stop： 在server停止并完成所有请求后执行；

  ```python
  bp = Blueprint('my_blueprint')
  
  @bp.listener('before_server_start')
  async def setup_connection(app, loop):
      global database
      database = mysql.connect(host='127.0.0.1'...)
  
  @bp.listener('after_server_stop')
  async def close_connection(app, loop):
      await database.close()
  ```

  

#### 使用案例：API版本控制

​	Blueprint对于API版本控制非常有用，其中一个Blueprint可能指向`/v1/<routes>`，另一个指向`/v2/<routes>`。初始化blueprint时，它可以采用可选的版本参数，该参数将添加到蓝图上定义的所有路径。 此功能可用于实现我们的API版本控制方案。

```python
# blueprints.py
from sanic.response import text
from sanic import Blueprint

blueprint_v1 = Blueprint('v1', url_prefix='/api', version="v1")
blueprint_v2 = Blueprint('v2', url_prefix='/api', version="v2")

@blueprint_v1.route('/')
async def api_v1_root(request):
    return text('Welcome to version 1 of our documentation')

@blueprint_v2.route('/')
async def api_v2_root(request):
    return text('Welcome to version 2 of our documentation')
```

当我们在应用程序上注册我们的blueprint时，路由`/v1/api`和`/v2/api`现在将指向各个blueprint，这允许为每个API版本创建子站点。

```python
# main.py
from sanic import Sanic
from blueprints import blueprint_v1, blueprint_v2

app = Sanic(__name__)
app.blueprint(blueprint_v1)
app.blueprint(blueprint_v2)

app.run(host='0.0.0.0', port=8000, debug=True)
```



#### 使用`url_for`创建URL

如果要为blueprint里面的路由生成URL，要记得路径节点的名字的格式是`<blueprint_name>.<handler_name>`。例如：

```python
@blueprint_v1.route('/')
async def root(request):
    url = request.app.url_for('v1.post_handler', post_id=5) # --> '/v1/api/post/5'
    return redirect(url)


@blueprint_v1.route('/post/<post_id>')
async def post_handler(request, post_id):
    return text('Post {} in Blueprint V1'.format(post_id))
```