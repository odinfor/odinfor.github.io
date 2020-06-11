---
layout: post
title: "「python」WSGI&uWSGI"
subtitle: 'python web服务中WSGI与uWSGI分析'
author: "odin"
header-style: text
tags:
  - python
---

python在web开发中一定见过WSGI、uWSGI、uwsgi这些。django和flask、trando这些框架都是运行在WSGI协议之上的。
django、flask、tronado这些框架运行起来本身就能提供服务访问，那么为什么部署还会另外使用到nginx或者是appache这些服务器呢。
![]({{site.baseurl}}/img/in-post/post-python/uwsgi&wsgi_1.png)
上图中左边是一些服务器，右边是常见的一些框架，中间只有一个WSGI。就上图展开聊一聊WSGI，uWSGI，和服务器、WSGI、框架之间的联系。

#### WSGI

WSGI是一种规范协议，全称`Web Server Gateway Interface`。不是服务器，不是通讯协议，也不是框架，而是一种**规范**。用来描述web server和web application通信的规范。实现web框架server和application的解耦。
WSGI协议主要包含两个部分：
* `WSGI server`：负责从客户端接收请求`request`，将请求转发给`application`，将`application`返回的请求响应结果`response`发送给客户端。
    >WSGI server工作原理：从底层解析http解析，然后调用应用程序，给应用程序提供(环境信息)和(回调函数)， 这个回调函数是用来将应用程序设置的http header和status等信息传递给服务器方.
* `WSGI application`：接收由`server`转发的请求，经过服务端相应的处理最后得到的结果发送给`WSGI server`。结果包含header,body和status,以便返回给服务器方。

`WSGI application`：中可以包含多个中间件`middleware`这些中间件需要同时实现server与application，因此可以在WSGI服务器与WSGI应用之间起调节作用：对服务器来说，中间件扮演应用程序，对应用程序来说，中间件扮演服务器。

![]({{site.baseurl}}/img/in-post/post-python/uwsgi&wsgi_2   .png)

WSGI协议其实是定义了一种server与application解耦的规范，上图很形象的展示了WSGI将server和application联系起来，即可以有多个实现WSGI server的服务器，也可以有多个实现WSGI application的框架，那么就可以选择任意的server和application组合实现自己的web应用。例如uWSGI和Gunicorn都是实现了WSGI server协议的服务器，Django，Flask是实现了WSGI application协议的web框架，可以根据项目实际情况搭配使用。这些便是WSGI的优点。

#### uwsgi

与`WSGI`一样是一种通信协议，是`uWSGI`服务器的独占协议，用于定义传输信息的类型(type of information)，每一个uwsgi packet前4byte为传输信息类型的描述，与WSGI协议是两种东西。

#### uWSGI
`uWSGI`是一个web服务器，实现了`http` ,`uwsgi`, `WSGI`等协议。

#### WSGI协议实现
以django为例，分析一下WSGI的实现过程。

##### django WSGI application

`WSGI application`应该实现为一个可调用对象，例如函数、方法、类(包含`call`方法)。需要接收两个参数：一个字典，该字典可以包含了客户端请求的信息以及其他信息，可以认为是请求上下文，一般叫做environment（编码中多简写为environ、env）一个用于发送HTTP响应状态（HTTP status ）、响应头（HTTP headers）的回调函数通过回调函数将响应状态和响应头返回给server，同时返回响应正文(response body)，响应正文是可迭代的、并包含了多个字符串。下面是Django中application的具体实现部分：

```python
class WSGIHandler(base.BaseHandler):
    initLock = Lock()
    request_class = WSGIRequest

    def __call__(self, environ, start_response):
        # 加载中间件
        if self._request_middleware is None:
            with self.initLock:
                try:
                    # Check that middleware is still uninitialized.
                    if self._request_middleware is None:
                        self.load_middleware()
                except:
                    # Unload whatever middleware we got
                    self._request_middleware = None
                    raise

        set_script_prefix(get_script_name(environ))
        # 请求处理之前发送信号
        signals.request_started.send(sender=self.__class__, environ=environ)
        try:
            request = self.request_class(environ)
        except UnicodeDecodeError:
            logger.warning('Bad Request (UnicodeDecodeError)',
                exc_info=sys.exc_info(),
                extra={'status_code': 400,})
            response = http.HttpResponseBadRequest()
        else:
            response = self.get_response(request)

        response._handler_class = self.__class__

        status = '%s %s' % (response.status_code, response.reason_phrase)
        response_headers = [(str(k), str(v)) for k, v in response.items()]
        for c in response.cookies.values():
            response_headers.append((str('Set-Cookie'), str(c.output(header=''))))
        # server提供的回调方法，将响应的header和status返回给server
        start_response(force_str(status), response_headers)
        if getattr(response, 'file_to_stream', None) is not None and environ.get('wsgi.file_wrapper'):
            response = environ['wsgi.file_wrapper'](response.file_to_stream)
        return response
```

可以看出application的流程包括:加载所有中间件，以及执行框架相关的操作，设置当前线程脚本前缀，发送请求开始信号；处理请求，调用`get_response()`方法处理当前请求，该方法的的主要逻辑是通过`urlconf`找到对应的`view`和`callback`，按顺序执行各种`middleware`和`callback`。调用由`server`传入的`start_response()`方法将响应header与status返回给server。返回响应正文

##### django WSGI Server

负责获取http请求，将请求传递给WSGI application，由application处理请求后返回response。以Django内建server为例看一下具体实现。
通过runserver运行django项目，在启动时都会调用下面的run方法，创建一个WSGIServer的实例，之后再调用其`serve_forever()`方法启动服务。

```python

def run(addr, port, wsgi_handler, ipv6=False, threading=False):
    server_address = (addr, port)
    if threading:
        httpd_cls = type(str('WSGIServer'), (socketserver.ThreadingMixIn, WSGIServer), {})
    else:
        httpd_cls = WSGIServer
    # 这里的wsgi_handler就是WSGIApplication
    httpd = httpd_cls(server_address, WSGIRequestHandler, ipv6=ipv6)
    if threading:
        httpd.daemon_threads = True
    httpd.set_app(wsgi_handler)
    httpd.serve_forever()
```

下面表示WSGI server服务器处理流程中关键的类和方法
![]({{site.baseurl}}/img/in-post/post-python/uwsgi&wsgi_3.png)


**WSGIServerr**
`run()`方法会创建`WSGIServer`实例，主要作用是接收客户端请求，将请求传递给`application`，然后将`application`返回的response返回给客户端。
* 创建实例时会指定HTTP请求的`handler：WSGIRequestHandler`类
* 通过`set_app`和`get_app`方法设置和获取`WSGIApplication`实例`wsgi_handler`
* 处理http请求时，调用`handler_request`方法，会创建`WSGIRequestHandler`实例处理http请求。
* `WSGIServer`中`get_request`方法通过`socket`接受请求数据。

**WSGIRequestHandler**
* 由`WSGIServer`在调用`handle_request`时创建实例，传入`request`、`cient_address`、`WSGIServer`三个参数，__init__方法在实例化同时还会调用自身的handle方法。
* `handle`方法会创建`ServerHandler`实例，然后调用其`run`方法处理请求

**ServerHandler**
* `WSGIRequestHandler`在其handle方法中调用run方法，传入`self.server.get_app()`参数，获取WSGIApplication，然后调用实例`(__call__)`，获取response，其中会传入start_response回调，用来处理返回的header和status。
* 通过application获取response以后，通过`finish_response`返回response

**WSGIHandler**
* `WSGI`协议中的`application`，接收两个参数，environ字典包含了客户端请求的信息以及其他信息，可以认为是请求上下文，`start_response`用于发送返回status和header的回调函数。

虽然上面一个WSGI server涉及到多个类实现以及相互引用，但其实原理还是调用WSGIHandler，传入请求参数以及回调方法start_response()，并将响应返回给客户端。

###### django simple_server

django的simple_server.py模块实现了一个简单的HTTP服务器，并给出了一个简单的demo，可以直接运行，运行结果会将请求中涉及到的环境变量在浏览器中展示出来。
其中包括上述描述的整个http请求的所有组件:`ServerHandler`, `WSGIServer`, WSGIRequestHandler，以及demo_app表示的简易版的`WSGIApplication`。
可以看一下整个流程：

```python

if __name__ == '__main__':
    # 通过make_server方法创建WSGIServer实例
    # 传入建议application，demo_app
    httpd = make_server('', 8000, demo_app)
    sa = httpd.socket.getsockname()
    print("Serving HTTP on", sa[0], "port", sa[1], "...")
    import webbrowser
    webbrowser.open('http://localhost:8000/xyz?abc')
    # 调用WSGIServer的handle_request方法处理http请求
    httpd.handle_request()  # serve one request, then exit
    httpd.server_close()
    
def make_server(
    host, port, app, server_class=WSGIServer, handler_class=WSGIRequestHandler
):
    """Create a new WSGI server listening on `host` and `port` for `app`"""
    server = server_class((host, port), handler_class)
    server.set_app(app)
    return server

# demo_app可调用对象，接受请求输出结果def demo_app(environ,start_response):
    from io import StringIO
    stdout = StringIO()
    print("Hello world!", file=stdout)
    print(file=stdout)
    h = sorted(environ.items())
    for k,v in h:
        print(k,'=',repr(v), file=stdout)
    start_response("200 OK", [('Content-Type','text/plain; charset=utf-8')])
    return [stdout.getvalue().encode("utf-8")]
```

demo_app()表示一个简单的WSGI application实现，通过make_server()方法创建一个WSGIServer实例，调用其handle_request()方法，该方法会调用demo_app()处理请求，并最终返回响应。

##### uWSGI

uWSGI旨在为部署分布式集群的网络应用开发一套完整的解决方案。主要面向web及其标准服务。由于其可扩展性，能够被无限制的扩展用来支持更多平台和语言。uWSGI是一个web服务器，实现了WSGI协议，uwsgi协议，http协议等。
uWSGI的主要特点是：
* 超快的性能
* 低内存占用
* 多app管理
* 详尽的日志功能（可以用来分析app的性能和瓶颈）
* 高度可定制（内存大小限制，服务一定次数后重启等）

uWSGI服务器自己实现了基于uwsgi协议的server部分，我们只需要在uwsgi的配置文件中指定application的地址，uWSGI就能直接和应用框架中的WSGI application通信。



#### 参考文章

https://www.jianshu.com/p/679dee0a4193

https://www.letiantian.me/2015-09-10-understand-python-wsgi/

http://xiaorui.cc/archives/3209