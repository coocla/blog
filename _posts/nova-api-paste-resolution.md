title: Paste Deployment简介及nova-api-paste.ini解析
categories: OpenStack
tags: [openstack, paste deployment]
date: 2015-04-27 21:53:09
---
## 前言
利用Paste Deployment添加API跨域支持
## 介绍
paste deployment是一个配置WSGI APP和发现服务的一个系统，对于WSGI APP的开发者，它提供了一个简单的函数(loadapp)，以便从配置文件和python egg中加载一个wsgi app，对于提供的WSGI APP，它只请求一个简单的入口，因此你不需要提供你的app详情给它。

通俗来讲：可以将它理解成是一种机制或者流程模式，在一个Server中，可以通过他将server中的app通过它进行连接组合。其中的连接方式他通过一个简单的接口提供给用户，用户只需要配置好paste deployment即可完成app的连接组合，这对用户完全是透明的。

与paste deployment的主要交互方式是配置文件，让整体server中app的流程处理按照配置文件中的流向去运行。
## 配置文件
配置文件中由多个sections(段)组成，每个sections是由[type:name]这样的格式组成，如果不是这样的格式，将会被忽略。<!--more-->
### type可选值
* app: 定义了一个符合wsgi协议的application
* composite: 表示需要将请求指向到多个或多种应用上
* filter: 定义一个过滤器，接受application为参数，并且返回一个application
* filter-app: 和filter类似，不同的是可以直接指明此filter要作用于哪个app
* pipeline: 当使用多个filter作用于一个app上需要使用pipeline，它提供一个* key为pipeline，value是一个列表，列表必须以app为结尾。

### app的几种使用方法
* 指向另外一个配置文件中的application
[app:name]
use = config:other_config.ini
* 指向一个一个egg的URL
[app:name]
use = egg:MyAPP
* 指向一个指定模块中可调用的application
[app:name]
use = call:MyProject.app:myapp
* 指向其他的section
[app:name]
use = other_section_name
* 指向具体的python代码
[app:name]
paste.app_factory = MyAPP.module:app_factor

### composite
* [composite:main]
use = egg:Paste
/ = mainapp
/blog = blog
/wiki = wiki
这里首先调用了composite中的一个组件Paste(urlmap)，根据请求的url进行匹配指向到对应的app上

### filter
* 第一种方式是filter-with，在定义app时指定对应的过滤器
[app:main]
use = egg:Project
filter-with=dev
[filter:dev]
use = egg:Dev
* 第二种是filter-app，利用next关键字指定该过滤器过滤哪个app
[filter-app:dev]
use = egg:Dev
next = main
[app:main]
use = egg:Project
* 第三种是pipeline，将多个filter组成一串过滤条件，形成以app结尾的列表
[pipeline:main]
pipeline = filter1 filter2 filter3 admin[filter:filter1]
...
[filter:filter2]
...
[filter:filter3]
...
[app:admin]
...
在app真正被调用的时候，流程是先执行了filter1-->filter2 -->filter3-->admin，期间如果任一filter没有满足，都将中断后续的流程。

## 解读nova中api-paste.ini
*/etc/nova/api-paste.ini*的具体内容：
> 这里着重看下OpenStack的流向

```
############
# Metadata #
############
[composite:metadata]
use = egg:Paste#urlmap
/: meta
利用urlmap将请求执行执行到 meta

[pipeline:meta]
pipeline = ec2faultwrap logrequest metaapp
这里使用pipeline,分别使用ec2faultwrap, logrequest 针对metaapp进行过滤.由此也可以看出openstack为了兼容EC2引入的metatdata服务，并且使用一个神奇的地址169.254.169.254

[app:metaapp]
paste.app_factory = nova.api.metadata.handler:MetadataRequestHandler.factory
定义个app,指向到nova中MetadataRequestHandler类

#######
# EC2 #
#######
[filter:ec2faultwrap]
paste.filter_factory = nova.api.ec2:FaultWrapper.factory
定义一个filter,直接指向到nova中的代码

[filter:logrequest]
paste.filter_factory = nova.api.ec2:RequestLogging.factory
定义一个filter,指向到nova中RequestLogging.factory方法

#############
# OpenStack #
#############

[composite:osapi_compute]
use = call:nova.api.openstack.urlmap:urlmap_factory
/: oscomputeversions
/v1.1: openstack_compute_api_v2
/v2: openstack_compute_api_v2
/v2.1: openstack_compute_api_v21
/v3: openstack_compute_api_v3
这里利用composite将xxx/xxx形式的请求交给oscomputeversions，形似xxxx/v1.1/xxxxx请求交给openstack_compute_api_v2，来实现API版本控制

[composite:openstack_compute_api_v2]
use = call:nova.api.auth:pipeline_factory
noauth = compute_req_id faultwrap sizelimit noauth ratelimit osapi_compute_app_v2
keystone = compute_req_id faultwrap sizelimit authtoken keystonecontext ratelimit osapi_compute_app_v2
keystone_nolimit = compute_req_id faultwrap sizelimit authtoken keystonecontext osapi_compute_app_v2
针对openstack_compute_api_v2的实现来看，首先调用了nova.api.auth中pipeline_factory方法,从源代码可以看出，实际上_load_pipeline调用了keystone，
keystone属于是一个filter的集合，请求会依次通过前面的这些filter，最后到达osapi_compute_app_v2这个app

[composite:openstack_compute_api_v21]
use = call:nova.api.auth:pipeline_factory_v21
noauth = request_id faultwrap sizelimit noauth osapi_compute_app_v21
keystone = request_id faultwrap sizelimit authtoken keystonecontext osapi_compute_app_v21

[composite:openstack_compute_api_v3]
use = call:nova.api.auth:pipeline_factory_v21
noauth = request_id faultwrap sizelimit noauth_v3 osapi_compute_app_v3
keystone = request_id faultwrap sizelimit authtoken keystonecontext osapi_compute_app_v3

[filter:request_id]
paste.filter_factory = nova.openstack.common.middleware.request_id:RequestIdMiddleware.factory

[filter:compute_req_id]
paste.filter_factory = nova.api.compute_req_id:ComputeReqIdMiddleware.factory

[filter:faultwrap]
paste.filter_factory = nova.api.openstack:FaultWrapper.factory

[filter:noauth]
paste.filter_factory = nova.api.openstack.auth:NoAuthMiddleware.factory

[filter:noauth_v3]
paste.filter_factory = nova.api.openstack.auth:NoAuthMiddlewareV3.factory

[filter:ratelimit]
paste.filter_factory = nova.api.openstack.compute.limits:RateLimitingMiddleware.factory

[filter:sizelimit]
paste.filter_factory = nova.api.sizelimit:RequestBodySizeLimiter.factory

[app:osapi_compute_app_v2]
paste.app_factory = nova.api.openstack.compute:APIRouter.factory
该app直接调用了nova.api.openstack.compute中的APIRouter类中的factory函数。

[app:osapi_compute_app_v21]
paste.app_factory = nova.api.openstack.compute:APIRouterV21.factory

[app:osapi_compute_app_v3]
paste.app_factory = nova.api.openstack.compute:APIRouterV3.factory

[pipeline:oscomputeversions]
pipeline = faultwrap oscomputeversionapp

[app:oscomputeversionapp]
paste.app_factory = nova.api.openstack.compute.versions:Versions.factory

##########
# Shared #
##########

[filter:keystonecontext]
paste.filter_factory = nova.api.auth:NovaKeystoneContext.factory

[filter:authtoken]
paste.filter_factory = keystonemiddleware.auth_token:filter_factory
```
从上面 来看在OpenStack中nova-api的请求流程其实是由filter[compute_req_id faultwrap sizelimit authtoken keystonecontext ratelimit]组合而成的。这些也完全可以由用户自定义。
那么答案来了，对于上篇文章
《OpenStack添加跨域支持》我们可以采用另外一种更为优雅的方式去支持。
编写一个遵循WSGI协议的App，并且针对HTTP请求添加跨域参数在api-paste.ini中添加filter，并且重新组合composite 

## 编写APP
这里我已经写好一个简单的APP， https://github.com/coocla/openstack_cors；
app必须 是一个 callable object，接受两个参数(environ,start_response)，这是 paste 系统交给 application 的，符合 WSGI 规范的参数. app 需要完成的任务是响应envrion中的请求，准备好响应头和消息体，然后交给 start_response 处理，并返回响应消息体。
而filter函数除了它返回的是一个 filter 之外，和 app 工厂函数类似，fliter 是一个 把一个 WSGI app 作为唯一参数的可调用的对象，返回一个过滤后的 app。
参考wsgi的官方文档，修改response的HEADER信息：http://pylonsbook.com/en/1.1/the-web-server-gateway-interface-wsgi.html#changing-the-status-and-headers

```
# -*- coding:utf-8 -*-
import webob.exc

class CorsMixin(object):
    def __init__(self, application):
        self.application = application

    @classmethod
    def factory(cls, global_config, **local_config):
        def _factory(app):
            return cls(app, **local_config)
        return _factory

    def __call__(self, environ, start_response):
        if environ['REQUEST_METHOD'] == 'OPTIONS':
            return self._OPTIONS(environ, start_response)

        def custom_start_response(status, headers, exc_info=None):
            headers.append(('Access-Control-Allow-Origin', '*'))
            headers.append(('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, PATCH'))
            headers.append(('Access-Control-Allow-Headers', 'X-Auth-Token, Content-Type'))
            return start_response(status, headers, exc_info)
    
        return self.application(environ, custom_start_response)

    def _OPTIONS(self, environ, start_response):
        headers = [('Access-Control-Allow-Origin', '*'),
                   ('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, PATCH'),
                   ('Access-Control-Allow-Headers', 'X-Auth-Token, Content-Type')]
        resp = webob.exc.HTTPAccepted('Options Accept', headers)
        return resp(environ, start_response)
```