# sync.Map 底层实现

在 syncmap 中实际存在两个 map：read map & dirty map。 可以看一下代码中的表示:

```go
type Map struct {
    mu sync.Mutex
    read atomic.Value // readOnly
    dirty map[interface{}]*entry
    misses int
}

//mu是用来保护dirty的互斥锁
//missed是记录没命中read的次数

type readOnly struct {
    m       map[interface{}]*entry
    amended bool // true if the dirty map contains some key not in m.
}

type entry struct {
	p unsafe.Pointer // *interface{}
}
```

p有三种值：

- nil: entry已被删除了，并且m.dirty为nil
- expunged: entry已被删除了，并且m.dirty不为nil，而且这个entry不存在于m.dirty中
- 其它： entry是一个正常的值

## 实现思路

要读受到的影响尽量小，那么最容易想到的想法，就是读写分离。golang sync map应该也是受到这个想法的启发设计出来的。使用了两个map，一个叫read，一个叫dirty，两个map存储的都是指针，指向value数据本身，所以两个map是共享value数据的，更新value对两个map同时可见。

dirty可以进行增删查，当时都要进行加互斥锁。

read中存在的key，可以无锁的读，借助CAS进行无锁的更新、删除操作，但是不能新增key，相当于dirty的一个cache，由于value共享，所以能通过read对已存在的value进行更新。

read不能新增key，那么数据怎么来的呢？sync map中会记录miss cache的次数，当miss次数大于等于dirty元素个数时，就会把dirty变成read，原来的dirty清空。

为了方便dirty直接变成read，那么得保证read中存在的数据dirty必须有，所以在dirty是空的时候，如果要新增一个key，那么会把read中的元素复制到dirty中，然后写入新key。

然后删除操作也很有意思，使用的是延迟删除，优先看read中没有，read中有，就把read中的对应entry指针中的p置为nil，作为一个标记。在read中标记为nil的，只有在dirty提升为read时才会被实际删除。

## Load 读取

```go
// Load returns the value stored in the map for a key, or nil if no
// value is present.
// The ok result indicates whether value was found in the map.
func (m *Map) Load(key interface{}) (value interface{}, ok bool) {
    read, _ := m.read.Load().(readOnly)
    e, ok := read.m[key]
    if !ok && read.amended {
        m.mu.Lock()
        // Avoid reporting a spurious miss if m.dirty got promoted while we were
        // blocked on m.mu. (If further loads of the same key will not miss, it's
        // not worth copying the dirty map for this key.)
        read, _ = m.read.Load().(readOnly)
        e, ok = read.m[key]
        if !ok && read.amended {
            e, ok = m.dirty[key]
            // Regardless of whether the entry was present, record a miss: this key
            // will take the slow path until the dirty map is promoted to the read
            // map.
            m.missLocked()
        }
        m.mu.Unlock()
    }
    if !ok {
        return nil, false
    }
    return e.load()
}

func (e *entry) load() (value interface{}, ok bool) {
    p := atomic.LoadPointer(&e.p)
    if p == nil || p == expunged {
        return nil, false
    }
    return *(*interface{})(p), true
}

func (m *Map) missLocked() {
    m.misses++
    if m.misses < len(m.dirty) {
        return
    }
    m.read.Store(readOnly{m: m.dirty})
    m.dirty = nil
    m.misses = 0
}
```

读取时，先去read读取；如果没有，就加锁，然后去dirty读取，同时调用missLocked()，再解锁。在missLocked中，会递增misses变量，如果misses>len(dirty)，那么把dirty提升为read，清空原来的dirty。

## Store 写入

```go
// Store sets the value for a key.
func (m *Map) Store(key, value interface{}) {
    read, _ := m.read.Load().(readOnly)
    if e, ok := read.m[key]; ok && e.tryStore(&value) {
        return
    }

    m.mu.Lock()
    read, _ = m.read.Load().(readOnly)
    if e, ok := read.m[key]; ok {
        if e.unexpungeLocked() {
            // The entry was previously expunged, which implies that there is a
            // non-nil dirty map and this entry is not in it.
            m.dirty[key] = e
        }
        e.storeLocked(&value)
    } else if e, ok := m.dirty[key]; ok {
        e.storeLocked(&value)
    } else {
        if !read.amended {
            // We're adding the first new key to the dirty map.
            // Make sure it is allocated and mark the read-only map as incomplete.
            m.dirtyLocked()
            m.read.Store(readOnly{m: read.m, amended: true})
        }
        m.dirty[key] = newEntry(value)
    }
    m.mu.Unlock()
}

// tryStore stores a value if the entry has not been expunged.
//
// If the entry is expunged, tryStore returns false and leaves the entry
// unchanged.
func (e *entry) tryStore(i *interface{}) bool {
    p := atomic.LoadPointer(&e.p)
    if p == expunged {
        return false
    }
    for {
        if atomic.CompareAndSwapPointer(&e.p, p, unsafe.Pointer(i)) {
            return true
        }
        p = atomic.LoadPointer(&e.p)
        if p == expunged {
            return false
        }
    }
}

func (m *Map) dirtyLocked() {
    if m.dirty != nil {
        return
    }

    read, _ := m.read.Load().(readOnly)
    m.dirty = make(map[interface{}]*entry, len(read.m))
    for k, e := range read.m {
        if !e.tryExpungeLocked() {
            m.dirty[k] = e
        }
    }
}

func (e *entry) tryExpungeLocked() (isExpunged bool) {
    p := atomic.LoadPointer(&e.p)
    for p == nil {
        if atomic.CompareAndSwapPointer(&e.p, nil, expunged) {
            return true
        }
        p = atomic.LoadPointer(&e.p)
    }
    return p == expunged
}

// unexpungeLocked ensures that the entry is not marked as expunged.
//
// If the entry was previously expunged, it must be added to the dirty map
// before m.mu is unlocked.
func (e *entry) unexpungeLocked() (wasExpunged bool) {
    return atomic.CompareAndSwapPointer(&e.p, expunged, nil)
}
```

写入的时候，先看read中能否查到key，在read中存在的话，直接通过read中的entry来更新值；在read中不存在，那么就上锁，然后double check。这里需要留意，分几种情况：

- double check发现read中存在，如果是expunged，那么就先尝试把expunged替换成nil，最后如果entry.p==expunged就复制到dirty中，再写入值；否则不用替换直接写入值。
- dirty中存在，直接更新
- dirty中不存在，如果此时dirty为空，那么需要将read复制到dirty中，最后再把新值写入到dirty中。复制的时候调用的是dirtyLocked()，在复制到dirty的时候，read中为nil的元素，会更新为expunged，并且不复制到dirty中。

## Delete 删除

```golang
// Delete deletes the value for a key.
func (m *Map) Delete(key interface{}) {
    read, _ := m.read.Load().(readOnly)
    e, ok := read.m[key]
    if !ok && read.amended {
        m.mu.Lock()
        read, _ = m.read.Load().(readOnly)
        e, ok = read.m[key]
        if !ok && read.amended {
            delete(m.dirty, key)
        }
        m.mu.Unlock()
    }
    if ok {
        e.delete()
    }
}

func (e *entry) delete() (hadValue bool) {
    for {
        p := atomic.LoadPointer(&e.p)
        if p == nil || p == expunged {
            return false
        }
        if atomic.CompareAndSwapPointer(&e.p, p, nil) {
            return true
        }
    }
}
```

删除很简单，read中存在，就把read中的entry.p置为nil(原子操作)，如果只在ditry中存在，那么就直接从dirty中删掉对应的entry。

从m.dirty中直接删除即可，就当它没存在过，但是如果是从m.read中删除，并不会直接删除，而是打标记：