# Handle request parameters

API requests can carry parameter information in a number of ways, such as

* Path parameters
* Query parameters
* Request Body (JSON/Form/File, etc.)
* Request headers (including cookies)

We will show you how to handle the various parameters of the request in UtilMeta.

## Path parameters

It is a common way to pass data in the request URL path. For example, by `GET/article/3` getting the article data with ID 3, the parameter of ID is provided in the URL path. The way to declare the path parameter in UtilMeta is as follows
```python
from utilmeta.core import api

class RootAPI(api.API):
	@api.get('article/{id}')
	def get_article(self, id: int):
		return {"id": id}
```

We pass the template string of the path in `@api` the first parameter of the decorator, as in `'article/{id}'` the example, use curly braces to define the path parameter, and declare a parameter of the same name in the function to receive, and can declare the expected type and rule.

Defining multiple path parameters is similar in usage
```python
from utilmeta.core import api
import utype
from typing import Literal

class RootAPI(api.API):
	@api.get('doc/{lang}/{page}')
	def get_doc(self, 
	           lang: Literal['en', 'zh'], 
	           page: int = utype.Param(1, ge=1)):
		return {"lang": lang, "page": page}
```
In this example, we declare two path parameters:

*  `lang` Can only take values in `'en'` and `'zh'`
*  `page` is a parameter greater than or equal to 1 and defaults to 1
 
Parameters with default values are passed in directly if they are not provided in the path, so we get the output when we request it `GET/doc/en`.
```json
{"lang": "en", "page": 1}
```

!!! tip

If the request parameter does not meet the declared rules or cannot be converted to the corresponding type, you will get `400 BadRequest` an error response, such as

*  `GET/doc/fr/3`: `lang` parameter does not have a value in `'en'` and `'zh'`
*  `GET/doc/en/0`: `page` The parameter does not meet the rule of greater than or equal to one

!!! warning

### Path regex

Sometimes we need the path parameter to satisfy certain rules, and we can easily do this by declaring a regular expression, which is used as follows
```python
from utilmeta.core import api, request

class RootAPI(API):
    @api.get('item/{code}')
    def get_item(self, code: str = request.PathParam(regex='[a-z]{1,9}')):
        return code
```

Some common path parameter configurations are built in the UtilMeta `request` module.

* The `request.PathParam` default path argument rule, that is `'[^/]+'`, matches all characters except the underscore of the path
*  `request.FilePathParam`: Matches all strings, including the path underscore. Commonly used when the path parameter needs to pass the file path, URL path, etc.
*  `request.SlugPathParam` Matches a string such as `how-to-guide` this consisting of an alphabetic data line to be used in the URL encoding of the article.

Here is an example.
```python
from utilmeta.core import api, request

class RootAPI(API):
    @api.get('file/{path}')
    def get_file(self, path: str = request.FilePathParam):
        return open(f'/tmp/{path}', 'r')
```

In this example, the `path` parameter gets `'path/to/README.md'` this path when we request `GET/file/path/to/README.md` it.

### Declare the request path

The examples above show how the UtilMeta declaration uses a template string to declare the request path, but this is not the only way.

In UtilMeta, the declaration rule of the API interface request path is

* Pass the path string in `@api` the decorator as a template for the interface request path.
* When there is no path string, the name of the function is used as the requested path
* When the name of a function is an HTTP verb, its path is automatically set to `'/'` and cannot be overridden

!!! Tip “API interface functions”

The following examples cover the above cases and provide a clear illustration of the path declaration rule
```python
from utilmeta.core import api

@api.route('/article')
class ArticleAPI(api.API):
    @api.get
    def feed(self): pass
    # GET /article/feed
    
    @api.get('{slug}')
    def get_article(self, slug: str): pass
	# GET /article/{slug}
	
    def get(self, id: int): pass
    # GET /article?id=<ID>
```

!!! Tip “Path match priority”

## Query parameters

Using query parameters to pass in key-value pairs is a very common way to pass parameters, for example, by `GET/article?id=3` getting the article data with ID 3

The way to query the parameter declaration is very simple, which can be defined directly in the function, such as
```python
from utilmeta.utils import *

class RootAPI(API):
	@api.get
	def doc(self, 
	        lang: Literal['en', 'zh'], 
	        page: int = utype.Param(1, ge=1)):
		return {"lang": lang, "page": page}
```

We get output when we ask `GET/doc?lang=en`.
```json
{"lang": "en", "page": 1}
```

!!! tip

### Parameter alias
If the parameter name cannot be represented as a Python variable (such as a syntax keyword or contains special symbols), you can use `utype.Param` the parameter of the `alias` component to specify the expected name of the field, such as
```python
from utilmeta.core import api
import utype

class RootAPI(API):
    @api.get
    def doc(self,
            cls_name: str = utype.Param(alias='class'),
            page: int = utype.Param(1, alias='@page')
            ):
        return {cls_name: page}
```

When you visit `GET/api/doc?class=tech&@page=3`, you will get

### Use the Schema class

In addition to declaring query parameters in a function, you can also define all query parameters as a Schema class for better combination and reuse. The usage is as follows
```python
from utilmeta.core import api, request
import utype
from typing import Literal

class QuerySchema(utype.Schema):
    lang: Literal['en', 'zh']
    page: int = utype.Field(ge=1, default=1)
    
class RootAPI(API):
    @api.get
    def doc(self, query: QuerySchema = request.Query):
        return {"lang": query.lang, "page": query.page}
```

In the example, we defined the query parameters `lang` `page` in `QuerySchema` and then injected them into the API function using `query: QuerySchema = request.Query`.

In this way, you can easily reuse query parameters between interfaces using class inheritance and composition.

!!! warning

## Request body

Request body data is often used to pass data such as objects, forms, or files in POST/PUT/PATCH methods

Typically in UtilMeta, you can use the Schema class to declare request body data in JSON or form format, using
```python
from utilmeta.core import api, request
import utype

class LoginSchema(utype.Schema):
	username: str
	password: str
	remember: bool = False

class UserAPI(api.API):
    @api.post
    def login(self, data: LoginSchema = request.Body):
		pass
```
We declare a Schema class called LoginSchema as a type hint for the request body parameter, and `Request.Body` mark the parameter as a request body parameter with a default value for the parameter.

When you use a Schema class to declare the request body, the interface has the ability to handle JSOM/XML and form data, for example, you can pass such a JSON request body.
```json
{
	"username": "alice",
	"password": "123abc",
	"remember": true
}
```

You can also use `application/x-www-form-urlencode` the request body in a format similar in syntax to query parameters, such as
```
username=alice&password=123abc&remember=true
```

*  `request.Body`: The underlying request body type, as long as the data in the request body can be resolved to the declared type.
*  `request.Json`: request body Content-Type needs to be
*  `request.Form`: request body Content-Type needs to be or

### List data
Some scenarios, such as batch creation and update, need to upload the request body data of list type, and the declaration method is to add `List[]` it outside the corresponding Schema class, such as
```python
from utilmeta.core import api, orm, request
from .models import User

class UserAPI(api.API):
	class UserSchema(orm.Schema[User]):
		username: str = utype.Field(regex='[a-zA-Z0-9]{3,20}')
	    password: str = utype.Field(min_length=6, max_length=20)
    
    @api.post
    def batch_create(self, users: List[UserSchema] = request.Body):
		for user in users:
			user.save()
```

If the client needs to pass list data, it needs to use the request body of JSON ( `application/json`) type. If the client only passes a JSON object or form, it will be automatically converted to a list with only this element.

After defining the interface in the example, you can request `POST/api/user/batch` and pass the body of the request.
```json
[{
	"username": "alice",
	"password": "123abc"
}, {
	"username": "bob",
	"password": "XYZ789"
}]
```

### Handle file uploads

If you need to support file upload, you just need to declare the type hint of the file field as a file. The usage is as follows
```python
from utilmeta.core import api, request, file
import utype

class FileAPI(api.API):
	class AvatarData(utype.Schema):
		user_id: int
		avatar: file.Image = utype.Field(max_length=10 * 1024 ** 2)
	
	@api.post
	def avatar(self, data: AvatarData = request.Body):
		pass
```

Several common file types are provided in the `utilmeta.core.file` file that you can use to declare file parameters.

*  `File`: Receive files of any type
*  `Image`: Receive picture files ( `image/*`)
*  `Audio`: Receive audio files ( `audio/*`)
*  `Video`: Receive video files ( `video/*`)

In addition, you can use the `max_length` rule parameter to limit the size of the file. In the example, we `avatar` only accept files below 10 M.

!!! tip

If you need to support uploading multiple files, just add the type declaration `List[]` for the file parameter, as shown in
```python
from utilmeta.core import api, request, file
import utype
from typing import List

class FileAPI(api.API):
	class FilesData(utype.Schema):
        name: str
        files: List[file.File] = utype.Field(max_length=10)

    @api.post
    def upload(self, data: FilesData = request.Body):
        for i, f in enumerate(data.files):
            f.save(f'/data/{data.name}-{i}')
```

#### Upload the file separately
If you want the client to use the entire binary file directly as the request body, instead of using the form nested in the form, you only need to specify the file parameter as `request.Body`, such as

```python
from utilmeta.core import api, request, file
import utype

class FileAPI(api.API):
    @api.post
    def image(self, image: file.Image = request.Body) -> str:
	    name = str(int(self.request.time.timestamp() * 1000)) + '.png'
        image.save(path='/data/image', name=name)
```

### Request body parameter

In addition to supporting the declaration of a complete request body Schema, you can also use `request.BodyParam` a separate declaration of fields in the request body.
```python
from utilmeta.core import api, request, file

class FileAPI(api.API):
	@api.post
	def upload(self, name: str = request.BodyParam,
	           file: file.File = request.BodyParam):
	    file.save(path='/data/files', name=name)
```

### Strings and other types of data
If you want the interface to accept a request body in the form of a string, you only need to specify the corresponding parameter as the request body, as shown in
```python
from utilmeta.core import api, request

class ArticleAPI(api.API): 
	@api.post
	def content(self, html: str = request.Body(
		max_length=10000,
		content_type='text/html'
	)):
		pass
```
In this function, we use `html` to specify and receive `'text/html'` the type of request body, and limit the maximum length of the upload text of the request body to 10000.

## Request header parameters

API requests usually carry request headers (HTTP Headers) to pass the meta-information of the request, such as permission credentials, negotiation cache, session cookies, etc. In addition to the default request headers, you can also customize the request headers. There are two kinds of attributes to declare the request headers.

*  `request.HeaderParam`: Declare a single request header parameter
*  `request.Headers`: Declare the complete request header Schema

```python
from utilmeta.core import api, request
import utype

class RootAPI(api.API):
	class HeaderSchema(utype.Schema):
		auth_token: str = utype.Field(length=12, alias='X-Auth-Token')
		meta: dict = utype.Field(alias='X-Meta-Data')
	
	@api.post
	def operation(self, headers: HeaderSchema = request.Headers):
		return [headers.auth_token, headers.meta]
```

Generally speaking, the customized request header begins with `X-` and uses hyphens `-` to connect words. We use `alias` parameters to specify the name of the request body parameter. For example, the request header parameter declared in the example is

*  `auth_token`: Target request header name is `X-Auth-Token` a string of length 12
*  `meta`: The name of the target request header is `X-Meta-Data`, an object that can be resolved to a dictionary.

!!! tip

When the request
```http
POST /api/operation HTTP/1.1

X-Auth-Token: OZ3tPOl6
X-Meta-Data: {"version":1.2}
```
You’ll get `["OZ3tPOl6", {"version": 1.2}]` a response.

!!! note

### General parameters

The common situation is that a request header needs to be reused among multiple interfaces, such as authentication credentials. In this case, in addition to declaring the same parameters in each interface, the API class of UtilMeta provides a more concise way: declare common parameters in the API class. The usage is as follows
```python
from utilmeta.core import api, request
import utype

class RootAPI(api.API):
    auth_token: str = request.HeaderParam(alias='X-Auth-Token')
	
    @api.post
	def operation(self):
		return self.auth_token
```

In this way, all interfaces defined in the API class need to provide `X-Auth-Token` this request header parameter, and you can also get the corresponding value `self.auth_token` directly.

### Parse cookies

The cookie field of the request header can carry a series of key parameters to maintain a session with the server or carry credential information. There are two ways to declare the cookie parameter

*  `request.CookieParam`: Declare a single Cookie parameter
*  `request.Cookies`: Declare a complete Cookie object

```python
from utilmeta.core import api, request

class RootAPI(api.API):
    sessionid: str = request.CookieParam(required=True)
    csrftoken: str = request.CookieParam(default=None)

    @api.post
    def operation(self):
        return [self.sessionid, self.csrftoken]
```

In this example, we declare two generic cookie parameters in the RootAPI class, and the request passes the cookie parameters as follows
```http
POST /api/operation HTTP/1.1

Cookie: csrftoken=xxxx; sessionid=xxxx;
```