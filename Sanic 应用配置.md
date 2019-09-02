## Sanic 应用配置

​	我们写的web应用可能会很复杂，Sanic提供了简洁的配置而不用写实际代码。 不同环境或安装的设置可能不同。



#### 应用配置基础

​	Sanic 把配置保存在应用对象的`config`属性中，这个配置对象可以使用点号或类似字典那样进行修改:

```
app = Sanic('myapp')
app.config.DB_NAME = 'appdb'
app.config.DB_USER = 'appuser'
```

其实，这个配置对象就是一个字典，可以是`update`方法一次性修改多个值：

```python
db_settings = {
    'DB_HOST': 'localhost',
    'DB_NAME': 'appdb',
    'DB_USER': 'appuser'
}
app.config.update(db_settings)
```

通常的约定是只有UPPERCASE配置参数。 下面描述的用于加载配置的方法仅查找这样的大写参数。



#### 加载配置

Sanic 提供了几种不同的加载配置的方法。

##### 1.从环境变量加载

任何以 `SANIC_` 开头定义的环境变量都会被应用到 Sanic 的配置中。比如，设置环境变量 `SANIC_REQUEST_TIMEOUT`，它就会被应用字典加载并保存为配置变量 `REQUEST_TIMEOUT`。

当然，你可以传递一个不同的前缀给Sanic 应用：

```
app = Sanic(load_env='YRX_')
```

那么，上面的变量就要定义为 `YRX_REQUEST_TIMEOUT`。同样，我们也可以禁止加载环境变量，只需设置 `load_env`为`False`即可：

```
app = Sanic(load_env=False)
```

##### 2.从对象加载

如果有很多配置值并且它们具有合理的默认值，那么将它们放入模块可能会有所帮助：

```
import myapp.default_settings

app = Sanic('myapp')
app.config.from_object(myapp.default_settings)
```

也可以使用类（class）或其它对象来加载配置。

##### 3.从文件加载

通常，我们希望从文件加载配置，而不是把它写到发布的应用程序中。这个需求可以通过 `from_pyfile(/path/to/config_file)` 来实现。但是，这需求程序知道配置文件的路径。但我们可以把配置文件路径定义为环境变量，并告诉 Sanic 应用使用它来找到配置文件：

```
app = Sanic('myapp')
app.config.from_envvar('MYAPP_SETTINGS')
```

然后，我们先定义环境变量 `MYAPP_SETTINGS` 在运行应用程序：

```
$ MYAPP_SETTINGS=/path/to/config_file python3 myapp.py
INFO: Goin' Fast @ http://0.0.0.0:8000
```

配置文件是常规Python文件，为了加载它们而执行。 这允许我们使用任意逻辑来构造正确的配置。 只有大写变量才会添加到配置中。 最常见的配置包括简单的键值对：

```
# config_file
DB_HOST = 'localhost'
DB_NAME = 'appdb'
DB_USER = 'appuser'
```



#### 内置的配置值

Sanic 只预先定义了少数几个配置值，它们可以在创建应用时被覆盖。

| Variable                  | Default   | Description                        |
| ------------------------- | --------- | ---------------------------------- |
| REQUEST_MAX_SIZE          | 100000000 | 请求数据的最大值 (bytes)           |
| REQUEST_BUFFER_QUEUE_SIZE | 100       | 流缓存队列的大小                   |
| REQUEST_TIMEOUT           | 60        | 请求超时时间 (sec)                 |
| RESPONSE_TIMEOUT          | 60        | 响应超时时间 (sec)                 |
| KEEP_ALIVE                | True      | 是否保持长链接                     |
| KEEP_ALIVE_TIMEOUT        | 5         | 保持TCP链接的时间 (sec)            |
| GRACEFUL_SHUTDOWN_TIMEOUT | 15.0      | 等待强制关闭非空闲链接的时间 (sec) |
| ACCESS_LOG                | True      | 是否启用访问日志                   |

##### 不同超时变量的差异

```
REQUEST_TIMEOUT
```

请求超时测量的是，从新打开的TCP连接传递到Sanic后端服务器的时刻，到收到整个HTTP请求的时刻之间的持续时间。 如果所花费的时间超过REQUEST_TIMEOUT值（以秒为单位），则将其视为客户端错误，以便Sanic生成HTTP 408响应并将其发送到客户端。 如果客户端通常会非常缓慢地传递非常大的请求负载或上传请求，请将此参数的值设置得更高。

```
RESPONSE_TIMEOUT
```

响应超时测量的是，Sanic服务器将HTTP请求传递给Sanic App的瞬间与HTTP响应发送到客户端的时间之间的持续时间。 如果所花费的时间超过RESPONSE_TIMEOUT值（以秒为单位），则将其视为服务器错误，以便Sanic生成HTTP 503响应并将其发送到客户端。 如果我们的应用程序可能具有延迟生成响应的长时间运行进程，请将此参数的值设置得更高。

```
KEEP_ALIVE_TIMEOUT
```

`Keep-Alive` 是HTTP 1.1中引入的HTTP功能。发送HTTP请求时，客户端（通常是Web浏览器应用程序）可以设置 Keep-Alive 头，以指示http服务器（Sanic）在发送响应后不关闭TCP连接。这允许客户端重用现有的TCP连接来发送后续HTTP请求，并确保客户端和服务器的网络流量更高效。

`KEEP_ALIVE` 配置变量在Sanic中默认设置为`True`。 如果我们的应用程序中不需要此功能，请将其设置为False以使所有客户端连接在发送响应后立即关闭，而忽略请求中的 Keep-Alive 头。

服务器保持TCP连接打开的时间由服务器本身决定。 在Sanic中，使用 `KEEP_ALIVE_TIMEOUT`值配置该值。 默认情况下，它设置为5秒。这与Apache HTTP服务器的默认设置相同，并且在允许客户端有足够的时间发送新请求和不同时保持打开太多连接之间取得了良好的平衡。 除非我们知道我们的客户端正在使用支持长时间保持打开的TCP连接的浏览器，否则不要超过75秒。

下面是其它 http 服务器的相关默认值以供参考：

```
Apache httpd server default keepalive timeout = 5 seconds
Nginx server default keepalive timeout = 75 seconds
Nginx performance tuning guidelines uses keepalive = 15 seconds
IE (5-9) client hard keepalive limit = 60 seconds
Firefox client hard keepalive limit = 115 seconds
Opera 11 client hard keepalive limit = 120 seconds
Chrome 13+ client keepalive limit > 300+ seconds
```