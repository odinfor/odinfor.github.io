---
layout: post
title: "「python」Django之CSRF"
subtitle: 'django防止csrf攻击策略'
author: "odin"
header-style: text
tags:
  - python
---
![]({{site.baseurl}}/img/in-post/post-python/django-girls.jpg)

在django中对post请求不加任何修饰时，一定会得到403 Forbidden的返回。通常我们需要设置csrf。对csrf的设置常见措施有以下几种：
* 全局禁用csrf。在setting.py配置文件中的MIDDLEWARE中注释`django.middleware.csrf.CsrfViewMiddleware`。当然这个方法比较少见也不推荐。
* 对指定的post请求取消csrf验证
    1. 方式一：在接口上添加装饰器`@csrf_exempt`对该接口取消csrf验证。
    2. 方式二：在路由上取消，如：`url(r'^post/get_data/$', csrf_exempt(post_data), name='post_data'),`
* 对指定的psot请求添加装饰器`@requires_csrf_token(view)`添加csrf验证。

那么什么是csrf？为什么需要csrf呢？
![]({{site.baseurl}}/img/in-post/你不会搜索吗.jpg)

#### 什么是csrf
那么我们先大致介绍一下，什么是csrf。
csrf是一种常见的跨站请求伪造攻击，是一种挟制用户在当前已登录的Web应用程序上执行非本意的操作的攻击方法。csrf的原理：
![]({{site.baseurl}}/img/in-post/post-python/csrf.png)
1. 首先用户打开浏览器，正常访问受信任的网站A。
2. 在用户正常输入确认信息后，网站A接受用户的登录，产生cookie信息并返回给用户浏览器。
3. 用户在对网站A的正常访问期间，在同一个浏览器中新建标签页访问网站B。
4. 网站B在接收到用户请求后，产生并返回一个攻击性代码，要求用户浏览器访问网站A并进行一些操作。
5. 浏览器在接收到这些攻击性代码后，根据网站B的请求，在用户不知情的情况下携带Cookie信息，向网站A发出请求。网站A并不知道该请求其实是由B发起的，所以会根据用户C的Cookie信息以C的权限处理该请求，导致来自网站B的恶意代码被执行。

csrf利用的就是对用户浏览器的信任。

#### 如何避免csrf
csrf防御措施主要有以下三种
1. 令牌同步模式。
    >当用户发送请求时，服务器端应用将令牌（英语：token，一个保密且唯一的值）嵌入HTML表格，并发送给客户端。客户端提交HTML表格时候，会将令牌发送到服务端，令牌的验证是由服务端实行的。令牌可以通过任何方式生成，只要确保随机性和唯一性（如：使用随机种子【英语：random seed】的哈希链 ）。这样确保攻击者发送请求时候，由于没有该令牌而无法通过验证。
2. 检查Referer字段。
    >HTTP头中有一个Referer字段，这个字段用以标明请求来源于哪个地址。在处理敏感数据请求时，通常来说，Referer字段应和请求的地址位于同一域名下。以上文银行操作为例，Referer字段地址通常应该是转账按钮所在的网页地址，应该也位于www.examplebank.com之下。而如果是CSRF攻击传来的请求，Referer字段会是包含恶意网址的地址，不会位于www.examplebank.com之下，这时候服务器就能识别出恶意的访问。  
    >这种办法简单易行，工作量低，仅需要在关键访问处增加一步校验。但这种办法也有其局限性，因其完全依赖浏览器发送正确的Referer字段。虽然http协议对此字段的内容有明确的规定，但并无法保证来访的浏览器的具体实现，亦无法保证浏览器没有安全漏洞影响到此字段。并且也存在攻击者攻击某些浏览器，篡改其Referer字段的可能。
3. 添加校验token。
    >csrf的攻击之所以会成功是因为服务器端身份验证机制可以通过Cookie保证一个请求是来自于某个用户的浏览器，但无法保证该请求是用户允许的。因此，预防csrf攻击简单可行的方法就是在客户端网页上添加随机数，在服务器端进行随机数验证，以确保该请求是用户允许的。

Django是通过添加校验token的方法来防御csrf攻击的。

#### django防御csrf策略
在客户端添加csrftoken，服务端通过中间件`django.middleware.csrf.CsrfViewMiddleware`进行验证。
在django当中防御csrf攻击的方式有两种：
1. 在表单当中附加csrftoken。 
2. 通过request请求中添加X-CSRFToken请求头。

##### 在表单中附加csrftoken
```python
"""后端"""
from django.shortcuts import render
from django.template.context_processors import csrf

def ajax_demo(request):
     # csrf(request)构造出{‘csrf_token’: token}
    return render(request, 'post_demo.html', csrf(request))
```

```javaScript
  // 前端
  $('#send').click(function(){
                
    $.ajax({
        type: 'POST',
        url:'{% url 'ajax:post_data' %}',
        data: {
                username: $('#username').val(),
                content: $('#content').val(),
               'csrfmiddlewaretoken': '{{ csrf_token }}'  关键点
            },
        dataType: 'json',
        success: function(data){

        },
        error: function(){
    
        }
    
    });
  });
```

##### 在request请求中添加X-CSRFToken请求头
```python
"""后端"""
from django.shortcuts import render
from django.views.decorators.csrf import ensure_csrf_cookie

@ensure_csrf_cookie
def ajax_demo(request):
    return render(request, 'ajax_demo.html')
```

```javaScript
//在进行post提交时，获取Cookie当中的csrftoken并在请求中添加X-CSRFToken请求头, 该请求头的数据就是csrftoken。
//通过$.ajaxSetup方法设置AJAX请求的默认参数选项，在每次ajax的POST请求时，添加X-CSRFToken请求头

<script type="text/javascript">
    $(function(){

        function getCookie(name) {
            var cookieValue = null;
            if (document.cookie && document.cookie != '') {
                var cookies = document.cookie.split(';');
                for (var i = 0; i < cookies.length; i++) {
                    var cookie = jQuery.trim(cookies[i]);
                    // Does this cookie string begin with the name we want?
                    if (cookie.substring(0, name.length + 1) == (name + '=')) {
                        cookieValue = decodeURIComponent(cookie.substring(name.length + 1));
                        break;
                    }
                }
            }
            return cookieValue;
        }

        <!--获取csrftoken-->
        var csrftoken = getCookie('csrftoken');
        console.log(csrftoken);

        //Ajax call
        function csrfSafeMethod(method) {
            // these HTTP methods do not require CSRF protection
            return (/^(GET|HEAD|OPTIONS|TRACE)$/.test(method));
        }

        $.ajaxSetup({
            crossDomain: false, // obviates need for sameOrigin test
            //请求前触发
            beforeSend: function(xhr, settings) {
                if (!csrfSafeMethod(settings.type)) {
                    xhr.setRequestHeader("X-CSRFToken", csrftoken);
                }
            }
        });

        $('#send').click(function(){
            console.log($("#comment_form").serialize());

            $.ajax({
                type: 'POST',
                url:'{% url 'ajax:post_data' %}',
                data: {
                        username: $('#username').val(),
                        content: $('#content').val(),
                       //'csrfmiddlewaretoken': '{{ csrf_token }}'
                    },
                dataType: 'json',
                success: function(data){
                         
                    
                },
                error: function(){

                }

            });
        });


    });
</script>
```


