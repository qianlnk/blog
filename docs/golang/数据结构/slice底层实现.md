# slice底层实现

```go
type slice struct {
	array unsafe.Pointer   //指向底层数组的指针
	len   int              //切片的长度
	cap   int              //切片的容量
}
```

切片是基于数组实现的，是长度动态不固定的数据结构，本质上是一个对数组子序列的引用，提供了对数组的轻量级访问。

**数组：**

 数组是一种具有固定长度的基本数据结构,在golang中与C语言一样数组一旦创建了它的长度就不允许改变,数组的空余位置用0填补,不允许数组越界。

 **slice 和 数组 区别**
 
(1)go是有数组的,只是平时用切片比较多。数组大小一旦创建就不能改变,数组长度大于元素个数的时候会用0补位，这跟其他语言是相通的。

(2)切片slice可以看作是对数组的一切操作,它是一个引用数据类型,其数据结构包括底层数组的地址,以及元素可操作长度len或可扩容长度cap。

(3)要想突破slice的cap限制，进行扩容就需要使用append()函数进行操作。如果append追加的元素后slice的总长度不超过底层数组的总长度，那么slice引用的地址不会发生改变,反之引用地址会变成新的数组的地址。

(4)slice是一个抽象的概念,它存在的意义在于方便对一个顺序结构进行一些方便操作,例如查找,排序,追加等等,这个类似于python的list。

## 扩容规则

```go
func growslice(et *_type, old slice, cap int) slice {
	...
	...
	
	newcap := old.cap
	doublecap := newcap + newcap
	if cap > doublecap {
		newcap = cap
	} else {
		if old.len < 1024 {
			newcap = doublecap
		} else {
			// Check 0 < newcap to detect overflow
			// and prevent an infinite loop.
			for 0 < newcap && newcap < cap {
				newcap += newcap / 4
			}
			// Set newcap to the requested cap when
			// the newcap calculation overflowed.
			if newcap <= 0 {
				newcap = cap
			}
		}
	}

...
...

```

1. 如果新申请的cap大于旧cap的两倍，那么最终容量就是新申请的cap。
2. 否则如果切片长度小于1024，则最终容量是旧cap的两倍。
3. 否则如果切片长度大于1024，则最终容量从旧容量开始循环增加原来的1/4，直到最终容量大于等于新申请的cap。
4. 基于3，如果旧容量就是0，那么最终容量就是新申请的容量。

## 扩容前后slice是否相同？

case1:

愿数组还有容量可以扩容，这种情况扩容以后指针还是指向原来的数组，对扩容后的切片操作可能会影响多个指针指向相同地址的slice。

case2:

原数组的容量已经达到最大值，再想扩容，GO默认会先开一片更大的内存区域，把原来的数组值拷贝过来，然后再append操作，这种情况不影响原数组。

## slice线程不安全
概念：

多个线程访问同一个对象时，调用这个对象的行为都可以获得正确的结果，那么这个对象就是线程安全的。

若有多个线程同时执行写操作，一般都需要考虑线程同步，否则的话就可能影响线程安全。

原理：

slice底层结构并没有使用加锁等方式，不支持并发读写，所以并不是线程安全的，使用多个 goroutine 对类型为 slice 的变量进行操作，每次输出的值大概率都不会一样，与预期值不一致; slice在并发执行中不会报错，但是数据会丢失