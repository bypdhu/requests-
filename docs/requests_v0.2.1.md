# v0.2.1 file attribute

## 1. 前述

### history
0.2.1 (2011-02-14)
- Added file attribute to POST and PUT requests for multipart-encode file uploads.
- Added Request.url attribute for context and redirects

### 感悟
核心代码还是一个文件core.py，但是增加了一个包poster，包括encode.py和streaminghttp.py，用来处理文件流。
看历史有两条，一条是说对于POST和PUT增加文件参数，一条是说增加了Request的类属性url(有啥用还不知道)。

### 用时
由于测试和学习一些基本用法，花了不少时间
- 60分钟阅读，30分钟编写
- 60分钟阅读，30分钟编写
- total： 120分钟阅读，60分钟编写


## 2. 概述
- Request类增加了类属性files
- 增加了packages.poster.encode.py 和packages.poster.streaminghttp.py两个文件


## 3. 详解

### core.py

#### Request class
- __repr__优化，去掉多余的try，'<Request [%s]>' % (self.method)在None时也可以
- 判断method优化，从self.method == 'GET'变为self.method.lower() == 'get'
- PUT
    - 添加，如果self.files有内容，则处理
    - register_openers，注册自定义的文件流handler，后面详解
    - 根据files处理得到data和headers，`datagen, headers = multipart_encode(self.files)`，后面详解
    - 更新headers
    - self.files没有内容时和之前一样
    - response增加url属性

#### post method
- 增加files参数，默认None

#### put method
- 增加files参数，默认{}。和post对比默认参数，猜测可能是post用于新建资源，参数不需要且不应该和其他请求一样，而put对资源更新，可以服用参数。


### packages.poster.streaminghttp.py
Streaming HTTP uploads module。从httplib和urllib2扩展。

#### register_openers method
注册http(s)流handlers到urllib2的全局默认openers，并优先使用。
- HTTP的handlers：StreamingHTTPHandler, StreamingHTTPRedirectHandler
- 如果httplib有属性HTTPS，handlers增加StreamingHTTPSHandler
- opener = urllib2.build_opener(*handlers)，根据handlers创建opener，先不管实现。
- urllib2.install_opener(opener)，用opener替换urllib2中全局的_opener
- 返回opener

#### _StreamingHTTPMixin class
HTTP和HTTPS的混合类，实现了send方法
- send方法和HTTPConnection.send类似，使用时根据调用顺序覆盖
- 具体的内容暂时看不懂，放过

#### StreamingHTTPConnection/StreamingHTTPSConnection class
- 覆盖send方法，`class StreamingHTTPConnection(_StreamingHTTPMixin, httplib.HTTPConnection):`

#### StreamingHTTPRedirectHandler class
继承自urllib2.HTTPRedirectHandler，重写了redirect_request方法
- handler_order减一，应该是比HTTPRedirectHandler提高一级优先级
- redirect_request看起来和urllib2.HTTPRedirectHandler.redirect_request一样，那就忽略了。

#### StreamingHTTPHandler/StreamingHTTPSHandler class
继承自urllib2.HTTPHandler
- 重写http_open方法，执行StreamingHTTPConnection而不是httplib.HTTPConnection
- 重写http_request方法，进行了Content-length判断


### packages.poster.encode.py
multipart/form-data encoding module.
PUT和POST时，将 name/value 形式编码为 multipart/form-data 形式，这是通过HTTP传输文件的标准方式。

#### multipart_encode method
将name/value元组或者MultipartParam对象列表编码为str
返回multipart_yielder生成器和headers
- 处理boundary
- 根据boundary和params获取headers

#### get_boundary method
获取boundary，简单略过

#### encode_and_quote method
顾名思义

#### _strify method
将unicode输入使用utf-8编码，或者使用str()转换

#### MultipartParam class
表示multipart/form-data类型请求的一个参数
- __init__方法，入参有name,value,filename,filetype,filesize,fileobj,cb。
    - ``self.name = Header(name).encode()``使用email.header中的Header类，这是一个MIME-compliant头
    - 使用_strify处理value
    - 处理filename，
        - 如果是unicode，则先 ``filename.encode("ascii", "xmlcharrefreplace")``，`u'abc"哈啊'` 得到 `'abc"&#21704;&#21834;'` 的结果
        - 或者用str()转换
        - 然后将上述self.filename.encode("string_escape").replace('"', '\\"')，将 \x 变为 \\x，将 " 变为 \\。得到 `'abc\\"&#21704;&#21834;'`
    - 使用_strify处理filetype
    - filesize、fileobj、cb赋值
    - 断言self.value和self.fileobj不能同时不为None
    - fileobj时，filesize不存在，则自动发现。
        - 首先尝试 `os.fstat(fileobj.fileno()).st_size`。其中fileobj.fileno()返回一个整型的文件描述符。os.fstat() 方法用于返回文件描述符fd的状态，类似 stat()。
        - 其次使用File的seek和tell。
            - `fileobj.seek(0, 2)` 表示设置偏移量为0，从文件末尾算起，即文件读取指针移动到文件末尾
            - `fileobj.tell()` 表示获取文件指针当前位置，此处即为整个文件的大小。
        - `fileobj.seek(0)`，将文件指针设置回文件开始位置0
        - 都获取不到则报错
- __cmp__方法，比较两个对象的大小，返回1或者-1。使用cmp()进行列表的比较，按照`['name', 'value', 'filename', 'filetype', 'filesize', 'fileobj']`的顺序
- reset方法，用于将fileobj的文件指针位置移到0
- from_file，这是一个classmethod，根据paramname和filename得到一个MultipartParam对象
    - 可以学习根据文件位置获取文件信息的方法：
         ```python
         filename=os.path.basename(filename), //获取文件名
         filetype=mimetypes.guess_type(filename)[0], //猜测文件类型
         filesize=os.path.getsize(filename), //获取文件大小
         fileobj=open(filename, "rb") //获取文件对象
         ```
- from_params，这也是一个classmethod，根据params得到一个MultipartParam对象列表，注意，是列表
    - params支持list和dict格式，同时支持name/value形式和MultipartParam对象。
    如 `[(key1,value1),(key2,value2)]`, `[(key3,value2),MultipartParam()]`, `{key4:value4,key5:value5}`
    - ``if hasattr(params, 'items'): params = params.items()``，处理字典为可迭代对象
    - 返回retval列表
    - 进行迭代 `for item in params:`
    - 如果item为MultipartParam对象，则直接添加到retval
    - 如果item的值value为MultipartParam对象，断言item.name==value.name，将value添加到retval
    - 如果value有read方法，即value为file-like对象，则根据value对象信息创建文件类型的MultipartParam对象，添加到retval
    - 如果value没有read方法，则当做name/value形式创建MultipartParam对象，添加到retval

#### encode_string method

#### encode_file_header method

#### get_body_size method

#### get_headers method


#### multipart_yielder class




### 


## 4. 