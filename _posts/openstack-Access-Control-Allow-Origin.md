title: OpenStack API添加跨域支持
categories: OpenStack
tags: [openstack, cross]
date: 2015-04-25 17:34:37
---
## 前言
OpenStack是目前一个比较火的开源的云计算管理平台项目，它是由几个主要的组件组合起来完成具体工作的，各个组件组件之间是由RESTFUL API基于消息队列互相通信的。
当前各大互联网公司也都无一例外的推出各自的公有或私有云平台，当然我们也不例外。openstack全套提供了很好很丰富的API，这使得进行二次开发将非常方便。但是问题也接踵而来，首先我们被跨域这个问题纠缠了。
## 什么是跨域？
通俗来说：如果在A网站中，我们希望使用Ajax来获得B网站中的特定内容，如果A网站与B网站不在同一个域中，那么就出现了跨域访问问题，出现跨域访问问题时，浏览器出于安全考虑，首先会向B网站的URL地址请求一个OPTIONS的请求，此时服务器端要回应给浏览器这些信息：允许谁跨域、允许哪些HTTP方法跨域、允许跨域请求时的头信息。浏览器在收到该信息后对比此时的ajax请求，全部都Match，正常发送。否则会抛出一个警告，并拦截掉本次的HTTP请求。<!--more-->
此时如果你还不明白什么是跨域，那么请通过这些权威信息来找答案。http://www.w3.org/TR/cors/http://en.wikipedia.org/wiki/Cross-origin_resource_sharing
## 如何支持跨域
为openstack中各个组件API添加代码，以支持跨域访问。
> 以下代码均以juno版本为例，省略掉中间的大量代码，请根据类和方法名定位代码添加位置。

### KeyStone
keystone/common/wsgi.py
```
class Middleware(Application):
    @webob.dec.wsgify()
    def __call__(self, request):
        if request.method == 'OPTIONS':
            return request.get_response(self._options)

    def _options(self, env, start_response):
        headers = [('Access-Control-Allow-Origin', '*'),
                   ('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, PATCH'),
                   ('Access-Control-Allow-Headers', 'X-Auth-Token, Content-type')]
        resp = webob.exc.HTTPAccepted('Options Accept', headers)
        return resp(env, start_response)


def render_response(body=None, status=None, headers=None, method=None):


    if hasattr(resp, 'headers'):
        resp.headers["Access-Control-Allow-Origin"] = "*"
        resp.headers["Access-Control-Allow-Methods"] = "GET, POST, PUT, DELETE, PATCH"
        resp.headers["Access-Control-Allow-Headers"] = "X-Auth-Token, Content-Type"
    return resp
```
### nova-api
nova/api/openstack/init.py
```
class FaultWrapper(base_wsgi.Middleware):
    @webob.dec.wsgify(RequestClass=wsgi.Request)
        if req.environ['REQUEST_METHOD'] == 'OPTIONS':
            return req.get_response(self._options)
        ......

    def _options(self, env, start_response):
        headers = [('Access-Control-Allow-Origin', '*'),
                   ('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, PATCH'),
                   ('Access-Control-Allow-Headers', 'X-Auth-Token, Content-type')]
        resp = webob.exc.HTTPAccepted('Options Accept', headers)
        return resp(env, start_response)
```
nova/api/openstack/wsgi.py
```
class Resource(wsgi.Application):
    def _process_stack(self, request, action, action_args,
                       content_type, body, accept):
        if hasattr(response, 'headers'):
            for hdr, val in response.headers.items():
                # Headers must be utf-8 strings
                response.headers[hdr] = utils.utf8(str(val))
            response.headers['Access-Control-Allow-Origin'] = '*'
            response.headers['Access-Control-Allow-Methods'] = 'GET, POST, PUT, DELETE, PATCH'
            response.headers['Access-Control-Allow-Headers'] = 'X-Auth-Token, Content-Type'
        return response
```
### Cinder-API
cinder/api/middleware/fault.py
```
class FaultWrapper(base_wsgi.Middleware):
    @webob.dec.wsgify(RequestClass=wsgi.Request)
    def __call__(self, req):
        if req.environ['REQUEST_METHOD'] == 'OPTIONS':
            return req.get_response(self._options)
            
    def _options(self, env, start_response):
        headers = [('Access-Control-Allow-Origin', '*'),
                   ('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, PATCH'),
                   ('Access-Control-Allow-Headers', 'X-Auth-Token, Content-type')]
        resp = webob.exc.HTTPAccepted('Options Accept', headers)
        return resp(env, start_response)
```
cinder/api/openstack/wsgi.py
```
class Resource(wsgi.Application):
    def _process_stack(self, request, action, action_args,
                       content_type, body, accept):

        if hasattr(response, 'headers'):
            response.headers['Access-Control-Allow-Origin'] = '*'
            response.headers['Access-Control-Allow-Methods'] = 'GET, POST, PUT, DELETE, PATCH'
            response.headers['Access-Control-Allow-Headers'] = 'X-Auth-Token, Content-Type'
        return response
```
### Glance-API
glance/common/wsgi.py
```
class Middleware(object):
    @webob.dec.wsgify
    def __call__(self, req):
        if req.environ['REQUEST_METHOD'] == 'OPTIONS':
            return self.process_response(self._options)
    def _options(self, env, start_response):
        headers = [('Access-Control-Allow-Origin', '*'),
                   ('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, PATCH'),
                   ('Access-Control-Allow-Headers', 'X-Auth-Token, Content-type')]
        resp = webob.exc.HTTPAccepted('Options Accept', headers)
        return resp(env, start_response)


class Resource(object):
    @webob.dec.wsgify(RequestClass=Request)
    def __call__(self, request):
        try:
            response = webob.Response(request=request)
            self.dispatch(self.serializer, action, response, action_result)

            if hasattr(response, 'headers'):
                response.headers['Access-Control-Allow-Origin'] = '*'
                response.headers['Access-Control-Allow-Methods'] = 'GET, POST, PUT, DELETE, PATCH'
                response.headers['Access-Control-Allow-Headers'] = 'X-Auth-Token, Content-Type'
            return response
        except webob.exc.WSGIHTTPException as e:
            return translate_exception(request, e)
        except webob.exc.HTTPException as e:
            return e
        # return unserializable result (typically a webob exc)
        except Exception:
            return action_result
```
### Neutron-API
neutron/openstack/common/middleware/request_id.py
```
class RequestIdMiddleware(base.Middleware):
    def _options(self, env, start_response):
        headers = [('Access-Control-Allow-Origin', '*'),
                   ('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE'),
                   ('Access-Control-Allow-Headers', 'X-Auth-Token, Content-Type')]
        resp = webob.exc.HTTPAccepted('Options Accept', headers)
        return resp(env, start_response)

    @webob.dec.wsgify
    def __call__(self, req):
        if req.environ['REQUEST_METHOD'] == 'OPTIONS':
            return self.process_response(self._options)

        req_id = context.generate_request_id()
        req.environ[ENV_REQUEST_ID] = req_id
        response = req.get_response(self.application)

        if HTTP_RESP_HEADER_REQUEST_ID not in response.headers:
            response.headers.add(HTTP_RESP_HEADER_REQUEST_ID, req_id)
        if hasattr(response, "headers"):
            response.headers['Access-Control-Allow-Origin'] = '*'
            response.headers['Access-Control-Allow-Methods'] = 'GET, POST, PUT, DELETE'
            response.headers['Access-Control-Allow-Headers'] = 'X-Auth-Token, Content-Type'
        return response
```
在添加以上代码后，分别重启对应的API即可让代码生效。添加时注意缩进。
```
# systemctl restart openstack-keystone
# systemctl restart openstack-nova-api
# systemctl restart openstack-cinder-api
# systemctl restart openstack-glance-api
# systemctl restart neutron-server
```



</br>