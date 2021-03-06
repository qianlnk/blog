# HTTP常见状态码

![](/uploads/upload_7f60a968b5e01391686fe11d9a3aa624.png)


1. 1xx

    1xx状态码属于提示信息，是协议处理的一种中间状态，实际用到的比较少。

2. 2xx

    2xx表示服务器成功处理了请求
    
    200: ok 一切正常
    204： no content 没有body数据
    206： partial content 部分body内容，应用于HTTP分块下载、断点续传。

3. 3xx
    
    3xx表示客户端请求的资源发生了变动，需要客户端使用新的URL重新发送请求获取资源，也就是**重定向**。
    
    301: moved permanently 永久重定向，说明资源不在了，需要使用新的URL访问
    302: found 暂时重定向，说明请求的资源还在，但暂时需要用另一个 URL 来访问。
    304: not modified 不具有跳转的含义，表示资源未修改，重定向已存在的缓冲文件，也称缓存重定向，用于缓存控制。
    
4. 4xx

    4xx表示客户端报文有误，服务端无法处理，属于客服端的错误
    
    400: bad request，请求有误，比较笼统的提示错误，不具体。
    403: forbidden，服务端禁止访问，认证不通过等
    404: not found，资源不存在

5. 5xx

  5xx表示服务器处理请求时发生内部错误，属于服务端的错误
  
  500: internal server error，比较笼统的错误提示，不具体。
  501: not implemented，表示功能还不支持“即将开业，敬请期待”。
  502: bad gateway，通常是服务器作为网关或代理时返回的错误码，表示服务器自身工作正常，访问 后端服务器发生了错误。
  503:servive unavaliable,表示服务器繁忙，暂时服务响应请求。
  