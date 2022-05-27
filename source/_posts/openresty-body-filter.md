---
layout: post
title: "OpenResty 使用body_filter_by_lua* 修改返回内容"
date: 2021-04-29
excerpt: "不过初看下来，很难将 `echo hello world`; 和 `delivered in chunks` 联系起来。这个 `chunk` 的大小是怎么确定的？看例子，应该跟 `echo/ngx.say` 这一类输出方式有关。但是会不会跟输出的大小也有关？如果我一次性 `ngx.say` 了很多内容，是否会分成多个 `chunks`发送？如果响应来自上游服务器，`chunks` 的数目又怎么定？"
categories: 
- OpenResty
tags: 
- OpenResty
comments: true
---

#### 背景

测试开发环境所有前端页面需要嵌入一段前端脚本，方便开发者在手机浏览器里调试，类似小程序调试的功能。


#### 谈谈 `body_filter_by_lua*` 多次调用的问题

正如 OpenResty 文档中指出，body_filter_by_lua* 可能会在一次请求中多次调用。

>Nginx output filters may be called multiple times for a single request because response body may be delivered in chunks.
Thus, the Lua code specified by in this directive may also run multiple times in the lifetime of a single HTTP request.

文档中举了个例子：

```
location /t {
  echo hello world;
  echo hiya globe;
```

`body_filter_by_lua*` 首次调用时，`ngx.arg[1]` 的值只是 `hello world`，不包括下面的 `hiya globe`。

不过初看下来，很难将 `echo hello world`; 和 `delivered in chunks` 联系起来。这个 `chunk` 的大小是怎么确定的？看例子，应该跟 `echo/ngx.say` 这一类输出方式有关。但是会不会跟输出的大小也有关？如果我一次性 `ngx.say` 了很多内容，是否会分成多个 `chunks`发送？如果响应来自上游服务器，`chunks` 的数目又怎么定？

要回答这个问题，需要看看 `Nginx` 内响应内容的组织方式。`Nginx` 上游产生的内容，存储为 `ngx_chain_t` 类型的数据。这其实是一条 `ngx_buf_t`链表。很容易可以想像到，这个链表就代表着数据流。上游产生的内容，像流水线上的包裹一样，不停地向下游传递。`output filter` 阶段像流水线上的机器，处理这些“包裹”。跟流水线上的机器不同的是，`Nginx` 中的 `output filter` 并非逐个处理这些“包裹”，而是一批一批地处理。上游成批成批地生产出这些包裹，每批包裹构成 `ngx_chain_t` 的子串，而 `output fiter`则遍历这一子串，把其中的每个包裹打开处理。

想到 `body_filter_by_lua*` 其实属于 `output filter` 的一种，我们就回到了一开始讨论的问题。既然 `body_filter_by_lua*`是一批一批处理上游的响应，那么它的调用次数就取决于上游的响应次数。上游的一次响应，如一次 `ngx.say`，会产生一个 `ngx_chain_t` 的子串（就 `ngx.say` 而言，这个子串仅包含单个 `ngx_buf_t`）。至于响应的大小，最多只会影响到子串的长短，具体情况则取决于具体实现。

以我们常用的 ngx.say 为例:

```
ngx.say('This ', 'will ', 'be ', 'in ', 'a ', 'buffer')
```

以上几个字符串会通过栈从 `lua` 域传递给 C 域。接着 `OpenResty` 计算它们的总长度，从 `buffer chain` 中找出一个空闲的大小合适的 ngx_buf_t，把它们拷贝进来。 之后就走 `http_output_filter` 把这个 `ngx_buf_t` （准确来说，是它所在的链表）发送出去。

那么，上游什么时候会把数据发完了？`Nginx` 采用了一个 `last_buf` 的标志位，如果某个 `ngx_buf_t` 是链表中的最后一个，跟上游交互的模块会设置这一个标志位为1. 映射回 `OpenResty` 的 `lua` 域，则是 `body_filter_by_lua*` 中的 `ngx.arg[2]`。你可能会注意到，`last_buf` 是一个设置在 `ngx_buf_t` 上的标志位，而传递给 `output filter` 的是 `ngx_chain_t`。`OpenResty` 把这一差别隐藏在实现之下——它会遍历当前输入的子串，如果某个 `ngx_buf_t` 存在 `last_buf`，那么就返回 true。

#### 自行尝试

```
-- access_by_lua_file：
ngx.say("123")
ngx.say("456")

-- body_filter_by_lua_file:
local chunk, eof = ngx.arg[1], ngx.arg[2]
print(chunk, eof)

-- 结果
-- body_filter_by_lua*首次调用时：123 false
-- body_filter_by_lua*第二次调用时：456 false
-- body_filter_by_lua*第三次调用时: 空 true
```

尽管这里只有两个 `echo`，但是 `body_filter_by_lua*` 会被调用三次！第三次调用的时候，`ngx.arg[1]` 为空字符串，而 `ngx.arg[2]`为 `true`。这是因为 `Nginx` 的 `upstream` 相关模块，以及 `OpenResty` 的 `content_by_lua`，会单独发送一个设置了 `last_buf` 的空 `buffer` ，来表示流的结束。所以我们需要在运行相关逻辑之前，检查 `ngx.arg[1]` 是否为空，但是需要注意的是 `ngx.arg[2]=true` 并不代表 `ngx.arg[1]`一定为空。

也许你已经发现了，子请求也会走到 `body_filter_by_lua*` 的流程。严格意义上，如果只希望 `body_filter_by_lua*` 修改响应给客户端的内容，需要额外用 `ngx.is_subrequest` 判断下：

```
if ngx.arg[1] and not ngx.is_subrequest then
```


官方文档也有一段说明，源码链接地址:[body_filter_by_lua ](https://github.com/openresty/lua-nginx-module#body_filter_by_lua)

>The input data chunk is passed via ngx.arg[1] (as a Lua string value) and the "eof" flag indicating the end of the response body data stream is passed via ngx.arg[2] (as a Lua boolean value).

流每次的内容输出在 `ngx.arg[1]` 中； `eof` 最后的标记在 `ngx.arg[2]` 中, 所以你要改输出内容那么就把 `ngx.arg[1]`改掉，如果不想要以后的内容了那么 `ngx.arg[2]=true` 就行.

#### 需要注意的地方

还有一个特别需要注意地方是，当代码运行到 `body_filter_by_lua*` 时，`HTTP`报头（header）已经发送出去了。如果在之前设置了跟响应体相关的报头，而又在 `body_filter_by_lua*`中修改了响应体，会导致响应报头和实际响应的不一致。举个简单的例子：假设上游的服务器返回了 `Content-Length` 报头，而 `body_filter_by_lua*` 又修改了响应体的实际大小。客户端收到这个报头后，按其中的 `Content-Length` 去处理，顺着一头栽进坑里。由于 `Nginx` 的流式响应，发出去的报头就像泼出去的水，要想修改只能提前进行。`OpenResty` 提供了跟 `body_filter_by_lua*` 相对应的 `header_filter_by_lua*`。`header_filter` 会在 `Nginx` 发送报头之前调用，所以可以在这里置空 `Content-Length` 报头：

```
header_filter_by_lua_block {
    ngx.header.content_length = nil
}
```

现在 `Nginx` 会代以 `Transfer-Encoding: chunked`，再也不会误导客户端了。同样可能需要处理的还有 `accept-range` 和 `etag` 等跟响应体相关的报头。HTTP1.1 之后基于流式处理的方式，`body_filter_by_lua` 基本在一个请求中会调用多次。 简单直白的理解就是流式输出，每次拿到了如果你要处理那么就处理，不处理就输出！！


#### 总结

- `body_filter_by_lua*` 可能在一次请求中调用多次，跟响应数据量无关，取决于响应次数
- `body_filter_by_lua*` 的最后一次调用时，`ngx.arg[1]` 一般为空字符串
- `body_filter_by_lua*` 也会在 `subrequest` 之中调用
- `body_filter_by_lua*` 有些时候离不开有 `header_filter_by_lua*` 辅佐


