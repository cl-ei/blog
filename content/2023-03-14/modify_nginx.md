---
available_keys: title, category, description, date, ref, author, tags: List
title: 修改编译nginx以删除或自定义 "Server" Header
tags: nginx
category: nginx
---
边缘计算等对内存极其敏感的场景下，在进行HTTP通信时往往需要移除一些不必要内容以节约内存。使用nginx正向代理时，它会添加默认的Server Header，即使在nginx.conf中指定了```server_tokens off;```，也仅仅是隐藏版本号，并未去除这个Header。

## 隐藏Header

这部分的逻辑在 ```src/http/ngx_http_header_filter_module.c``` 中定义:
```C
...
static u_char ngx_http_server_string[] = "Server: nginx" CRLF;
static u_char ngx_http_server_full_string[] = "Server: " NGINX_VER CRLF;
static u_char ngx_http_server_build_string[] = "Server: " NGINX_VER_BUILD CRLF;
...

static ngx_int_t
ngx_http_header_filter(ngx_http_request_t *r)
{
   ...
    if (r->headers_out.server == NULL) {
        if (clcf->server_tokens == NGX_HTTP_SERVER_TOKENS_ON) {
            len += sizeof(ngx_http_server_full_string) - 1;

        } else if (clcf->server_tokens == NGX_HTTP_SERVER_TOKENS_BUILD) {
            len += sizeof(ngx_http_server_build_string) - 1;

        } else {
            len += sizeof(ngx_http_server_string) - 1;
        }
    }
    ...
    b = ngx_create_temp_buf(r->pool, len);
    ...
    if (r->headers_out.server == NULL) {
        if (clcf->server_tokens == NGX_HTTP_SERVER_TOKENS_ON) {
            p = ngx_http_server_full_string;
            len = sizeof(ngx_http_server_full_string) - 1;

        } else if (clcf->server_tokens == NGX_HTTP_SERVER_TOKENS_BUILD) {
            p = ngx_http_server_build_string;
            len = sizeof(ngx_http_server_build_string) - 1;

        } else {
            p = ngx_http_server_string;
            len = sizeof(ngx_http_server_string) - 1;
        }

        b->last = ngx_cpymem(b->last, p, len);
    }
```
可以得知，在处理header时，nginx先计算header部分的内存大小，再申请一块buffer，然后进行填充，所以对```headers_out.server```进行了两次判断。可以对上述两个if判断块进行注释，这样在请求时就不会有server字段：
```shell
~/nginx # curl -I localhost/

HTTP/1.1 200 OK
Date: Wed, 15 Mar 2023 03:10:25 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Wed, 15 Mar 2023 02:55:32 GMT
Connection: keep-alive
ETag: "641133a4-267"
Accept-Ranges: bytes
```

## 使其可配置

在```ngx_http_header_filter```可以看到```clcf```结构体，类型为```ngx_http_core_loc_conf_t```，其为conf文件中http部分配置的解析逻辑：
```C
struct ngx_http_core_loc_conf_s {
    ngx_str_t     name;          /* location name */
    ngx_str_t     escaped_name;

#if (NGX_PCRE)
    ngx_http_regex_t  *regex;
#endif
    ...
    ngx_http_location_tree_node_t   *static_locations;
#if (NGX_PCRE)
    ngx_http_core_loc_conf_t       **regex_locations;
#endif
    ...
```
实现自定义server name，可以把配置添加到这个块当中:
```C
    // add
    ngx_str_t     customized_server_name;
```
同时要处理解析配置文件的地方，nginx从这里加载配置，并等待event，在event handler当中根据addr：port等参数寻找server的配置，并组装成request对象。conf文件解析的主要逻辑在```ngx_conf_handler```等函数当中，这里遍历每个ngx_command数组，每个元素对应一条解析项，配置的解析根据其属性进行对应的处理：
```C
struct ngx_command_s {
    ngx_str_t             name;
    ngx_uint_t            type;
    char               *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
    ngx_uint_t            conf;
    ngx_uint_t            offset;
    void                 *post;
};
```
其中，其中，`name`是配置项名称，`type`决定这个配置项可以在哪些块，nginx使用位标记来标识，`set`则为设置配置的回调方法。`conf`用于指示配置项所处内存的相对偏移位置，`offset`表示当前配置项在整个存储配置项的结构体中的偏移位置。由此，可以在```src/http/ngx_http_core_module.c```中定义自定义的command：
```C
static ngx_command_t  ngx_http_core_commands[] = {
    ...
    
    // add
    { ngx_string("customized_server_name"),
      NGX_HTTP_MAIN_CONF|NGX_CONF_TAKE1,
      ngx_conf_set_str_slot,
      NGX_HTTP_LOC_CONF_OFFSET,
      offsetof(ngx_http_core_loc_conf_t, customized_server_name),
      NULL },
    ...
    ngx_null_command
}
```
其中`ngx_conf_set_str_slot`是nginx预置的回调函数。若在多个域同时出现配置项时，nginx需要根据某些优先规则确定使用那个配置，nginx实现了一个merge机制来处理这个逻辑，对于没有merge函数的配置，最终不会生效。http域的merge定义在此文件```ngx_http_core_merge_loc_conf```函数中，在这里优先取子层存在的配置，若无则取上层的，否则取默认。可以添加上自定义配置项的merge函数：
```C
static char *
ngx_http_core_merge_loc_conf(ngx_conf_t *cf, void *parent, void *child)
{
    ngx_http_core_loc_conf_t *prev = parent;
    ngx_http_core_loc_conf_t *conf = child;
    ...
    // add
    ngx_conf_merge_str_value(conf->customized_server_name,
                             prev->customized_server_name, "undefined");
    ...
}
```
这样一来，就可以在`ngx_http_header_filter`当中获取到此配置项，修改这两处的逻辑：
```C
static ngx_int_t
ngx_http_header_filter(ngx_http_request_t *r)
{
    ...
    // modify
    if (r->headers_out.server == NULL) {
        if (clcf->server_tokens == NGX_HTTP_SERVER_TOKENS_ON) {
            len += sizeof("Server: ") - 1 + clcf->customized_server_name.len + 2;
        }
    }
    ...
    b = ngx_create_temp_buf(r->pool, len);
    ...
    
    // modify
    if (r->headers_out.server == NULL) {
        if (clcf->server_tokens == NGX_HTTP_SERVER_TOKENS_ON) {
            p = clcf->customized_server_name.data;
            len = clcf->customized_server_name.len;

            b->last = ngx_cpymem(b->last, "Server: ", sizeof("Server: ") - 1);
            b->last = ngx_cpymem(b->last, p, len);
            *b->last++ = CR; *b->last++ = LF;
        }
    }
    ...
```
需要注意```len += sizeof("Server: ") - 1 + clcf->customized_server_name.len + 2;```计算长度时，sizeof会包含字符串结尾的`\0`，拷贝到buffer的内容不包含这部分，故需减一。每条Header结束的`\r\n`占用2字节，故在末尾加2。

编译运行，并在`nginx.conf`中添加：
```
...
http {
    customized_server_name jjcjj;
    
    include       mime.types;
    default_type  application/octet-stream;
    ...
```
启动后，可以看到自定义的Server已经生效：
```shell
~/nginx # curl -I localhost/

HTTP/1.1 200 OK
Server: jjcjj
Date: Wed, 15 Mar 2023 03:10:25 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Wed, 15 Mar 2023 02:55:32 GMT
Connection: keep-alive
ETag: "641133a4-267"
Accept-Ranges: bytes
```
当未指定`customized_server_name`时，Server会返回默认值`undefined`；添加`server_tokens off`则不会返回Server Header，效果同上文所述。
