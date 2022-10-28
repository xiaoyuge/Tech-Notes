动态Nginx负载均衡的配置，可以通过Consul+Consul-Template方式，但是这种方案有个缺点：每次发现配置变更都需要reload Nginx，而reload是有一定损耗的。而且，如果你需要长连接支持的话，那么当reload时Nginx长连接所在worker进程会进行优雅退出，并当该worker进程上的所有连接都释放时，进程才真正退出（表现为worker进程处于worker process is shutting down）。因此，如果能做到不reload就能动态更改upstream，那么就完美了。

目前的开源解决方法有3种：

1、Tengine的Dyups模块

2、微博的Upsync模块+Consul

3、使用OpenResty的balancer_by_lua。
这里使用的是Consul+OpenResty 来实现动态负载均衡。Consul的安装这里将不再介绍。
OpenResty简介

OpenResty 是一个基于 Nginx 与 Lua 的高性能 Web 平台，其内部集成了大量精良的 Lua 库、第三方模块以及大多数的依赖项。用于方便地搭建能够处理超高并发、扩展性极高的动态 Web 应用、Web 服务和动态网关。
Lua简介

Lua是一个简洁、轻量、可扩展的程序设计语言，其设计目的是为了嵌入应用程序中，从而为应用程序提供灵活的扩展和定制功能。Lua由标准C编写而成，代码简洁优美，几乎在所有操作系统和平台上都可以编译，运行。
如何做到动态呢?

正常使用Nginx作为API网关, 需要如下配置, 当加后端实例时, reload一下Nginx, 使其生效
upstream backend {
    server 192.168.0.1;
    server 192.168.0.2;
}
那么只要让upstream变成动态可编程就OK了, 当新增后端实例时, 无需reload, 自动生效。

在OpenResty通过长轮训和版本号及时获取Consul的kv store变化。Consul提供了time_wait和修改版本号概念，如果Consul发现该kv没有变化就会hang住这个请求5分钟，在这５分钟内如果有任何变化都会及时返回结果。通过比较版本号我们就知道是超时了还是kv的确被修改了。

Consul的node和service也支持阻塞查询，相对来说用service更好一点，毕竟支持服务的健康检查。阻塞api和kv一样，加一个index就好了。
OpenResty安装

笔者使用的是MAC，下面重点介绍MAC系统上的操作。
Mac
brew tap openresty/brew
brew install openresty
Linux
sudo yum install yum-utils
sudo yum-config-manager --add-repo https://openresty.org/package/centos/openresty.repo

sudo yum install openresty

#命令行工具 resty
sudo yum install openresty-resty
Mac OS系统上安装完之后Nginx的配置文件在目录/usr/local/etc/openresty，启动命令在目录/usr/local/opt/openresty/nginx/sbin。
启动
openresty
在浏览器输入localhost


openresty -h



到这里OpenResty就已经安装好了。

重启
openresty -s reload
安装Lua的包管理器LuaRocks

LuaRocks是Lua模块的包管理器，安装LuaRocks之后可以通过LuaRocks来快速安装Lua的模块，官网地址是https://luarocks.org/。

进入OpenResty的安装目录/usr/local/opt/openresty可以看到已经安装了luajit，也是OpenResty默认使用的lua环境。



下载LuaRocks安装包http://luarocks.github.io/luarocks/releases/luarocks-3.0.4.tar.gz，解压之后进入LuaRocks源码目录编译安装到OpenResty的安装目录。
./configure --prefix=/usr/local/opt/openresty/luajit --with-lua=/usr/local/opt/openresty/luajit --lua-suffix=luajit --with-lua-include=/usr/local/opt/openresty/luajit/include/luajit-2.1
make build
make install
安装完LuaRocks之后，可以看到LuaRocks的执行命令在目录/usr/local/opt/openresty/luajit

安装完LuaRocks之后就可以安装搭建环境需要的依赖包了。
./luarocks install luasocket
修改配置文件

在Consul注册好服务之后通过Consul的http://127.0.0.1:8500/v1/catalog/service/api_tomcat2接口就可以获取到服务的列表。这里不再讲述Consul的服务注册。

在默认配置文件目录/usr/local/etc/openresty创建一个servers文件夹来放新的配置文件，创建lualib文件夹来放lua脚本，修改配置文件nginx.conf，添加include servers/*.conf;。

在lualib文件夹下创建脚本upstreams.lua
local http = require "socket.http"
local ltn12 = require "ltn12"
local cjson = require "cjson"

local _M = {}

_M._VERSION="0.1"

function _M:update_upstreams()
    local resp = {}

    http.request{
        url = "http://127.0.0.1:8500/v1/catalog/service/api_tomcat2", sink = ltn12.sink.table(resp)
    }
       
        local resp = table.concat(resp);
    local resp = cjson.decode(resp);

    local upstreams = {}
    for i, v in ipairs(resp) do
        upstreams[i] = {ip=v.Address, port=v.ServicePort}
    end
        
    ngx.shared.upstream_list:set("api_tomcat2", cjson.encode(upstreams))
end

function _M:get_upstreams()
   local upstreams_str = ngx.shared.upstream_list:get("api_tomcat2");
   local tmp_upstreams = cjson.decode(upstreams_str);
   return tmp_upstreams;
end

return _M
通过luasockets查询Consul来发现服务，update_upstreams用于更新upstream列表，get_upstreams用于返回upstream列表，此处可以考虑worker进程级别的缓存，减少因为json的反序列化造成的性能开销。

还要注意使用的luasocket是阻塞API，这可能会阻塞我们的服务，使用时要慎重。

创建文件test_openresty.conf
lua_package_path "/usr/local/etc/openresty/lualib/?.lua;;";
lua_package_cpath "/usr/local/etc/openresty/lualib/?.so;;";

lua_shared_dict upstream_list 10m;

# 第一次初始化
init_by_lua_block {
    local upstreams = require "upstreams";
    upstreams.update_upstreams();
}

# 定时拉取配置
init_worker_by_lua_block {
    local upstreams = require "upstreams";
    local handle = nil;

    handle = function ()
        --TODO:控制每次只有一个worker执行
        upstreams.update_upstreams();
        ngx.timer.at(5, handle);
    end
    ngx.timer.at(5, handle);
}

upstream api_server {
    server 0.0.0.1 down; #占位server

    balancer_by_lua_block {
        local balancer = require "ngx.balancer";
        local upstreams = require "upstreams";    
        local tmp_upstreams = upstreams.get_upstreams();
        local ip_port = tmp_upstreams[math.random(1, table.getn(tmp_upstreams))];
        balancer.set_current_peer(ip_port.ip, ip_port.port);
    }
}

server {
    listen       8000;
    server_name  localhost;
    charset utf-8;
    location / {
         proxy_pass http://api_server;
         access_log  /usr/local/etc/openresty/logs/api.log  main;
    }
}
init_worker_by_lua是每个Nginx Worker进程都会执行的代码，所以实际实现时可考虑使用锁机制，保证一次只有一个人处理配置拉取。另外ngx.timer.at是定时轮询，不是走的长轮询，有一定的时延。有个解决方案，是在Nginx上暴露HTTP API，通过主动推送的方式解决。

Agent可以长轮询拉取，然后调用HTTP API推送到Nginx上，Agent可以部署在Nginx本机或者远程。

对于拉取的配置，除了放在内存里，请考虑在本地文件系统中存储一份，在网络出问题时作为托底。
获取upstream列表，实现自己的负载均衡算法，通过ngx.balancer API进行动态设置本次upstream server。通过balancer_by_lua除可以实现动态负载均衡外，还可以实现个性化负载均衡算法。
可以使用lua-resty-upstream-healthcheck模块进行健康检查，后面会单独介绍。

到这里，其实还有一个问题没有解决掉，虽然upstream中的占位server是下线的，但是nginx在检测upstream列表中server的健康状态的时候是会去检测这个占位server的，最好的方式还是在启动之后把它彻底从upstream列表中给移除掉。
