## Sanic 配置 、请求响应 

项目中任何相当复杂的应用程序都需要未融入实际代码的配置。不同环境或安装的设置可能不同



#### 加载配置文件的方式

- 方式一（在主页面中使用）：SANIC将配置保存在应用程序对象的config属性中。配置对象只是一个可以使用点标记法或类似字典进行修改的对象

  ```python
  app = Sanic('myapp')
  app.config.DB_NAME = 'appdb'
  app.config.DB_USER = 'appuser'
  
   #由于config对象实际上是一个字典，因此可以使用updata方法一次性传值
  db_settings = {
      'DB_HOST': 'localhost',
      'DB_NAME': 'appdb',
      'DB_USER': 'appuser'
  }
  app.config.update(db_settings)
  
   #一般来说，约定只使用大写配置参数。下面描述的加载配置的方法只查找这样的大写参数。
  ```

- 方式二（从对象中加载）：如果有很多配置值，并且它们有合理的默认值，那么将它们放入模块可能会有所帮助

  ```python
  import myapp.default_settings
  
  app = Sanic('myapp')
  # default_settings是一个配置类的对象
  app.config.from_object(myapp.default_settings)
  
  # 或者通过路径配置：
  app = Sanic('myapp')
  app.config.from_object('config.path.config.Class')
  ```



#### 日志

- 简单使用

  ```python
  from sanic import Sanic
  from sanic.log import logger
  from sanic.response import text
  
  app = Sanic('test')
  
  @app.route('/')
  async def test(request):
      # 当出发请求时回打印下面一句话
      logger.info('Here is your log')
      return text('Hello World!')
  
  if __name__ == "__main__":
    app.run(debug=True, access_log=True)
  ```

- 要使用自己的日志配置，只需使用 `logging.config.dictConfig` 或通过 `log_config` 初始化时 `Sanic` 应用程序

  ```python
  from sanic import Sanic,Blueprint
  from sanic.response import text,redirect
  import logging
  
  class Log(object):
      __instance = None
      def __new__(cls, *args, **kwargs):
          if not cls.__instance:
              cls.__instance = super().__new__(cls)
          return cls.__instance
  
      def __init__(self,level = logging.DEBUG):
          if 'logger' not in self.__dict__:
              logger = logging.getLogger()
              logger.setLevel(level)
              fh = logging.FileHandler('test.log', encoding='utf-8')
              ch = logging.StreamHandler()
              formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')
              fh.setFormatter(formatter)
              ch.setFormatter(formatter)
              logger.addHandler(fh)
              logger.addHandler(ch)
              self.logger = logger
  log = Log()
  # 设置日志去重写默认的配置
  app = Sanic('test')
  
  @app.route("/")
  def test(request):
      log.logger.info("received request; responding with 'hey'")
      return text("hey")
  
  if __name__ == '__main__':
      # 通过设置access_log 可以开关默认的打印日志
      app.run(host="0.0.0.0", port=8000)
  ```

  

#### 请求数据

当服务端收到HTTP请求时，路由函数将传递一个请求对象。

以下变量可以作为request对象的属性访问

- json（any) - JSON body,任何json的请求体，当客户端POST来的数据是json格式时，可以通过`request.json`来访问：

  ```python
  from sanic import Sanic,Blueprint
  from sanic.response import json
  
  app = Sanic()
  
  # 打开postman 在请求体中的发送json数据
  @app.route("/json")
  def post_json(request):
      return json({ "received": True, "message": request.json })
  
  if __name__ == '__main__':
      app.run(host="0.0.0.0", port=8000,debug=True)
  ```

- `args`（dict） - 查询字符串变量。查询字符串变量就是说URL中问号`?`及其后面的部分，比如，`?name=jim&age=12`。这样带查询变量的URL被解析后，`args`字典就是这样的：`{'name':['jim'], 'age': ['12']}`。request对象的属性`query_string`就是未解析的字符串值：`name=jim&age=12`。该属性提供了默认的解析查询变量的策略，稍微我们介绍如何改变解析策略。

  ```python
      #/query_string?name=wang&xing=li
      @app.route("/query_string")
      def query_string(request):
          return json({ "parsed": True, "args": request.args, "url": request.url, "query_string": request.query_string })
      
      返回数据：
          {
              "parsed":true,
              "args":{"name":["wang"],"xing"["li"]},
              "url":"http://127.0.0.1:8000/query_string?name=wang&xing=li",
              "query_string":"name=wang&xing=li"，
           }
  ```

- `raw_args`（dict） - 在许多情况下，您需要访问压缩程度较低的字典中的url参数。对于之前的同一个URL `?key1=value1&key2=value2`， `raw_args`字典看起来就像：`{'key1': 'value1', 'key2': 'value2'}`。

  ```python
  #/test_request_args?school=qh
  @app.route("/test_request_args")
  async def test_request_args(request):
      return json({
          "parsed": True,
          "url": request.url,
          "query_string": request.query_string,
          "args": request.args,
          "raw_args": request.raw_args,
          "query_args": request.query_args,
      })
      
      #返回结果
      {
          "parsed": true,
          "url": "http://127.0.0.1:8000/test_request_args?school=qh",
          "query_string": "school=qh",
          "args": {
              "school": [
                  "qh"
              ]
          },
          "raw_args": {
              "school": "qh"
          },
          "query_args": [
              [
                  "school",
                  "qh"
              ]
          ]
      }
  ```

- files类型（dictionary of `File` objects） ，拥有name、body和type的文件对象的字典客户端（浏览器）上传的文件包含在Form data中，`files`字典的key是Form对象中文件对象的名称（不是文件名），这个文件对象有name（文件名）、body（文件数据）和type（文件类型）三个属性。

  ```python
  @app.route("/files")
  def post_json(request):
      # 获取文件类型
      test_file = request.files.get('test')
  
      file_parameters = {
          'body': test_file.body,
          'name': test_file.name,
          'type': test_file.type,
      }
  
      return json({ "received": True, "file_names": request.files.keys(), "test_file_parameters": file_parameters })
  
  ---------------------------------------------------------------------------
   #file的用法：
  @app.route('/')
  async def index(request):
      html = ('<html><body>'
              '<form action="/files" method="post" enctype="multipart/form-data">'
              '<input type="file" name="file1" /> <br />'
              '<input type="file" name="file2" /> <br />'
              '<input type="submit" value="Upload" />'
              '</form>'
              '</body></html>')
      return response.html(html)
  
  @app.route("/files", methods=['POST'])
  async def post_json(request):
      test_file = request.files.get('file1')
      file_parameters = {
          'body': len(test_file.body),
          'name': test_file.name,
          'type': test_file.type,
      }
  
      return response.json({
          "received": True,
          "file_object_names": request.files.keys(),
          "test_file_parameters": file_parameters
      })
  
   #注意 从结果可以看出，requests.files.keys()是网页html中input的name，不是文件名。
  ```

- `form` （dict） - post表单变量。

  ```python
  @app.route("/form")
  def post_json(request):
      return json({ "received": True, "form_data": request.form, "test": request.form.get('test') })
  ```

- `body`（bytes） - 发送原始主体。无论内容类型如何，该属性都允许检索请求的原始数据。

  ```python
  @app.route("/users", methods=["POST",])
  def create_user(request):
      return text("You are trying to create a user with the following POST: %s" % request.body)
  
  
   # 请求体数据都在这
   #http://127.0.0.1:8000/test?school=qh
  @app.route("/test")
  async def test_request_args(request):
      return json({
          "raw_args": request.raw_args,
          'body':str(request.body),
      })
  
  # 结果：
  {
      "raw_args": {
          "school": "qh"
      },
      "body": "b'{\\n\\t\"name\":\"wang\"\\n}'"
  }
  ```

- `headers` （dict） - 包含请求标头的不区分大小写的字典。

- `ip` （str） - 请求者的IP地址。

- `port` （str） - 请求者的端口地址。

- `socket` （tuple） - 请求者的（IP，端口）。

- `url`：请求的完整URL，即： `http://localhost:8000/posts/1/?foo=bar`

- `scheme`：与请求关联的URL方案：`http`或`https`

- `host`：与请求关联的主机： `localhost:8080`

- `path`：请求的路径： `/posts/1/`

- `query_string`：请求的查询字符串：`foo=bar`或一个空白字符串`''`

- `uri_template`：匹配路由处理程序的模板： `/posts/<id>/`

- `token`：授权标头(Authorization)的值： `Basic YWRtaW46YWRtaW4=`





#### 使用 get 和 `getlist `访问数据

返回字典的请求属性实际上会返回一个`dict`被调用的子类 `RequestParameters`。使用这个对象的关键区别在于`get`和`getlist`方法之间的区别。主要用户获取url上携带的params元素。

- `get(key, default=None)`按照正常操作，除了当给定键的值是列表时，*只返回第一个项目*。

- `getlist(key, default=None)`正常操作，*返回整个列表*。

  ```python
  app = Sanic()
  
  #/test?title=1&title=2
  @app.route("/test")
  async def test_request_args(request):
      #{"get": "1"}
      # return json({'get':request.args.get('title')})
      
      #{"get": [ "1", "2"]}
      return json({'get': request.args.getlist('title')})
  ```

  

#### 使用request.endpoint属性访问处理程序名称

request.endpoint属性保存处理程序的名称。例如，下面的路由将返回“hello”。

- 简单实例

  ```python
  app = Sanic()
  @app.get("/")
  def hello(request):
      return text(request.endpoint)
  ```

- 蓝图中使用：它将包含两者，并用一个句号分隔。例如，下面的路由将返回foo.bar

  ```python
  from sanic import Sanic
  from sanic import Blueprint
  from sanic.response import text
  
  app = Sanic(__name__)
  blueprint = Blueprint('foo')
  
  @blueprint.get('/')
  async def bar(request):
      return text(request.endpoint)
  
  app.blueprint(blueprint)
  
  app.run(host="0.0.0.0", port=8000, debug=True)
  ```

  

#### 响应

- 纯文本（text）

  ```python
  from sanic import response
  
  @app.route('/text')
  def handle_request(request):
      return response.text('Hello world!')
  ```

  详细讲解：Sanic 返回纯文本内容给浏览器

  ```python
  response.text() 语法：
      def text(
          body,
          status=200, headers=None,
          content_type="text/plain;
          charset=utf-8"
      )
      
  response.text() 参数：
      body：响应要返回的文本字符串；
      status：默认 http 状态码200，正常返回不要修改；
      headers：自定义 http 响应头；
      content_type：纯文本的content type，不要修改；
  
  这里面，body是必需的参数，可以通过传入headers来自定义响应头，其它参数不要修改。
  比如，自定义响应头headers： ---> 在响应头：response header中发现多了一组键值
  return text('Welcom to Python',
              headers={'X-Serverd-By': 'YuanRenXue Python'})
  
  -------------------------------------------------------------------------
   # response.text() 返回值，返回一个HTTPResponse类的实例。多数情况下，路由函数直接返回这个实例。当需要再进一步处理响应（比如，设置响应cookies）时，要把它赋值给一个变量。  
      
  @app.route('/text')
  async def text(request):
      return response.text(
          'Welcom to Python',
          headers={'X-Serverd-By': 'YuanRenXue Python'}
      )
  
  #结果：
  HTTP/1.1 200 OK
  Connection: keep-alive
  Keep-Alive: 5
  X-Serverd-By: YuanRenXue Python
  Content-Length: 25
  Content-Type: text/plain; charset=utf-8
  ```

  

- HTML

  ```python
  from sanic import response
  
  @app.route('/html')
  def handle_request(request):
      return response.html('<p>Hello world!</p>')
  ```

  详细讲解：Sanic 返回html文本内容给浏览器。一般在服务器端渲染网页的web应用返回的都是html文本，Sanic可借助jinja2实现html模板的渲染。

  ```python
  response.html() 语法：
  def html(body, status=200, headers=None)
  
  response.html() 参数
      body：响应要返回的html文本字符串；
      status：默认 http 状态码200，正常返回不要修改；
      headers：自定义 http 响应头
      
  response.html() 返回值
  	返回一个HTTPResponse类的实例。多数情况下，路由函数直接返回这个实例。当需要再进一步处理响应（比如，设置响应cookies）时，要把它赋值给一个变量。
  	
  @app.route('/html')
  async def html(request):
      return response.html(
          'Welcom to Python',
          headers={'X-Serverd-By': 'YuanRenXue Python'}
      )
  ```

  

- JSON

  ```python
  from sanic import response
  
  @app.route('/json')
  def handle_request(request):
      return response.json({'message': 'Hello world!'})
  ```

  详解讲解：Sanic 返回json格式的文本内容给浏览器，这个json数据格式多用于网页异步加载AJAX的后端接口，或者是实现API与http客户端进行数据交换。

  ```python
  response.json() 语法
      def json(
          body,
          status=200,
          headers=None,
          content_type="application/json",
          dumps=json_dumps,
          **kwargs
      )
  
  response.json() 参数
      body：响应要返回的JSON数据，通常是一个字典；
      status：默认 http 状态码200，正常返回不要修改；
      headers：自定义 http 响应头；
      content_type：纯文本的content type，不要修改；
      dumps：把json数据（字典）dumps为字符串的函数；
      kwargs：dumps函数的参数；
      这里面，body是必需的参数，可以通过传入headers来自定义响应头，content_type参数不要修改。如果响应自己的dumps函数可以为设置dumps参数。如果想修改默认的dumps函数的参数，可以传值给kwargs，比如，eunsure_ascii=False，indent=4等等，更多参数详见Python的json模块的json.dumps()函数。
      
  response.json() 返回值
  	返回一个HTTPResponse类的实例。多数情况下，路由函数直接返回这个实例。当需要再进一步处理响应（比如，设置响应cookies）时，要把它赋值给一个变量。
      
  @app.route('/json')
  async def json(request):
      return response.json(
          {'msg': 'Welcom to Python'},
          headers={'X-Serverd-By': 'YuanRenXue Python'},
          ensure_ascii=False,
          indent=4,
      )    
  ```

  

- 文件

  ```python
  from sanic import response
  
  app = Sanic()
  
  @app.route('/file')
  async def handle_request(request):
      #将会打开xx.txt文件，在前端读取
      return await response.file('./static/xx.txt')
  ```

  详细讲解：Sanic 返回文件数据给浏览器，以此实现文件下载功能。读写文件是一个IO操作，所以这是一个异步函数，使用时记得加`await`。

  ```python
  response.file() 语法
      async def file(
          location,
          status=200,
          mime_type=None,
          headers=None,
          filename=None,
          _range=None,
      )
  
  response.file() 参数
      location：响应要返回的文件的路径；
      status：默认 http 状态码200，正常返回不要修改；
      mime_type：文件格式；
      headers：自定义 http 响应头；
      filename：如果传值则写入响应头headers的Content-Disposition；
      _range：指定文件的某一部分；
  	这里面，location是必需的参数；可以通过传入headers来自定义响应头；如果没有传入mime_type参数，其内部会使用模块mimetypes的mimetypes.guess_type()函数通过filename来获得；_range对象需要有四个属性：start, end, size, total来描述截取文件中的某一部分。
  	
  response.file() 返回值
  返回一个HTTPResponse类的实例。多数情况下，路由函数直接返回这个实例。当需要再进一步处理响应（比如，设置响应cookies）时，要把它赋值给一个变量。	
  
  @app.route('/file')
  async def file(request):
      return await response.file(
          './welcom-to-猿人学.jpg',
          headers={'X-Serverd-By': 'YuanRenXue Python'}
      )
  ```

  

- stream 数据：对于一些大型文件，如：图片、音频、视频等，Sanic已经为我们搭建了byte类型文件传输方法。Sanic 返回流数据给浏览器。流数据的意思就是，不是一次性把所有数据返回，而是一部分一部分地返回。

  ```python
  @app.route('/big_file')
  async def handle_request(request):
      return await response.file_stream('./static/1.jpg')
  
  -----------------------------------------------
  app = Sanic()
  #http://127.0.0.1:8000/static/1.jpg 可以访问到图片
  app.static('static','./statics')
  @app.route('/big_file')
  async def handle_request(request):
      return await response.file_stream('./statics/1.jpg')
  ```

  详细讲解:

  ```python
  response.stream() 语法
      def stream(
          streaming_fn,
          status=200,
          headers=None,
          content_type="text/plain; charset=utf-8",
      )
      
  response.stream() 参数
      streaming_fn：用于流响应写数据的协程函数；
      status：默认 http 状态码200，正常返回不要修改；
      headers：自定义 http 响应头；
      content_type：纯文本的content type，按需修改；
      这里面，streaming_fn是必需的参数，可以通过传入headers来自定义响应头，其它参数不要修改。
  
  response.stream() 返回值,返回一个StreamingHTTPResponse类的实例
  
  @app.route('/stream')
  async def streaming(request):
      async def streaming_fn(response):
          await response.write('Welcom to @{}\n'.format(time.strftime('%H:%M:%S')))
          await asyncio.sleep(3)
          await response.write('Python @{}\n'.format(time.strftime('%H:%M:%S')))
      return response.stream(
          streaming_fn,
          headers={'X-Serverd-By': 'YuanRenXue Python'}
      )
  
  ```

  

- 重定向

  ```python
  @app.route('/redirect')
  def handle_request(request):
      return response.redirect('/json')
  
  
  --------------------------------
   #重定向加反向解析
  @app.route('/redirect')
  def handle_request(request):
      return response.redirect('/')  #重定向
  
  @app.route('/redirect1')
  def handle_request(request):
      url = app.url_for('index')  #反向解析
      return response.redirect(url)
  
  @app.route('/',name='index')
  async def handle_request(request):
      return text('首页')
  ```

  详细讲解：Sanic 返回一个重定向URL让浏览器去访问。它是通过在响应头headers里面设置 Location 来实现的。

  ```python
  response.redirect() 语法
      def redirect(
          to,
          headers=None, 
          status=302,
          content_type="text/html; charset=utf-8"
      )
      
  response.redirect() 参数
      to：响应要返回的重定向URL字符串；
      status：默认 http 状态码302，正常返回不要修改；
      headers：自定义 http 响应头；
      content_type：HTTP 响应头的 content type，不要修改    
      这里面，to是必需的参数，可以通过传入headers来自定义响应头，其它参数不要修改。
  比如，自定义响应头headers：
  
  @app.route('/redirect')
  async def redirect(request):
      return response.redirect(
          'https://www.yuanrenxue.com/',
          headers={'X-Serverd-By': 'YuanRenXue Python'}
      )
  ```

  

- raw 原生数据响应

  ```python
  @app.route('/raw')
  def handle_request(request):
      # 访问网址，会将数据按照byte进行下载，因为我们传给网页的响应数据的格式识别不了，所以浏览器会自动下载
      return response.raw(b'raw data')
  ```

- 修改标题或状态

  ```python
  @app.route('/json')
  def handle_request(request):
      return response.json(
          {'message': 'Hello world!'},
          headers={'X-Served-By': 'sanic'},
          status=200
      )
  ```

  详解讲解：Sanic 返回二进制数据给浏览器。

  ```
  response.raw() 语法
      def raw(
          body,
          status=200, headers=None,
          content_type="application/octet-stream;
      )	
      
  response.raw() 参数
      body：响应要返回的bytes字符串，不是str；
      status：默认 http 状态码200，正常返回不要修改；
      headers：自定义 http 响应头；
      content_type：纯文本的content type，不要修改
      raw()跟text()非常像，不同的是，raw()传入的body必须是bytes字符串（即，str编码后的结果）。这里面，body是必需的参数，可以通过传入headers来自定义响应头，其它参数不要修改。
  ```

  

#### HTTPResponse 类

大多数情况下，我们的web 应用返回的都是`HTTPResponse`类的实例，这包括纯文本(Plain Text)、HTML、JSON、文件（File）、重定向（Redirect）、原数据（Raw）。它们的不同，往往体现在这个类的初始化参数`content_type`上面。

- 下面是`HTTPResponse`类的初始化声明源码：

  ```python
  class HTTPResponse(BaseHTTPResponse):
      __slots__ = ("body", "status", "content_type", "headers", "_cookies")
  
      def __init__(
          self,
          body=None,
          status=200,
          headers=None,
          content_type="text/plain",
          body_bytes=b"",
      ):
          self.content_type = content_type
  
          if body is not None:
              self.body = self._encode_body(body)
          else:
              self.body = body_bytes
  
          self.status = status
          self.headers = CIMultiDict(headers or {})
          self._cookies = None
  ```

  ​	子模块`response`的对应的响应函数最终都会返回一个该类的实例对象。通过给该类的初始化方法传递不同的参数值达到生成不同类型的响应的目的。`HTTPResponse`类有个主要方法`output()`用来生成最终的`bytes`字符串返回给浏览器（客户端）。我们不需要理解它的具体实现，只需要知道有它的存在就可以了。除非我们想继承`HTTPResponse`类实现自己的特殊类。

#### StreamingHTTPResponse 类

该类是流响应使用的，对应`response.stream()`函数和`response.file_stream()`函数。

- 该类的初始化方法与`HTTPResponse`类似但又有不同：

  ```
  class StreamingHTTPResponse(BaseHTTPResponse):
      __slots__ = (
          "protocol",
          "streaming_fn",
          "status",
          "content_type",
          "headers",
          "_cookies",
      )
  
      def __init__(
          self, streaming_fn, status=200, headers=None, content_type="text/plain"
      ):
          self.content_type = content_type
          self.streaming_fn = streaming_fn
          self.status = status
          self.headers = CIMultiDict(headers or {})
          self._cookies = None
  ```

  