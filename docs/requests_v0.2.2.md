# v0.2.2 Monkey-patch & Cookie

## 1. 前述

0.2.2 (2011-02-14)
- Still handles request in the event of an HTTPError. (Issue #2)
- Eventlet and Gevent Monkeypatch support.
- Cookie Support (Issue #1)

### 感悟
和上一个版本是同一天提交的，改动不大。就core.py文件有修改

### 用时
- 30分钟阅读，30分钟编写
- total： 30分钟阅读，30分钟编写


## 2. 概述
- 略

## 3. 详解

#### eventlet.monkey_patch 
开头添加的，具体用途这里不讨论
```python
try:
	import eventlet
	eventlet.monkey_patch()
except ImportError:
	pass

if not 'eventlet' in locals():
	try:
		from gevent import monkey
		monkey.patch_all()
	except ImportError:
		pass
```

#### Request class
- __init__方法添加类属性cookiejar
- _get_opener方法添加cookie_handler，处理cookiejar
- _build_response方法，将重复操作提取
- 失败时，仍然和成功同样的方式处理返回结果。因为失败时的返回对象是类似于成功对象的。
- get/head/delete/put/post添加cookie参数

## 4. 收获
略