# openresty-emc-atmos-proxy

## Overview

This configuration proxies requests to the /rest/objects API of EMC ATMOS and adds in the X-EMC headers for authentication/authorization. Use Nginx with LUA or OpenResty or Tengine with LUA.
The server block in the configuration is for port 80 but could also be extended for HTTPS.

## Configuration

To use change the "atmos_user", "atmos_key", and the "@atmos" block to point to your ATMOS instance.

## RAW

```
worker_processes  24;
events { 
     worker_connections  1024; 
}

http {
    access_log off;
    proxy_redirect off;
    include       mime.types;
    default_type  application/json;
    client_max_body_size 200M;
    sendfile        on;
    keepalive_timeout  65;

    server {
        listen       80;

        location ~* /rest/objects {
                set_formatted_gmt_time $timestr "%a, %d %b %Y %H:%M:%S GMT";
                more_clear_headers 'Date*';
                content_by_lua '
                local atmos_user = "xxxxxxxxxxxx/user" # <--- atmos user
                local atmos_key = "xxxxxxxxxxxxxx" # <--- atmos user key
                ngx.req.set_header("x-emc-uid", atmos_user)
                ngx.req.set_header("x-emc-date", ngx.var.timestr)
                local h = ngx.req.get_headers()
                local atmos_args, atmos_range = "", ""
                if (ngx.var.args) then
                        atmos_args = "?"..ngx.var.args
                end
                if h["Range"] ~= nil then
                        atmos_range = h["Range"]
                end
                local str = ngx.req.get_method().."\\n"..h["Content-Type"].."\\n"..atmos_range.."\\n\\n"..ngx.var.uri..atmos_args.."\\nx-emc-date:"..h["x-emc-date"].."\\nx-emc-uid:"..h["x-emc-uid"]
                local digest = ngx.encode_base64(ngx.hmac_sha1(ngx.decode_base64(atmos_key), str))
                ngx.req.set_header("x-emc-signature", digest)
                ngx.exec("@atmos");
                ';
        }
        location @atmos {
                proxy_pass_request_headers on;
                proxy_pass http://x.x.x.x; # <--- atmos IP
        }
    }
}
