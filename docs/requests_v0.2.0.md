# v0.2.0 Birth

## 1. 前述

### history
0.2.0 (2011-02-14)
- Birth!

### 感悟
哈哈，代码只有一个core.py，突然觉得大神也是一步一个坑走出来的。只要有信心，人人是大神。
来看一下这一个文件的组成。
历史记录只有一个词：Birth!哈哈，见证大神作品诞生了。

### 用时
- 30分钟阅读，30分钟编写
- total： 30分钟阅读，30分钟编写


## 2. 概述

- 导入模块只有urllib和urllib2
- 文件模块信息
- 全局AUTOAUTHS变量，可以记录url对应的用户名密码，请求时不用输入用户名和密码时就可以自动从里面查找使用。
- _Request类，继承自urllib2.Request,内部使用
- Request类，请求对象，包含一个主要方法send，支持GET/HEAD/DELETE/PUT/POST
- Response类，返回对象
- AuthObject类，基本认证对象
- get/head/delete/put/post方法
- add_autoauth方法
- 异常RequestException(Exception), 继承自RequestException的 AuthenticationError, URLRequired, InvalidMethod

## 3.详解

#### _Request class
继承自urllib2.Request，输入参数一样，增加了一个get_method方法，记录初始化时的method，或者返回urllib2.Request.get_method(self)

#### Request class
包含request请求需要的方法和属性的类，或者说实现。
- 允许的方法为('GET', 'HEAD', 'PUT', 'POST', 'DELETE')
- __init__不带参数，看样子是想要后续set。初始化了类属性，包括url,headers,method,params,data,response,auth,sent。 response初始化为Response()
- 果然，有一个__setattr__方法。看一下实现，其中赋值的方式没用过，mark一下。
```python
def __setattr__(self, name, value):
    if (name == 'method') and (value):
        if not value in self._METHODS:
            raise InvalidMethod()
    object.__setattr__(self, name, value)
```
缺点呢，没有对method进行忽略大小写处理。
重点学习一下object.__setattr__(self, name, value)
- __repr__方法，返回实例对象的可打印字符串
- _get_opener，获取合适的urllib2的opener对象。如果有auth，则处理返回urllib2.build_opener(handler).open；否则返回urllib2.urlopen
- send方法，核心方法。
输入参数有一个anyway，表示当sent设置为true时，仍然继续发送。
先check，然后根据方法为(GET,HEAD,DELETE)/PUT/POST分为三部分判断发送。最后设置selt.sent。
    - GET/HEAD/DELETE
        - params为dict时，使用urllib.urlencode编码为string
        - 使用url/params/method构造_Request对象req
        - 设置req的headers
        - 获取合适的opener
        - 使用opener打开req，使用try-catch。
        - 设置response对象的status_code和headers，如果请求为GET，设置content
        - except urllib2.HttError why: 生效时，设置response的status_code为why.code。试了一下，现在这个exception已经没有了，为URLError
    - PUT
        - 构造_Request对象req时，使用了url和method，没有使用params
        - 设置req的headers
        - **设置req.data**
        - 获取合适的opener，然后打开req
        - 设置response的status_code、headers和content
    - POST
        - **设置req.data时，当self.data为dict时，编码为string**
        - 其他基本同PUT
- self.sent = True if success else False。 那不用success这个变量啊，使用self.sent不就行了

#### Response class
请求的返回对象
- 包含三个类属性: content/status_code/headers
- __repr__方法

#### AuthObject class
很简单，初始化记录用户名和密码

#### get method
Sends a GET request. Returns :class:`Response` object
输入参数为 url/params/headers/auth。
其中params和headers的默认值为{}，这样不太好吧。。。看一下实现吧
- 初始化一个Request对象r
- 设置method/url/params/headers/auth
- 调用r.send()
- 返回r.response

#### head method
同get

#### post method
基本同get，输入参数没有params，多了个data，默认值为{}

#### put method
基本同post，只是data的默认值为''，不知道为啥 

#### delete method
基本同get，只是从get搬过来的params参数并没有用到，哈哈哈

#### add_autoauth method
需要手动注册，AUTOAUTHS里面才有内容，用于auth自动发现


## 4. 收获
全看懂了，哈哈哈