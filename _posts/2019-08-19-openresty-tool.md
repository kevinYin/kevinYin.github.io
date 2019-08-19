---
layout: post  
title:  "利用openresty写的小工具"  
date:   2019-08-19 00:30  
categories: server  
permalink: /Priest/openresty-tool

---

## 背景  
公司内部有一个api网关服务janus，用于拦截所有的流量，并进行鉴权、防刷以及协议转换等，这里的协议转换比如http转RPC，通过网关解析http请求并且通过网关服务后台的配置找到对应的RPC服务进行调用，从而实现前端接口直接调用RPC服务。在供应链很多系统都是通过这种方式实现前端代码来实现RPC服务的调用。

## 问题
但在开发在进行开发时候会遇到一个问题，就是几十号人一起开发的时候，大家如果想通过本地前端http连接自己的分支的接口，就需要自己部署一套janus网关，并且指定服务调用到自己RPC服务。一来是janus的部署成本高，二是janus还会传递cookie信息给RPC服务，开发测试的时候需要快速切换cookie比较麻烦。  

## 分析
基于上述的问题，我发现本质上，只是需要一个可以按照janus网关的配置规则对http请求进行一个转换即可，并且可以快速切换cookie的工具。另外了解到的信息是，内部的RPC服务启动后会开启一个基于jetty的restful服务，可以通过rest方式在本地调用rpc，但是协议与前端的本地调用有差异，因此前端的http请求需要经过转换。  

## 解决思路 
第一时间就想到一个比较合适的工具：**openresty**，是一款基于 NGINX 和 LuaJIT 的 Web 平台，openresty可以用来做很多很酷的事情，但是在这里，我用的是将所有的前端调用RPC的http请求全部转移到本地的openresty，在openresty里对请求body做修改和 URL做重写，最后再将结果给按照网关的响应协议组装返回给前端即可  
总结上面的问题，我列下写一个这样的工具，需要实现的点就是：  
> 1.前端的http请求参数做转化，让本地的RPC服务的rest接口可以正确接收   
> 2.cookie可以随时修改  
> 3.封装响应结果以符合网关服务的封装结果格式  

## 实现  

### openresty的执行阶段概念  
一个请求经过openresty到完成会经过以下几个阶段（copy from moonbingbing.gitbooks.io）：   
1. set\_by\_lua*: 流程分支处理判断变量初始化  
2. rewrite\_by\_lua*: 转发、重定向、缓存等功能(例如特定请求代理到外网)  
3. access\_by\_lua*: IP 准入、接口权限等情况集中处理(例如配合 iptable 完成简单防火墙)  
4. content\_by\_lua*: 内容生成  
5. header\_filter\_by\_lua*: 响应头部过滤处理(例如添加头部信息)  
6. body\_filter\_by\_lua*: 响应体过滤处理(例如完成应答内容统一成大写)  
7. log\_by\_lua*: 会话完成后本地异步完成日志记录(日志可以记录在本地，还可以同步到其他机器  

很容易地我们实现就是从 **access\_by\_lua** 和 **body\_filter\_by\_lua**2个阶段进行拦截处理，由于逻辑比较多，我是通过lua 文件来实现，所以也就是 **access\_by\_lua\_file** 和 **body\_filter\_by\_lua\_file**.

### nginx.conf
入口在nginx.conf文件，配置如下：  

```
server {
        listen       80;
        server_name  localhost;

        lua_need_request_body on;

        location / {
        
            set $newURL '';

            access_by_lua_file lua_code/rebuild_req.lua;

            proxy_pass $newURL;

            body_filter_by_lua_file lua_code/resp.lua;
        }
    }
```

**newURL:** 是组装后的请求要转发到本地的OSP restful接口路径  
**rebuild_req.lua：** 实现了将 request body重新组装，cookie设置，URL组装  
**resp.lua：**实现将response body按照网关服务的返回格式进行封装


### access\_by\_lua\_file：rebuild_req.lua
1.获取请求包体   

```
-- 需要声明
ngx.req.read_body()  
local function getRequestData()
    local data = ngx.req.get_body_data()
    if nil == data then
        local file_name = ngx.req.get_body_file()
        if file_name then
            data = getFile(file_name)
        end
    end
    return data;
end
```
2.设置cookie   

```
-- cookies做成在配置文件，方便随时修改
local function setCookie()
    local cookieFile = FileRead("D:\\cookie.json");  
    print("cookies内容", cookieFile);
    local json = cjson.decode(cookieFile); 
    -- 设置osp cookis
    ngx.header['Set-Cookie'] = json
end
```

3.解析网关配置json文件，获取构建新的request body的信息

```
local function getServiceAndMethod(fileArr)
    local serviceVersion
    local outboundMethod
    for i, v in pairs(fileArr) do  
        local janusBody = FileRead(v);
        local janusJson = cjson.decode(janusBody);
        local outboundVersion = ''
        for i, w in ipairs(janusJson) do  
            if ngx.var.request_uri == w.inboundService then
                for j,c in ipairs(w.serviceList) do
                    outboundMethod = c.outboundMethod;
                    serviceVersion = string.format('%s-%s', c.outboundService, c.outboundVersion)
                    break
                end
                break
            end
        end
        if serviceVersion then
            return serviceVersion,outboundMethod
        end
    end
end
```
4.请求URL修改   

``` 
-- 匹配URL获取 proxy_pass
local proxy = 'http://127.0.0.1:8988/AskRestServlet?'

-- 构建请求链接
ngx.var.newURL = proxy.."methodName="..outboundMethod.."&&format=JSON&&serviceName="..serviceVersion.."&&requestContent=" .. reqData.."&&timeout=50000"
```

### body\_filter\_by\_lua\_file ：resp.lua
修改响应参数，主要把response body放在 {'data': $repsonse_body, 'code':200}
  
```
local function setResp()
    local headers = ngx.header
    local json_util = require("cjson")
    
    local function res_json()
        local res = {}
        res["data"] = ngx.ctx.buf
        res["code"] = "200"
        return json_util.encode(res)
    end
    
    local status = ngx.var.status
    local chunk, eof = ngx.arg[1], ngx.arg[2]  -- 获取当前的流 和是否时结束
    local info = ngx.ctx.buf
    chunk = chunk or ""
    if info then-- 这个可以将原本的内容记录下来
    else
        ngx.ctx.buf = chunk
    end
    if eof then
        ngx.ctx.buffered = nil
        if status == 413 or status == "413" then  
            ngx.arg[1] = res_json() 
        else
            ngx.arg[1] = res_json()
        end
    else
        ngx.arg[1] = nil
    end 
end
```

## 最后  
openresty的作者写了很多module，但是在windows上没法使用，所以在写得过程折腾了不少，由于公司内部的限制只能在windows上使用，所以很多开源的模块都无法使用。工具开发的时间比较短，不是那么完善，后续再陆续实践中改进。  
感谢openresty的作者章亦春。