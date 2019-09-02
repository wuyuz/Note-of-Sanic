## Sanic 之 Cookies、版本控制



#### Cookies:

​	cookie是保存在用户浏览器中的数据片段。SANIC可以读写cookie，cookie存储为键值对。写Web应用（网站等）经常会用到Cookies。Sanic可以读写Cookies，并以key-value（键值对）的方式存储。警告：因为Cookies很容易被客户端修改，所以不能把登录信息直接保存到cookies里面，而是经过加密后再放到cookies。后面我将写一篇用于Sanic加密cookies的文章来介绍在Sanic中使用加密cookies的方法



#### 读写 Cookies

- 用户的cookie可以通过 `Request` 对象的 `cookies` 字典。

  ```python
  from sanic.response import json,text
  
  app = Sanic()
  
  # 写cookie
  @app.route("/set_cookie")
  async def set_cookie(request):
      response = text("There's a cookie up in this response")
      response.cookies['test'] = 'It worked!'
      # 设置了domain的话，只能响应的域名下才能访问，否则读取不到
      # response.cookies['test']['domain'] = '.gotta-go-fast.com'
      
      # 名为test的cookie10秒后失效
      response.cookies["test"]["max-age"] = 10
      response.cookies['test']['httponly'] = True
      return response
  
  #读cookie
  @app.route("/get_cookie")
  async def test(request):
      # 在request中读取cookie
      test_cookie = request.cookies.get('test')
      return text("Test cookie set to: {}".format(test_cookie))
  
  if __name__ == '__main__':
      app.run(host="0.0.0.0", port=8000,debug=True)
  ```

- Cookie的相关设置

  ```
  Cookie可以像字典一样设置，并且具有如下参数：
      expires：过期时间，Cookie在客户端浏览器上过期的时间
      path：此Cookie使用的URL的子集。默认为/
      comment：评论(元数据)
      domain：Cookie的有效域
      max-age：Cookie的活跃秒数
      secure：指定Cookie是否仅通过HTTPS发送
      httponly：指定Cookie是否不能被Javascript读取
  ```


- 删除cookies

  ```python
  @app.route("/cookie")
  async def test(request):
      response = text("Time to eat some cookies muahaha")
      # 删除一个不存在的cookie，就会把它的Max-age设置为0
      del response.cookies['kill_me']
  
      # 该cookie将在5秒内销毁。
      response.cookies['short_life'] = 'Glad to be here'
      response.cookies['short_life']['max-age'] = 5
  
      # 还是删除一个不存在的cookies
      del response.cookies['favorite_color']
      # This cookie will remain unchanged
      response.cookies['favorite_color'] = 'blue'
      response.cookies['favorite_color'] = 'pink'
  
      # 从cookies字典中移除`favorite_color`键
      del response.cookies['favorite_color']
      return response
  
  ```

  

Cookies 不为人知的部分，读取cookies用户（客户端）发来的数据都保存在`Request`对象里面，其中也包含cookies。通过该对象的`cookies`字典即可访问用户的cookies。在Sanic的文档中并没有列举出所有的cookie的设置方式，只举例说Cookie在text的情况下的设置方式，其实cookie的设置需要借助与Httpresponse对象，也就是多种响应对象的基础上

- 下列，是我登陆设置重定向时，设置cookie

  ```python
  def login(request):
      if request.method == 'GET':
          form = LoginForm()
          return template('login.html', form=form,request=request)
      else:
          form = LoginForm(formdata=request.form)
          if form.validate():
              dic = request.form
              user = session.query(models.Users).filter_by(**dic).first()
              if user:
                  #cookie必须封装在响应对象中
                  response = redirect('/index')
                  response.cookies['text']='isxxx'
                  print(response)
                  return response
              else:
                  return redirect('/register')
          else:
              print(form.errors)
          return template('login.html', form=form,request=request)
  
  ```

  

#### 响应模块

- Sanic自带了html，专用于返回html页面，但是并不能满足我们的要求，我们需要自定义jinja2模块

  ```python
  from sanic import Blueprint
  from jinja2 import Environment, PackageLoader, select_autoescape
  
  user = Blueprint('users')
  user.static('/static', './static')
  
  #sanic 的html自带了，但是无法满足要求渲染，我们需要自己定义template
  env = Environment(
      #'serv.users'表示所在文件路径位置
      loader=PackageLoader('serv.users', '../templates'),
      autoescape=select_autoescape(['html', 'xml', 'tpl']))
  
  def template(tpl, **kwargs):
      template = env.get_template(tpl)
      return html(template.render(kwargs))
  
  @user.route('/index')
  def index(request):
      return template('index.html')
  ```

  

#### 版本控制

​	Sanic实现了简洁的版本控制，通过传递关键词参数`version`给路由装饰器或blueprint初始化方法就可以实现。这将会在url前面添加形似`v{version}`的url前缀，其中`{version}`就是版本号。

- 每个路由的版本控制，给路由装饰器传递版本号实现路由级的版本控制。

  ```python
  from sanic import Sanic
  from sanic.response import text
  
  app = Sanic()
  
  @app.route("/text", version=1)
  async def test(request):
      return text('Hi, Version 1\n')
  
  @app.route("/text", version=2)
  async def test(request):
      return text('Hi, Version 2\n')
  
  
  if __name__ == '__main__':
      app.run(host='127.0.0.1', port=8888, debug=True)
      
   # localhost:8888/v1/text
   # localhost:8888/v2/text
  ```

- 初始化blueprint时传递版本号，可以应用到该blueprint下的所有路由。

  ```python
  from sanic import response
  from sanic.blueprints import Blueprint
  
  bp = Blueprint('test', version=1)
  
  @bp.route('/html')
  def handle_request(request):
      return response.html('<p>Hello world!</p>')
  
  #localhost:8888/v1/text
  ```

  

#### 静态文件

​	我们在写web app（网站）的时候会用到很多静态文件，比如css，JavaScript，图片等，这些文件及其文件夹可以通过`app.static()`方法注册，从而被访问到。该方法有两个必需参数，节点URL和文件名。

- 再次讲解

  ```python
  from sanic import Sanic
  from sanic.blueprints import Blueprint
  
  app = Sanic(__name__)
  
  # 提供文件夹`static`里面的文件到URL `/static`的访问。
  app.static('/static', './static')
  # 使用`url_for`创建URL时，默认为'static'的`name`可被省略。
  app.url_for('static', filename='file.txt') == '/static/file.txt'
  app.url_for('static', name='static', filename='file.txt') == '/static/file.txt'
  
  # 通过URL /the_best.png访问文件 /home/ubuntu/test.png
  app.static('/the_best.png', '/home/ubuntu/test.png', name='best_png')
  
  # 通过url_for创建静态文件URL
  # 如果没有定义name 和 filename 参数时可以省略它们。
  # 如果上面没有定义name='best_png'，则下面的url_for可省略name参数
  app.url_for('static', name='best_png') == '/the_best.png'
  app.url_for('static', name='best_png', filename='any') == '/the_best.png'
  
  # 需要为其它静态文件定义`name`
  app.static('/another.png', '/home/ubuntu/another.png', name='another')
  app.url_for('static', name='another') == '/another.png'
  app.url_for('static', name='another', filename='any') == '/another.png'
  
  # 同样, 也可以对blueprint使用`static`
  bp = Blueprint('bp', url_prefix='/bp')
  bp.static('/static', './static')
  bp.static('/the_best.png', '/home/ubuntu/test.png', name='best_png')
  app.blueprint(bp)
  
  app.url_for('static', name='bp.static', filename='file.txt') == '/bp/static/file.txt'
  app.url_for('static', name='bp.best_png') == '/bp/test_best.png'
  
  app.run(host="0.0.0.0", port=8000)
  ```

  流化大文件：有时候需要Sanic提供大文件（比如，视频，图片等）的访问，可以选择使用**流文件**替代直接下载。比如：

  ```python
  from sanic import Sanic
  
  app = Sanic(__name__)
  
  app.static('/large_video.mp4', '/home/ubuntu/large_video.mp4', stream_large_files=True)
  ```

  当`stream_large_files`为`True`时，Sanic将会使用`file_stream()`替代`file()`来提供静态文件访问。它默认的块大小是**1KB**，当然你可以根据需要修改块大小。比如

  ```python
  from sanic import Sanic
  
  app = Sanic(__name__)
  
  chunk_size = 1024 * 1024 * 8 # 设置块大小为 8KB
  app.static('/large_video.mp4', '/home/ubuntu/large_video.mp4', stream_large_files=chunk_size)
  ```

  



