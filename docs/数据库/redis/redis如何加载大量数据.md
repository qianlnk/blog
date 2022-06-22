# redis如何加载大量数据

如果选择Shell + redis客户端一条条导入

每个RTT都占用很多时间很难高效

这里使用redis的pipe：

格式：
```shell
*3\r\n            命令起始，定义共3个输入参数
$3\r\n            下一个参数字节长度
SET\r\n           命令参数
$3\r\n             下一个参数字节长度
key\r\n           变量参数
$5\r\n            下一个参数字节长度
value\r\n        值参数
```

命令：

```shell
cat 1.txt  | redis-cli --pipe
```

只占一个RTT时间
