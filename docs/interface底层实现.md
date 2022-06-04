# interface底层实现

根据 interface 是否包含有 method，底层实现上用两种 struct 来表示：iface 和 eface。eface表示不含 method 的 interface 结构，或者叫 empty interface。对于 Golang 中的大部分数据类型都可以抽象出来 _type 结构，同时针对不同的类型还会有一些其他信息。

### _type

```go
type _type struct {
    size       uintptr // type size
    ptrdata    uintptr // size of memory prefix holding all pointers
    hash       uint32  // hash of type; avoids computation in hash tables
    tflag      tflag   // extra type information flags
    align      uint8   // alignment of variable with this type
    fieldalign uint8   // alignment of struct field with this type
    kind       uint8   // enumeration for C
    alg        *typeAlg  // algorithm table
    gcdata    *byte    // garbage collection data
    str       nameOff  // string form
    ptrToThis typeOff  // type for pointer to this type, may be zero
}
```

### eface

```go
type eface struct {
    _type *_type
    data  unsafe.Pointer
}
```

### iface

```go
type iface struct {
    tab  *itab
    data unsafe.Pointer
}

// layout of Itab known to compilers
// allocated in non-garbage-collected memory
// Needs to be in sync with
// ../cmd/compile/internal/gc/reflect.go:/^func.dumptypestructs.
type itab struct {
    inter  *interfacetype
    _type  *_type
    link   *itab
    bad    int32
    inhash int32      // has this itab been added to hash?
    fun    [1]uintptr // variable sized
}
```

### nil

```go
// nil is a predeclared identifier representing the zero value for a pointer, channel, func, interface, map, or slice type.

var nil Type // Type must be a pointer, channel, func, interface, map, or slice type

```

也就是说，只有pointer, channel, func, interface, map, or slice 这些类型的值才可以是nil.

总结：

一个接口包括动态类型和动态值。
如果一个接口的动态类型和动态值都为空，则这个接口为空的。

```go
package main

import ("fmt")


func main(){

       var a interface{} = nil // _type = nil, data = nil
       var b interface{} = (*int)(nil) // _type 包含 *int 类型信息, data = nil

       fmt.Println(a==nil)
       fmt.Println(b==nil)
}
```

```
true
false
```