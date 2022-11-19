# range

## range实现原理

for init;cond;post{
  iter_init
  index=index_temp
  value=value_temp
  original statements
}
	
1. 编译器会从for-range语句中提取出初始化语句init，条件语句cond，和迭代语句post。
2. 然后通过一个for循环实现，循环体中有两个赋值语句，分别对应的是for-range的两个返回值。

## range for slice

循环开始前循环次数就已经确定了，所以循环过程中新添加的元素是没办法遍历到的

## range for map

遍历map时没有指定循环次数，循环体与遍历slice类似。由于map底层实现与slice不同，map底层使用hash表实现，插入数据位置是随机的，所以遍历过程中新插入的数据不能保证被遍历到

## range for channel
		
channel遍历是依次从channel中读取数据,读取前是不知道里面有多少个元素的。
如果channel中没有元素，则会阻塞等待，
如果channel已被关闭，则会解除阻塞并退出循环。

## range性能优化
### 切片遍历优化：
通过index去获得value，而不是直接在range接受value
```go
func RangeSlice(slice []int) {
    for index, value := range slice {
        _, _ = index, value
    }
}
```

遍历过程中每次迭代会对index和value进行赋值，如果数据量大或者value类型为string时，对value的赋值操作可能是多余的，可以在for-range中忽略value值，使用slice[index]引用value值

### Map遍历优化：
直接获取value可能比通过可以获取value更为高效
```go
func RangeMap(myMap map[int]string) {
    for key, _ := range myMap {
        _, _ = key, myMap[key]
    }
}
```

函数中for-range语句中只获取key值，然后跟据key值获取value值，虽然看似减少了一次赋值，但通过key值查找value值的性能消耗可能高于赋值消耗。能否优化取决于map所存储数据结构特征、结合实际情况进行.