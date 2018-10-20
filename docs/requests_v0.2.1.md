# v0.2.1 file upload

## 1. 前述

### history
0.2.1 (2011-02-14)
- Added file attribute to POST and PUT requests for multipart-encode file uploads.
- Added Request.url attribute for context and redirects

### 感悟
核心代码还是一个文件core.py，但是增加了一个包poster，包括encode.py和streaminghttp.py，用来处理文件流。
看历史有两条，一条是说对于POST和PUT增加文件参数，一条是说增加了Request的类属性url(有啥用还不知道)。

### 用时
由于测试和学习一些基本用法，花了不少时间，当然，阅读和记录是同时的。大概算一下时间。
- 60分钟阅读，20分钟编写
- 60分钟阅读，20分钟编写
- 50分钟阅读，20分钟编写
- total： 170分钟阅读，60分钟编写


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
- __init__方法，入参有name,value,filename,filetype,filesize,fileobj,cb。初始化各种参数。
    - 包装name，``self.name = Header(name).encode()``使用email.header中的Header类，这是一个MIME-compliant头
    - 使用_strify处理value
    - 处理filename，
        - 如果是unicode，则先 ``filename.encode("ascii", "xmlcharrefreplace")``，`u'abc"哈啊'` 得到 `'abc"&#21704;&#21834;'` 的结果
        - 或者用str()转换
        - 然后将上述self.filename.encode("string_escape").replace('"', '\\"')，将 \x 变为 \\x，将 " 变为 \\。得到 `'abc\\"&#21704;&#21834;'`
    - 使用_strify处理filetype
    - filesize、fileobj、cb赋值
    - 断言self.value和self.fileobj不能同时不为None
    - fileobj时，filesize不存在，则自动发现。
        - 首先尝试 ``os.fstat(fileobj.fileno()).st_size``。其中fileobj.fileno()返回一个整型的文件描述符。os.fstat() 方法用于返回文件描述符fd的状态，类似 stat()。
        - 其次使用File的seek和tell。
            - ``fileobj.seek(0, 2)`` 表示设置偏移量为0，从文件末尾算起，即文件读取指针移动到文件末尾
            - ``fileobj.tell()`` 表示获取文件指针当前位置，此处即为整个文件的大小。
        - ``fileobj.seek(0)``，将文件指针设置回文件开始位置0
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
- encode_hdr方法，传入boundary，返回这个参数对象的编码header
    - boundary编码和转换
    - 一个类headers列表，用于生成单个对象的body部分
    - 下面生成form-data的header Content-Disposition，
    - 文件形式如 `Content-Disposition：form-data; name="%s"; filename="%s"`，key-value形式如 `Content-Disposition：form-data; name="%s"`
    - 设置content-type
    - headers最后加入两个空字符串，用于后续生成两个空行
    - ``"\r\n".join(headers)``添加换行回车，生成一个对象的body字符串
    - 如 `--this_is_boundary\r\nContent-Disposition: form-data; name="a"; filename="f"\r\nContent-Type: text/plain; charset=utf-8\r\n\r\n`
- encode方法，传入boundary，返回参数对象的编码字符串
    - 根据self.value判断，如果不为空，则value=self.value；如果为空，则认为这是一个文件参数，读取文件流，value=self.fileobj.read()
    - 判断value中不应该有boundary匹配的字符串
    - 调用encode_hdr，加上value内容，构造返回。
    - 如一个文件参数的返回为：`--this_is_boundary\r\nContent-Disposition: form-data; name="a"; filename="f"\r\nContent-Type: text/plain; charset=utf-8\r\n\r\nthis is file content.\r\n`
- iter_encode方法，传入boundary和缓冲块大小，返回参数对象的编码生成器
    - 使用get_size方法获取大小total
    - 设置读取的初始位置current=0
    - 如果self.value存在，则参数对象是key-value形式
        - 使用encode方法编码参数的header，得到block对象
        - current增加block的大小
        - 返回生成器``yield block``，调用next方法时返回block。最多只会执行一次。
        - 最后如果有回调函数，则执行``self.cb(self, current, total)``。根据当前对象、已传输大小、总的大小进行需要的处理。
        - 可以达到延时执行回调函数的效果，即调用结束最后执行回调函数。mark一下
    - 如果self.value为None，则参数对象为文件形式
        - 前面部分一样，都是编码参数对象的header、返回生成器、执行回调函数
        - 然后是文件处理独有的部分，进入无限循环。
        - 读取文件缓存块大小的内容。
        - 如果block没有内容，表示已经读取结束，则生成器(最多执行一次)最后返回`"\r\n"`，当然，current也得先+2。最后执行回调函数。
        - 如果block有内容，先添加到last_block中，
            - 然后正则判断文件内容中不应该有边界标识
            - last_block只记录最后一段字符串，大小为边界标识+4(前后的`--`)。想一下，这个确实可以保证后面再添加的内容拼起来也不会和边界标识重复。节省空间，很巧妙，mark一下！！！
            - 后面就是一样了，注意这里返回的生成器会是多次调用的，回调函数也是。
- get_size方法，获取用boundary处理并编码后的参数对象的字节大小
    - 根据encode方法和encode_hdr方法内容可知，计算的方法为encode_hdr的到的header大小+value大小+2('\r\n')

#### encode_string method
传入boundary、name、value，构造MultipartParam对象，返回最终编码字符串

#### encode_file_header method
只处理文件形式参数的头信息，返回编码的头信息字符串

#### get_body_size method
获取多个参数的body字节数
    - 计算方法为每个参数的get_size方法得到的大小+len(boundary) + 6
    - 6的来由：文件读取完毕最后添加"\r\n"两个字节，最后添加一个boundary标识，前后都要加`--`

#### get_headers method
获取multipart/form-data类型的headers，包括Content-Type和Content-Length

#### multipart_yielder class
- __init__方法，初始化params(在这里是MultipartParam对象列表)、boundary、回调函数cb
- __iter__方法，返回自身self
- next方法，核心方法，参数对象的生成器执行函数
    - 如果self.param_iter不为None，获取一个参数对象的完整body字符串
        - 同前面的执行器函数类似，也是先获取block
        - current计数增加
        - 执行回调函数
        - 返回block
        - 直到StopIteration，则重置self.p(param对象)和self.param_iter为None。
    - 否则，继续，如果self.i(处理过的params个数)为None，结束本方法。self.i的初始值为0，会继续往下。
    - 如果self.i>=len(self.params)，表示params应该都处理完毕，进入终止边界处理流程
        - 将相关参数都置为None，包括self.p,self.i,self.param_iter
        - block为边界结束标识，形式为``"--%s--\r\n" % self.boundary``
        - current计数增加，增量为len(self.boundary)+6
        - 执行回调函数
        - 返回表示结束边界的block，此时，调用者应该已经获取了所有参数对象和结束边界拼成的完整body字符串
    - 如果self.i<len(self.params)，则处理下一个param对象
        - 从i=0开始，获取params中的参数对象self.p
        - 将self.param_iter设置为对象self.p的生成器
        - i+=1
        - 返回self.next()，即调用自身。
    - 所以，next方法的逻辑即为： 
        - 第一次
            - 先直接到最后 self.p=self.params[0]
            - self.param_iter=self.p.iter_encode(self.boundary)
            - i+=1，此时i=1
            - 然后第一次调用还未结束，因为继续调用了自身
            - 此时，self.param_iter不为None，返回的是self.p的生成器对象
        - 然后每次调用next，得到的block可能是同一个参数对象的内容，也可能是另一个参数对象的内容
        - 直到结束
    - 可以学习一下此处next的逻辑
        - 处理多个对象，每个对象还有各自的生成器，然后多个对象拼接，最后添加结束标识
        - 处理一个对象，多次调用next直到结束此对象的调用(StopIteration)，将此对象和执行器设为None，这样就可以执行下一个对象
        - 根据结束标识(StopIteration)，结束next方法
        - 根据已经处理过的数量判断，进入终止操作流程，设置结束标识，并将此对象和执行器设为None(如果判断结束标识的部分在最前面，这里就不需要了)
        - 设置新的对象和执行器
        - 返回自身
- reset方法
    - 重置参数对象

## 4. 收获
这一段里最需要学习的是生成器的用法，虽然知道咋回事，但是基本没有用过，后面要考虑生成数据时多用一下生成器。