# 批量杀死慢查询

//查询ugcs库中超过30s的慢查询
```
var currOp = db.currentOp(    {      "active" : true,      "secs_running" : { "$gt" : 30 },      "ns" : /^ugcs\./    } )
```

//批量杀进程
```
for (op in currOp.inprog) {db.killOp(currOp.inprog[op].opid)}
```