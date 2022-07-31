## Nginx

### Request body and Request Parameter

RequestBody主要用来接收前端传递给后端的json字符串中的数据的(请求体中的数据的)；
GET方式无请求体，所以使用RequestBody接收数据时，不能使用GET方式提交数据，要用POST方式进行提交。
在后端的同一个接收方法里，RequestBody与RequestParam()可以同时使用，RequestBody最多只能有一个，而RequestParam可以有多个。

注：一个请求，只有一个RequestBody；一个请求，可以有多个RequestParam。


### client header config

#### client_header_buffer_size

```bash
client_header_buffer_size
Syntax: client_header_buffer_size size;
Default: client_header_buffer_size 1k;
Context: http, server
```

假设client_header_buffer_size的配置为1k，如果(请求行+请求头)的大小如果没超过1k，放行请求。如果(请求行+请求头)的大小如果超过1k，则以large_client_header_buffers配置为准

#### large_client_header_buffers

```bash
large_client_header_buffers
Syntax: large_client_header_buffers number size;
Default: large_client_header_buffers 4 8k;
Context: http, server
```

假设large_client_header_buffers的配置为`4 8k`，则对请求有如下要求

1. 请求行(request line)的大小不能超过8k，否则返回414错误
2. 请求头(request header)中的每一个头部字段的大小不能超过8k，否则返回400错误（实际是494错误，但nginx统一返回400了）`curl -H "header1=aaa" -H "header2=bbb" -v http://127.0.0.1/`，这里的header1=xxx和header2=xxx就是请求头中的头部字段
3. (请求行+请求头)的大小不能超过32k(4 * 8k)

#### 总结

对nginx处理header时的方法，学习后理解如下：

1. 先处理请求的request_line，之后才是request_header。
2. 这两者的buffer分配策略相同。
3. 先根据client_header_buffer_size配置的值分配一个buffer，如果分配的buffer无法容纳 request_line/request_header，那么就会再次根据large_client_header_buffers配置的参数分配large_buffer，如果large_buffer还是无法容纳，那么就会返回414（处理request_line）/400（处理request_header）错误。

根据对手册的理解，我理解这两个指令在配置header_buffer时的使用场景是不同的，个人理解如下：

1. 如果你的请求中的header都很大，那么应该使用client_header_buffer_size，这样能减少一次内存分配。
2. 如果你的请求中只有少量请求header很大，那么应该使用large_client_header_buffers，因为这样就仅需在处理大header时才会分配更多的空间，从而减少无谓的内存空间浪费。

client_header_buffer_size的最终值，是nginx在解析conf后，default_server中经过merge的最终值。





### client body config

#### client_body_buffer_size

这个值有必要大一点，如果request_body内容每读到，影响日志收集和问题排查

Context: http, server, and location

Specifies the size of the buffer holding the body of client requests. If this size is exceeded, the body (or at least part of it) will be written to the disk. Note that, if the client_body_in_file_only directive is enabled, request bodies are always stored to a file on the disk, regardless of their size (whether they fit in the buffer or not).

Syntax: Size value

Default value: 8k or 16k (2 memory pages) depending on your computer architecture

但是有个问题

```bash
# Linux上查看memory pages是4K
$ getconf PAGE_SIZE
4096
```

如下栗子：

如下，request_length请求的字节数（包括请求行、请求头和请求体）是15132 ，大于8K，body内容写在tmp文件中了，没在内存里，所以读不到信息。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/request-body-is-empty.jpg)

如下，17080字节，request_body为空

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/body-size-1.jpg)

将client_body_buffer_size修改为32K

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/body-size-2.jpg)

#### client_max_body_size

client_max_body_size 默认 1M，表示 客户端请求服务器最大允许大小，在“Content-Length”请求头中指定。如果请求的正文数据大于client_max_body_size，HTTP协议会报错 413 Request Entity Too Large。就是说如果请求的正文大于client_max_body_size，一定是失败的。如果需要上传大文件，一定要修改该值。

##### client_body_temp

client_body_temp 指定的路径中，默认该路径值是/tmp/.
所以配置的client_body_temp地址，一定让执行的Nginx的用户组有读写权限。否则，当传输的数据大于client_body_buffer_size，写进临时文件失败会报错。
这个问题我们遇到过。

```bash
20648 open() "/usr/local/openresty-1.9.7.5/nginx/client_body_temp/0000000019" failed (13: Permission denied)
```

/usr/local/openresty-1.9.7.5/nginx/client_body_temp/这个文件夹权限改为执行Nginx的用户群组就可以解决。
在这个问题上和语言就相关了，如果使用的是PHP，PHP会自己将临时文件读取出来，放置到请求数据里面，这是没有问题的，开发者也不需要关心。肯定是完整的数据。
如果使用的openresty lua 开发的话，就需要开发者自己读取出来，让后续的逻辑使用。

                function getFile(file_name)
                    local f = assert(io.open(file_name, 'r'))
                    local string = f:read("*all")
                    f:close()
                    return string
                end
    
                ngx.req.read_body()
                local data = ngx.req.get_body_data()
                if nil == data then
                    local file_name = ngx.req.get_body_file()
                    ngx.say(">> temp file: ", file_name)
                    if file_name then
                        data = getFile(file_name)
                    end
                end
    
                ngx.say("hello ", data)


#### 总结

传输的数据大于client_max_body_size，一定是传不成功的。小于client_body_buffer_size直接在内存中高效存储。如果大于client_body_buffer_size小于client_max_body_size会存储临时文件，临时文件一定要有权限。
如果追求效率，就设置 client_max_body_size client_body_buffer_size相同的值，这样就不会存储临时文件，直接存储在内存了。

### 转载

1. https://zhuanlan.zhihu.com/p/35391236
2. https://www.cnblogs.com/Small-sunshine/p/13914034.html
3. https://blog.csdn.net/qq_44099797/article/details/121186389
4. https://blog.csdn.net/zzhongcy/article/details/122844605