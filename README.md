Name
====

lua-resty-dynamic-balancer - Lua library for dynamic routing in OpenResty/ngx_lua

Table of Contents
=================

* [Name](#name)
* [Functions](#functions)
    * [init_host_upstream](#init_host_upstream)
    * [prepare](#prepare)
    * [action](#action)

Synopsis
========
```nginx
http {
    upstream foo {
        server 0.0.0.1;   # just an invalid address as a place holder
        balancer_by_lua_file /path/to/balancer.lua

        keepalive 10;  # connection pool
    }

    server {
        # this is the real entry point
        listen 80;

        location / {
            access_by_lua_file /path/to/access.lua;

            # make use of the upstream named "foo" defined above:
            proxy_pass  http://foo;
            proxy_next_upstream_tries   8;
            proxy_set_header    Host    $host;
            proxy_set_header    X-Real-IP   $remote_addr;

        }
    }
}
```

`access.lua`
```access.lua
local easy_balancer = require("easy_balancer")
local cluster_id = require("cluster_id")

if ngx.now() - easy_balancer.get_host_time(ngx.var.host) > 30 then
    local redis = require("resty.redis")
    local red = redis:new()
    red:set_timeout(3000)
    local ok, err = red:connect("127.0.0.1", 6379)
    if not ok then
        ngx.exit(501)
    end

    local res, err = red:hget(cluster_id, ngx.var.host)
    if not ok then
        ngx.exit(501)
    end

    if res == ngx.null then
        res = ""
    end

    easy_balancer.init_host_upstream(ngx.var.host, res)
end

local prepared = easy_balancer.prepare(ngx.var.host, ngx.var.uri)

if prepared == nil then
    ngx.exit(503)
end
ngx.ctx.prepared = prepared
```

`balancer.lua`
```
easy_balancer = require("easy_balancer");
easy_balancer.action(ngx.ctx.prepared)
```

Functions
=========

init_host_upstream
------------------
**syntax:** `easy_balancer.init_host_upstream(host, servers_string)`
*servers_string:*`"192.168.1.1； 192.168.1.2:8080; 192.168.1.3; 192.168.1.4:8080"`

prapare
-------
**syntax:** `prepared_request = easy_balancer.prepare(host, hash_key_string)`

action
------
**syntax:** `easy_balancer.action(host, prepared_request)`
