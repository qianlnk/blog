# HTTP常见字段

- Host: 客户端发送请求时，用来指定服务器的域名。
- Content-Length:服务器在返回数据时，会有 Content-Length 字段，表明本次回应的数据长度。
- Content-Type: 字段用于服务器回应时，告诉客户端，本次数据是什么格式
- Content-Encoding: 字段说明数据的压缩方法。表示服务器返回的数据使用了什么压缩格式
- Connection: 字段最常用于客户端要求服务器使用 TCP 持久连接，以便其他请求复用。HTTP/1.1 版本的默认连接都是持久连接，但为了兼容老版本的 HTTP，需要指定 Connection 首部字段的值为 Keep-Alive。