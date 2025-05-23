# 配置运行与部署

本篇文档将介绍如何配置和运行 UtilMeta 项目，以及在生产环境中的部署

## 服务初始化参数

```python
from utilmeta import UtilMeta

service = UtilMeta(__name__, ...)
```

UtilMeta 服务的第一个参数接收当前模块的名称（`__name__`），此外支持的参数有

* `backend`：传入运行时框架的模块或者名称，如 `backend='flask'`
* `name`：服务的名称，应该反映着服务的领域和业务，之后会用户服务的注册，发现与检测等功能
* `description`：服务的描述
* `production`：是否处于生产状态，默认为 False，在部署的生产环境中应该置为 True，这个参数会影响底层框架的运行配置，比如 `django` 的 `PRODUCTION` 设置，与 flask / starlette / fastapi / sanic 的 `debug` 参数

* `host`：服务运行时监听的主机 IP，默认为 `127.0.0.1`，也可以设置为运行主机的 IP 或者 `0.0.0.0` (公开访问)
* `port`：服务运行时监听的端口号，默认取决于运行时框架，比如 flask 使用的是 `5000`，其他框架一般使用的是 `8000`

* `version`：指定服务当前的版本号，你可以传入一个字符串，如 `'1.0.2'`，也可以传入一个元组，比如 `(0, 1, 0)`，版本号的规范建议遵守 [语义版本标准](https://semver.org/)
* `asynchronous`：强制指定服务是否提供异步接口，默认由运行时框架的特性决定
* `api`：传入 UtilMeta 的根 API 类或它的引用字符串 
* `route`：传入 UtilMeta 的根 API 挂载的路径字符串，默认为 `'/'`，即挂载到根路径
* `auto_reload`: 设为 `True` 可以启用服务在代码更新时的自动重载（不建议用于生产），默认未开启

当你初始化 UtilMeta 后，除了直接导入外，你还可以用这种方式导入当前进程的 UtilMeta 服务实例

```python
from utilmeta import service
```

!!! warning
	一个进程中只能定义一个 UtilMeta 服务实例

### 选择 `backend` 框架

目前 UtilMeta 内置支持的运行时框架有

* `django`
* `flask`
* `starlette`
* `fastapi`
* `sanic`
* `tornado`

!!! tip
	如果你还希望支持更多的运行时框架实现，请在 UtilMeta 框架的 [Issues](https://github.com/utilmeta/utilmeta-py/issues) 中提出

你使用 `backend` 参数指定的框架需要先安装到你的 Python 环境中，然后你可以使用类似如下的方式将包导入后传入 `backend` 参数

=== "django"
	```python hl_lines="7"
	import django
	from utilmeta import UtilMeta
	
	service = UtilMeta(
		__name__, 
		name='demo',
		backend=django
	)
	```
=== "flask"
	```python hl_lines="7"
	import flask
	from utilmeta import UtilMeta
	
	service = UtilMeta(
		__name__, 
		name='demo',
		backend=flask
	)
	```
=== "starlette"
	```python hl_lines="7"
	import starlette
	from utilmeta import UtilMeta
	
	service = UtilMeta(
		__name__, 
		name='demo',
		backend=starlette
	)
	```
=== "sanic"
	```python hl_lines="7"
	import sanic
	from utilmeta import UtilMeta
	
	service = UtilMeta(
		__name__, 
		name='demo',
		backend=sanic
	)
	```
#### 注入自定义应用

一些运行时框架往往会提供开发者一个同名的应用类，比如 `Flask`, `FastAPI`, `Sanic`，其中可以定义一些初始化参数，如果你需要对其中的参数进行配置，则可以把定义出的应用实例传入 `backend` 参数，比如

```python
from fastapi import FastAPI

fastapi_app = FastAPI(debug=False)

service = UtilMeta(
    __name__,
    name='demo',
    backend=fastapi_app,
)
```

!!! tip
	你还可以使用这种方式在 UtilMeta 框架项目中接入你选择的运行时框架的原生接口，详细的用法可以参考 [从现有项目迁移](../migration) 这篇文档

### 异步服务

不同的运行时框架对于异步的支持程度不同，如果没有显式指定 `asynchronous` 参数，服务接口是否为异步取决于各自的特性，如
 
* **Django**：同时支持 WSGI 与 ASGI，但是 `asynchronous` 默认为 False
!!! tip
	对于使用 Django 作为 `backend` 的服务，如果开启 `asynchronous=True` 则会得到一个 ASGI 应用，否则会得到一个 WSGI 应用

* **Flask**：支持 WSGI，处理异步函数需要将其先转化为同步函数，`asynchronous` 默认为 False
* **Sanic**：支持 ASGI，`asynchronous` 默认为 True
* **Tornado**：自行基于 `asyncio` 实现了 HTTP Server，`asynchronous` 默认为 True
* **Starlette/FastAPI**：支持 ASGI，`asynchronous` 默认为 True

如果你希望编写异步（`async def` ）接口，请选择一个默认支持异步的运行时框架，这样可以发挥出你的异步接口的性能，如果你选择的运行时框架默认不启用异步（如 `django` / `flask`），则需要开启  `asynchronous=True` 选项，否则将无法执行其中的异步函数


### 设置服务的访问地址

UtilMeta 服务还有一个 `origin` 参数可以指定这个服务提供公开访问的源地址，比如
`
```python hl_lines="10"
from utilmeta import UtilMeta
import django

service = UtilMeta(
    __name__,
    name='blog',
    backend=django,
    api='service.api.RootAPI',
    route='/api',
    origin='https://blog.mysite.com',
)
```

由于一个请求在抵达后端服务前可能经过很多网关或负载均衡，所以后端服务部署的地址往往不是互联网能直接访问到的地址，服务的 `origin` 参数让我们可以设置服务提供访问的地址

比如例子中的 blog 服务的基准 URL 就是 `https://blog.mysite.com/api`，也会体现在生成的 OpenAPI 文档中

!!! tip
	`origin` 参数提供的地址需要满足 “源（Origin）” 的定义，即包含网络协议，主机，端口号（如果有），但不包含 URL 路径，根路径应该体现在 `route` 参数中

## 服务的方法与钩子

实例化的 UtilMeta 服务还有一些方法或钩子可以使用

### `use(config)` 注入配置

服务实例的 `use` 方法可以用于注入配置，例如
```python
from utilmeta import UtilMeta
from config.env import env

def configure(service: UtilMeta):
    from utilmeta.core.server.backends.django import DjangoSettings
    from utilmeta.core.orm import DatabaseConnections, Database
    from utilmeta.conf.time import Time

    service.use(DjangoSettings(
        apps_package='domain',
        secret_key=env.DJANGO_SECRET_KEY
    ))
    service.use(DatabaseConnections({
        'default': Database(
            name='db',
            engine='sqlite3',
        )
    }))
    service.use(Time(
        time_zone='UTC',
        use_tz=True,
        datetime_format="%Y-%m-%dT%H:%M:%S.%fZ"
    ))
```

UtilMeta 内置的常用配置有

* `utilmeta.core.server.backends.django.DjangoSettings`：配置 Django 项目，如果你的服务以 Django 作为运行时框架，或者需要使用 Django 模型的话就需要使用这个配置项
* `utilmeta.core.orm.DatabaseConnections`：配置数据库连接
* `utilmeta.core.cache.CacheConnections`：配置缓存连接
* `utilmeta.conf.time.Time`：配置服务的时区与接口中的时间格式

!!! warning
	一个类型的配置只能使用 `use` 注入一次


UtilMeta 提供了一个 `DjangoSettings` 配置，能够方便地对 Django 项目和需要使用 Django ORM 的项目进行 Django 配置

### `setup()` 配置的安装

一些服务的配置项需要在服务启动前进行安装与准备，比如对于使用 Django 模型的服务，`setup()` 会调用 `django.setup` 函数完成模型的发现，你需要在调用这个方法之后才能导入你定义的  Django 模型以及依赖这些模型的接口与 Schema 类，比如

```python hl_lines="8"
from utilmeta import UtilMeta
from config.conf import configure 
import django

service = UtilMeta(__name__, name='demo', backend=django)
configure(service)

service.setup()

from user.models import *
```

否则会出现类似如下的错误
```python
django.core.exceptions.ImproperlyConfigured: 
Requested setting INSTALLED_APPS, but settings are not configured, ...	
```

当然对于使用 Django 的项目，最佳实践是使用引用字符串来指定根 API，这样在服务配置文件中就不需要包含对 Django 模型的导入了，例如

=== "main.py"  
	```python hl_lines="8"
	from utilmeta import UtilMeta
	import django
	
	service = UtilMeta(
		__name__,
	    name='demo',
	    backend=django,
		api='service.api.RootAPI',
		route='/api',
	)
	```
=== "service/api.py"  
	```python
	from utilmeta.core import api
	
	class RootAPI(api.API):
	    @api.get
	    def hello(self):
	        return 'world'
	```

### `application()` 获取 WSGI/ASGI 应用

你可以通过调用服务的 `application()` 方法返回它生成的 WSGI / ASGI 应用，比如在 Hello World 示例中

```python hl_lines="18"
from utilmeta import UtilMeta
from utilmeta.core import api
import django

class RootAPI(api.API):
    @api.get
    def hello(self):
        return 'world'

service = UtilMeta(
    __name__,
    name='demo',
    backend=django,
    api=RootAPI,
    route='/api'
)

app = service.application()

if __name__ == '__main__':
    service.run()
```

这个生成的 `app` 的类型取决于服务实例指定的 `backend` ，如

* `flask`: 返回一个 Flask 实例
* `starlette`：返回一个 Starlette 实例
* `fastapi`：返回一个 FastAPI 实例
* `sanic`：返回一个 Sanic 实例
* `django`：默认返回一个 WSGIHandler，如果指定了 `asynchronous=True`，则会返回一个 ASGIHandler
* `tornado`：返回一个 `tornado.web.Application` 实例

如果你使用 uwsgi 和 gunicorn 等 WSGI 服务器部署 API 服务的话，其中都需要指定一个 WSGI 应用，你只需要将对应的配置项设为 `app` 的引用即可，比如

=== "gunicorn.py"
	```python
	wsgi_app = 'server:app'
	```
=== "uwsgi.ini"
	```ini
	[uwsgi]
	module = server.app
	```

!!! warning "使用 `sanic`"
	当你使用 `sanic` 作为运行时框架时，即使不使用 WSGI 服务器，也需要在启动服务的文件中声明 `app = service.application()`，因为  `sanic` 会启动新的进程来处理请求，如果没有 `application()` 对接口的加载，新的进程将检测不到任何路由

### `@on_startup` 启动钩子

你可以使用服务实例的 `@on_startup` 装饰器装饰一个启动钩子函数，在服务进程启动前调用，可以用于进行一些服务的初始化操作，如

```python hl_lines="10"
from utilmeta import UtilMeta
import starlette

service = UtilMeta(
    __name__,
    name='demo',
    backend=starlette,
)

@service.on_startup
async def on_start():
    import asyncio
    print('prepare')
    await asyncio.sleep(0.5)
    print('done')
```

对于支持异步的 `backend` 框架，如 Starlette / FastAPI / Sanic / Tornado，你可以使用异步函数作为启动钩子函数，否则你需要使用同步函数，比如 Django / Flask

### `@on_shutdown` 终止钩子

你可以使用服务实例的 `@on_shutdown` 装饰器装饰一个终止钩子函数，在服务进程结束前调用，可以用于进行一些服务进程的清理操作，如

```python hl_lines="10"
from utilmeta import UtilMeta
import starlette

service = UtilMeta(
    __name__,
    name='demo',
    backend=starlette,
)

@service.on_shutdown
def clean_up():
    # clean up
    print('done!')
```


## 环境变量管理

在实际后端开发中，你往往需要使用很多密钥，比如数据库密码，第三方应用密钥，JWT 密钥等等，这些信息如果硬编码到代码中有泄露的风险，很不安全，并且这些密钥信息在开发，测试和生产环境中也往往有着不同的配置，所以更适合使用环境变量来管理

UtilMeta 提供了一个内置环境变量组件 `utilmeta.conf.Env` 便于你管理这些变量与密钥信息，使用的方式如下

```python
from utilmeta.conf import Env

class ServiceEnvironment(Env):
    PRODUCTION: bool = False
    DJANGO_SECRET_KEY: str = ''
    COOKIE_AGE: int = 7 * 24 * 3600

    # cache -----
    REDIS_DB: int = 0
    REDIS_PORT: int = 6379
    REDIS_PASSWORD: str

    # databases ---------
    DB_HOST: str = ''
    DB_USER: str  
    DB_PASSWORD: str
    DB_PORT: int = 5432

env = ServiceEnvironment(sys_env='DEMO_')
```

通过继承 `utilmeta.conf.Env` 类，你可以在其中声明服务需要的环境变量，环境变量在解析时会忽略大小写，但我们建议使用大写的方式与其他属性进行区分，你可以为每个变量声明类型与默认值，`Env` 会将变量转化为你声明的类型，在获取不到对应的变量时使用你指定的默认值

`Env` 子类在实例化时可以指定环境变量的来源，如

### `sys_env` 系统环境变量

你可以指定一个前缀，从而会在系统环境变量中拾取 **前缀+名称** 的变量，比如例子中指定的前缀为 `'DEMO_'`，那么系统环境变量中的 `DEMO_DB_PASSWORD` 将会被解析为环境变量数据中的 `DB_PASSWORD` 属性

如果你不需要指定任何前缀，可以使用 `sys_env=True` 
### `file` 配置文件

除了从系统环境变量中拾取外，你还可以使用 `file` 参数指定一个 JSON 或 INI 格式的配置文件

```python
from utilmeta.conf import Env

class ServiceEnvironment(Env):
    pass

env = ServiceEnvironment(file='/path/to/config.json')
```

在文件中声明对应的环境变量，如

=== "json"
	```json
	{
	    "DB_USER": "my_user",
	    "DB_PASSWORD": "my_password"
	}
	```
=== "ini"
	```ini
	DB_USER=my_user
	DB_PASSWORD=my_password
	```

!!! warning
	如果你使用的是配置文件，请把配置文件放在项目目录之外，或者在 `.gitignore` 中把它从版本管理中排除

## 常用配置

### `DjangoSettings` 配置

UtilMeta 提供了一个 `DjangoSettings` 配置，可以为所有使用 Django 作为 `backend` 的项目和使用 django ORM 的项目提供声明式的 django 配置， `DjangoSettings` 的常用配置参数如下

* `secret_key`：指定 Django 项目的密钥，推荐在环境变量中生成一个长的随机密钥
* `apps`：可以传入一个包列表用于直接指定 Django 的 `INSTALLED_APPS`
* `apps_package`：如果你的 django app 都定义在某个文件夹中，例如：
```  hl_lines="3"
/project
	/config
	/domain
		/app1
			/migrations
			models.py
		/app2
			/migrations
			models.py
```

	 你可以指定 `apps_package='domain'` 来便捷地识别 `/domain`  中所有的 app，目前的识别方式是检测其中是否包括 `migrations` 文件夹（即 django 用于保存 app 中数据模型变更记录的文件夹）
	 
	 如果有多个这样的包你还可以传入一个包的相对路径的列表，如 `apps_package=['domain.apps', 'vendors']`

* `middleware`：可以传入一个 django 中间件的列表
* `module_name`：指定 django 的配置文件引用
* `extra`：可以传入一个字典指定额外的 Django 配置


另外，如果你没有为 `DjangoSettings` 指定 `module_name`，它将默认使用 UtilMeta 服务所在的模块作为配置，所以你也可以在服务 `setup()` **之前** 直接在这个文件中声明 django 配置，用法与原生 django 配置一样，例如

```python
from utilmeta import UtilMeta
from config.conf import configure 
import django

DATA_UPLOAD_MAX_NUMBER_FILES = 1000

service = UtilMeta(__name__, name='demo', backend=django)
configure(service)

service.setup()
```

!!! warning
	`service.setup()` 也会同步触发 django 的 `setup()` 从而加载所有 django 设置，所以在它之后进行的 django 配置无法被加载

### `DatabaseConnections` 数据库配置

在 UtilMeta 中，`DatabaseConnections` 用于配置数据库连接，它接受一个字典参数，字典的键是连接的名称，值是一个 `Database` 实例，用于配置数据库的地址和连接信息，在 UtilMeta 的 ORM 中，如果没有显式指定，会默认使用名称为 `'default'` 的数据库连接

```python
from utilmeta.core.orm import DatabaseConnections, Database
from config.env import env

service.use(DatabaseConnections({
	'default': Database(
		name='blog',
		engine='postgresql',
		host=env.DB_HOST,
		user=env.DB_USER,
		password=env.DB_PASSWORD,
		port=env.DB_PORT,
	)
}))
```

!!! note
	如果你使用过 Django，应该对这种配置方式很属性，在 UtilMeta 中使用 django ORM 时，`DatabaseConnections` 也会生成 django 的 `DATABASES` 配置

你可以在 [ORM 配置数据库连接](../schema-query/#orm_2) 中查看详细的用法

### `CacheConnections` 缓存配置

`CacheConnections` 用于配置缓存连接，语法与  `DatabaseConnections` 类似，连接字典的值指定一个缓存实例，其中可以配置缓存的地址与连接信息，例如

```python
from utilmeta.core.cache import CacheConnections, Cache
from utilmeta.core.cache.backends.redis import RedisCache
from config.env import env

service.use(CacheConnections({
	'default': RedisCache(
		port=env.REDIS_PORT,
		db=env.REDIS_DB,
		password=env.REDIS_PASSWORD
	),
	'fallback': Cache(engine='django.core.cache.backends.locmem.LocMemCache')
}))
```

目前 UtilMeta 支持两种缓存配置

* **DjangoCache**：默认的缓存配置，将会使用 Django 的缓存进行实现，其中 `engine` 参数可以传入 Django 的缓存类
* **RedisCache**：Redis 缓存配置，同时支持同步与异步用法，同步的用法由 Django 实现，异步的用法使用 `aioredis` 实现

### `Time` 时间与时区配置

`Time` 用于配置项目的时间与时区设置，影响 API 接口的时间序列化与数据库中的时间存储

```python
from utilmeta.conf import Time

service.use(Time(
	time_zone='UTC',
	use_tz=True,
	datetime_format="%Y-%m-%dT%H:%M:%SZ"
))
```

其中的参数包括

* `time_zone`：指定时间的时区，默认为本机的时区，可以使用 `'UTC'` 来指定 UTC 时区
* `use_tz`：是否为项目的所有 datetime 时间开启时区（timezone aware），默认为 True，会同步 Django 的 `USE_TZ` 配置
* `date_format`：指定 `date` 类型的序列化格式，默认为 `%Y-%m-%d`
* `time_format`：指定 `time` 类型的序列化格式，默认为 `%H:%M:%S`
* `datetime_format`：指定 `datetime` 类型的序列化格式，默认为 `%Y-%m-%d %H:%M:%S`


### `Preference` 偏好参数配置

你可以使用 `Preference` 来配置 UtilMeta 框架的特性中的参数

```python
from utilmeta.conf import Preference

service.use(Preference(
	client_default_request_backend=httpx,
	default_aborted_response_status=500,
	orm_raise_non_exists_required_field=True
))
```

其中常用的参数有：

* `strict_root_route`: 是否严格校验根 API 路由，默认为 False, 例如当 API 的根路由为 `/api` 时，`/api/user` 与 `/user` 均会访问到 `/user` 所在的根 API 函数，你可以设为 True，这样只有 `/api/user` （即以根路由开头的路径）才会被处理，其他路径都会返回 404 响应
* `api_default_strict_response`：API 函数是否对生成的响应开启严格结果校验，默认为 None，如果设为 True，则 API 函数生成的响应默认都会对响应数据的类型和结构进行校验，校验不通过会直接抛出错误
* `client_default_request_backend`：指定默认的 `Client` 类底层请求库，目前 UtilMeta 支持 `httpx`, `aiohttp`, `requests` 与 `urllib`，默认为 `urllib`
* `default_aborted_response_status`：如果客户端请求失败无法得到响应时默认生成的响应码，默认为 503
* `default_timeout_response_status`：如果客户端请求超时默认生成的响应码，默认为 504
* `orm_default_query_distinct`：`orm.Query` 查询是否默认进行 `DISTINCT` ，默认不开启，需要在 `orm.Query` 类中指定 `__distinct__ = True` 才会进行去重处理
* `orm_default_gather_async_fields`：在 `orm.Schema` 的异步查询方法中，是否对无关联的关系字段查询进行 `asyncio.gather` 聚合，默认是 False
* `orm_raise_non_exists_required_field`：在 `orm.Schema`中的必传的字段没有在模型中检测到有对应名称的字段时，是否抛出错误，默认为 False，会给出 warnings 提示
* `orm_schema_query_max_depth`：使用 `orm.Schema` 进行关系查询的最大查询深度，默认是 100，虽然关系查询有自动检测避免无限循环嵌套的机制，这个参数也可以作为兜底策略应对其他可能的情况增强关系查询的鲁棒性
* `dependencies_auto_install_disabled`:  是否对运行服务或执行命令所需的未安装依赖库 **禁用** 自动安装，默认为 False，UtilMeta 检测到的未安装依赖会自动运行 `pip install` 进行安装，但是如果你的环境可能导致安装多次失败重试的话，你可以考虑开启这个参数对自动安装进行禁用，避免依赖安装占用大量的进程资源

## 运行服务

UtilMeta 服务实例提供了一个 `run()` 方法用于运行服务，我们已经看到过它的用法了
```python hl_lines="21"
from utilmeta import UtilMeta
from utilmeta.core import api
import django

class RootAPI(api.API):
    @api.get
    def hello(self):
        return 'world'

service = UtilMeta(
    __name__,
    name='demo',
    backend=django,
    api=RootAPI,
    route='/api'
)

app = service.application()

if __name__ == '__main__':
    service.run()
```

我们一般在 ` __name__ == '__main__'` 的版块内调用 `service.run()`，这样你就可以通过使用 python 执行这一文件从而运行服务，例如

```
python server.py
```

 `run()` 方法会根据服务实例的 `backend` 执行对应的运行策略，比如

* **Django**：通过调用 `runserver` 命令运行服务，不建议在生产环境中使用
* **Flask**：直接调用 Flask 应用的 `run()` 方法运行服务 
* **Starlette/FastAPI**：将使用 `uvicorn` 运行服务
* **Sanic**：直接调用 Sanic 应用的 `run()` 方法运行服务 
* **Tornado**：使用 `asyncio.run` 运行服务

另外如果你在 `meta.ini` 中使用 `main` 指定了包含 `service.run()` 的入口文件

```ini
[service]
main = server
```

你也可以通过运行
```
meta run
```

来启动服务，在 Linux 系统中，还可以加上 `-d` 参数指定为 nohup 运行，不被当前终端的关闭影响
### 自定义运行

对于 Flask, FastAPI, Sanic 等框架，你可以通过 `service.application()` 获取到生成的 Flask, FastAPI Sanic 应用，所以你也可以直接调用它们的 `run()` 方法从而传入对应框架支持的参数

```python hl_lines="21"
from utilmeta import UtilMeta
from utilmeta.core import api
import flask

class RootAPI(api.API):
    @api.get
    def hello(self):
        return 'world'

service = UtilMeta(
    __name__,
    name='demo',
    backend=flask,
    api=RootAPI,
    route='/api'
)

app = service.application()

if __name__ == '__main__':
    app.run(
        debug=False,
        port=8000
    )
```

## 部署服务

通过 `run()` 方法我们可以在调试时快速运行一个可访问服务实例，但是在实际部署时，我们往往需要额外的配置从而使得服务稳定地运行在我们期望的地址，并且充分发挥运行环境的性能

Python 开发的 API 服务常用的一种部署架构为

![ BMI API Doc ](https://utilmeta.com/assets/image/server-deploy.png)

将你开发的 API 服务使用 uwsgi / gunicorn 等 WSGI 服务器或 Daphne 等 ASGI 服务器运行，它们会更高效地进行多进程（worker）管理和请求分发

然后使用一个反向代理服务（如 Nginx）将 API 的根路由解析到 WSGI/ASGI 服务器提供的端口，同时可以代理图片或 HTML 等静态文件，然后根据需要对外提供 80 端口的 HTTP 服务或 443 端口的 HTTPS 服务

但是由于不同的运行时框架的特性不同，我们建议根据你使用的 `backend` 选择部署策略

* **Django**：默认生成  WSGI 应用，可以使用 uWSGI / Gunicorn 部署，如果是异步服务生成的 ASGI 应用，也可以使用 Daphne ASGI 服务器部署
* **Flask**：生成  WSGI 应用，可以使用 uWSGI / Gunicorn 部署
* **Sanic**：自身就是一个多进程服务应用，可以直接使用 Nginx 代理
* **Starlette/FastAPI**：可以使用搭载了 Uvicorn worker 的 Gunicorn 部署
* **Tornado**：自身就是一个异步服务应用，可以直接使用 Nginx 代理

!!! tip
	有些框架如 Sanic 和 Tornado，本身就可以直接运行可靠的高性能服务，所以无需使用 WSGI / ASGI 服务器，可以直接运行并接入 Nginx 代理

### uWSGI

使用 uWSGI 之前需要先安装
```
pip install uwsgi
```

之后可以使用 `ini` 文件编写 uwsgi 的配置文件，如
```ini
[uwsgi]
module = server:app
chdir = /path/to/your/project
daemonize = /path/to/your/log
workers = 5
socket=127.0.0.1:8000
```

其中重要的参数包括

* `chdir`：指定服务的运行目录，一般是项目的根目录
* `module`：指定你服务的 WSGI 应用，相对服务的运行目录，假设你的 WSGI 应用位于 `server.py` 中的 `app` 属性，就可以使用 `server:app` 定义
* `daemonize`：设置日志文件地址
* `workers`：设置服务运行的进程数量，一般可以设为服务器的 CPU 数 x  2 + 1
* `socket`：uwsgi 服务监听的 socket 地址，用于与前置的 Nginx 等代理服务器通信

uwsgi 服务器的运行命令如下

```
uwsgi --ini /path/to/your/uwsgi.ini
```

### Gunicorn

使用 Gunicorn 之前需要先安装
```
pip install gunicorn
```

之后可以直接使用 Python 文件编写 Gunicorn 的配置文件，如

```python
wsgi_app = 'server:app'
bind = '127.0.0.1:8000'
workers = 5
accesslog = '/path/to/your/log/access.log'
errorlog = '/path/to/your/log/error.log'
```

其中主要的参数有

* `wsgi_app`：指定你服务的 WSGI 应用的引用，假设你的 WSGI 应用位于 `server.py` 中的 `app` 属性，就可以使用 `server:app` 定义
* `bind`：服务监听的地址，用于与前置的 Nginx 等代理服务器通信
* `workers`：设置服务运行的进程数量，一般可以设为服务器的 CPU 数 x  2 + 1
* `accesslog`：设置服务的访问日志地址
* `errorlog`：设置服务的运行于错误日志地址

此外你还可以使用 `worder_class` 属性指定工作进程的实现，可以根据接口的类型优化运行效率，如

* `'uvicorn.workers.UvicornWorker'`：适合异步接口的，如 Starlette / FastAPI，需要先安装 `uvicorn`
* `'gevent'`：适合同步接口，如 Django / Flask，会使用 `gevent` 库中的协程（green thread）提高接口的并发性能，需要先安装 `gevent`

gunicorn 服务器的运行命令如下

```
gunicorn -c /path/to/gunicorn.py
```

### Nginx

对于使用 uWSGI 的 API 服务，假设运行并监听 8000 端口，Nginx 的配置大致为

```nginx
server{
    listen 80;
    server_name example.com;
    charset utf-8;
    include /etc/nginx/proxy_params;
    
    location /api/{
        include /etc/nginx/uwsgi_params;
        uwsgi_pass 127.0.0.1:8000;
    }
}
```

对于 Gunicorn 或者其他直接运行的服务，假设运行在 8000 端口，Nginx 的配置大致为

```nginx
server{
    listen 80;
    server_name example.com;
    charset utf-8;
    include /etc/nginx/proxy_params;
    
    location /api/{
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header REMOTE_ADDR $remote_addr;
    }
}
```

!!! tip
	为了使得 API 服务能够获取到请求的来源 IP（而不是反向代理服务的 IP），我们需要使用 `proxy_set_header` 将对应的请求头参数传递过去

你应该把配置好的 nginx 文件放置在 ·`/etc/nginx/sites-enabled/` 中从而启用对应的配置，可以使用如下命令检测配置是否有问题

```
nginx -t
```

如果没有问题，就可以使用如下命令重启 nginx 服务使得更新的配置生效

```
nginx -s reload
```

## 服务观测与运维管理

UtilMeta 框架内置了一个 API 服务管理系统，可以方便地观测与管理对本地与线上的 API 服务，提供了数据，接口，日志，监控，测试，报警等一系列运维管理功能，在 [Operations 运维管理系统配置](../ops) 中有这个系统的详细介绍与配置方式

你也可以直接进入 [UtilMeta API 服务管理平台](https://ops.utilmeta.com) 进行体验
