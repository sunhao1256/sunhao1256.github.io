---
title: "Nginx转发规则"
date: 2022-07-28T10:14:49+08:00
tags: ['Nginx']
---

# Location

| Syntax:  | `**location** [ = | ~ | ~* | ^~ ] *uri* { ... }` `**location** @*name* { ... }` |
| :------- | ------------------------------------------------------------ |
| Default: | —                                                            |
| Context: | `server`, `location`                                         |

Sets configuration depending on a request URI.

The matching is performed against a normalized URI, after decoding the text encoded in the “`%XX`” form, resolving references to relative path components “`.`” and “`..`”, and possible [compression](https://nginx.org/en/docs/http/ngx_http_core_module.html#merge_slashes) of two or more adjacent slashes into a single slash.

A location can either be defined by a prefix string, or by a regular expression. Regular expressions are specified with the preceding “`~*`” modifier (for case-insensitive matching), or the “`~`” modifier (for case-sensitive matching). To find location matching a given request, nginx first checks locations defined using the prefix strings (prefix locations). Among them, the location with the longest matching prefix is selected and remembered. Then regular expressions are checked, in the order of their appearance in the configuration file. The search of regular expressions terminates on the first match, and the corresponding configuration is used. If no match with a regular expression is found then the configuration of the prefix location remembered earlier is used.

`location` blocks can be nested, with some exceptions mentioned below.

For case-insensitive operating systems such as macOS and Cygwin, matching with prefix strings ignores a case (0.7.7). However, comparison is limited to one-byte locales.

Regular expressions can contain captures (0.7.40) that can later be used in other directives.

If the longest matching prefix location has the “`^~`” modifier then regular expressions are not checked.

Also, using the “`=`” modifier it is possible to define an exact match of URI and location. If an exact match is found, the search terminates. For example, if a “`/`” request happens frequently, defining “`location = /`” will speed up the processing of these requests, as search terminates right after the first comparison. Such a location cannot obviously contain nested locations.



> In versions from 0.7.1 to 0.8.41, if a request matched the prefix location without the “`=`” and “`^~`” modifiers, the search also terminated and regular expressions were not checked.



Let’s illustrate the above by an example:

> ```nginx
> location = / {
>     [ configuration A ]
> }
> 
> location / {
>     [ configuration B ]
> }
> 
> location /documents/ {
>     [ configuration C ]
> }
> 
> location ^~ /images/ {
>     [ configuration D ]
> }
> 
> location ~* \.(gif|jpg|jpeg)$ {
>     [ configuration E ]
> }
> ```

The “`/`” request will match configuration A, the “`/index.html`” request will match configuration B, the “`/documents/document.html`” request will match configuration C, the “`/images/1.gif`” request will match configuration D, and the “`/documents/1.jpg`” request will match configuration E.

The “`@`” prefix defines a named location. Such a location is not used for a regular request processing, but instead used for request redirection. They cannot be nested, and cannot contain nested locations.

If a location is defined by a prefix string that ends with the slash character, and requests are processed by one of [proxy_pass](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass), [fastcgi_pass](https://nginx.org/en/docs/http/ngx_http_fastcgi_module.html#fastcgi_pass), [uwsgi_pass](https://nginx.org/en/docs/http/ngx_http_uwsgi_module.html#uwsgi_pass), [scgi_pass](https://nginx.org/en/docs/http/ngx_http_scgi_module.html#scgi_pass), [memcached_pass](https://nginx.org/en/docs/http/ngx_http_memcached_module.html#memcached_pass), or [grpc_pass](https://nginx.org/en/docs/http/ngx_http_grpc_module.html#grpc_pass), then the special processing is performed. In response to a request with URI equal to this string, but without the trailing slash, a permanent redirect with the code 301 will be returned to the requested URI with the slash appended. If this is not desired, an exact match of the URI and location could be defined like this:

> ```nginx
> location /user/ {
>     proxy_pass http://user.example.com;
> }
> 
> location = /user {
>     proxy_pass http://login.example.com;
> }
> ```

# proxy_pass

| Syntax:  | `**proxy_pass** *URL*;`                      |
| :------- | -------------------------------------------- |
| Default: | —                                            |
| Context: | `location`, `if in location`, `limit_except` |

Sets the protocol and address of a proxied server and an optional URI to which a location should be mapped. As a protocol, “`http`” or “`https`” can be specified. The address can be specified as a domain name or IP address, and an optional port:

> ```
> proxy_pass http://localhost:8000/uri/;
> ```

or as a UNIX-domain socket path specified after the word “`unix`” and enclosed in colons:

> ```
> proxy_pass http://unix:/tmp/backend.socket:/uri/;
> ```



If a domain name resolves to several addresses, all of them will be used in a round-robin fashion. In addition, an address can be specified as a [server group](https://nginx.org/en/docs/http/ngx_http_upstream_module.html).

Parameter value can contain variables. In this case, if an address is specified as a domain name, the name is searched among the described server groups, and, if not found, is determined using a [resolver](https://nginx.org/en/docs/http/ngx_http_core_module.html#resolver).

A request URI is passed to the server as follows:

- If the `proxy_pass` directive is specified with a URI, then when a request is passed to the server, the part of a normalized

   request URI matching the location is replaced by a URI specified in the directive:

  > ```nginx
  > location /name/ {
  >     proxy_pass http://127.0.0.1/remote/;
  > }
  > ```

- If `proxy_pass` is specified without a URI, the request URI is passed to the server in the same form as sent by a client when the original request is processed, or the full normalized request URI is passed when processing the changed URI:

  > ```nginx
  > location /some/path/ {
  >     proxy_pass http://127.0.0.1;
  > }
  > ```

  > Before version 1.1.12, if `proxy_pass` is specified without a URI, the original request URI might be passed instead of the changed URI in some cases.

In some cases, the part of a request URI to be replaced cannot be determined:

- When location is specified using a regular expression, and also inside named locations.

  In these cases, `proxy_pass` should be specified without a URI.

- When the URI is changed inside a proxied location using the `rewrite` directive, and this same configuration will be used to process a request(`break`):

  > ```nginx
  > location /name/ {
  >     rewrite    /name/([^/]+) /users?name=$1 break;
  >     proxy_pass http://127.0.0.1;
  > }
  > ```

  In this case, the URI specified in the directive is ignored and the full changed request URI is passed to the server.

- When variables are used in `proxy_pass`:

  > ```nginx
  > location /name/ {
  >     proxy_pass http://127.0.0.1$request_uri;
  > }
  > ```

  In this case, if URI is specified in the directive, it is passed to the server as is, replacing the original request URI.

- 在nginx中配置proxy_pass时，当在后面的url加上了/，相当于是绝对根路径，则nginx不会把location中匹配的路径部分代理走;如果没有/，则会把匹配的路径部分也给代理走。 

首先是location进行的是模糊匹配

1. 没有“/”时，location /abc/def可以匹配/abc/defghi请求，也可以匹配/abc/def/ghi等
2. 而有“/”时，location /abc/def/不能匹配/abc/defghi请求，只能匹配/abc/def/anything这样的请求


```shell
下面四种情况分别用http://192.168.1.4/proxy/test.html 进行访问。

第一种：

location  /proxy/ {

proxy_pass http://127.0.0.1:81/;

}

结论：会被代理到http://127.0.0.1:81/test.html 这个url


第二种(相对于第一种，最后少一个 /)

location  /proxy/ {

proxy_pass http://127.0.0.1:81;

}

结论：会被代理到http://127.0.0.1:81/proxy/test.html 这个url


第三种：

location  /proxy/ {

proxy_pass http://127.0.0.1:81/ftlynx/;

}

结论：会被代理到http://127.0.0.1:81/ftlynx/test.html 这个url。


第四种(相对于第三种，最后少一个 / )：

location  /proxy/ {

proxy_pass http://127.0.0.1:81/ftlynx;

}

结论：会被代理到http://127.0.0.1:81/ftlynxtest.html 这个url
```

