# 分布式锁

几个要用到的redis命令：

- setnx(key, value)：“set if not exits”，若该key-value不存在，则成功加入缓存并且返回1，否则返回0。

- get(key)：获得key对应的value值，若不存在则返回null。

- getset(key, value)：此方法返回key对应的旧value值，若不存在则返回null；且将Redis服务器中key对应的旧value更新为新的value。

- expire(key, seconds)：设置key-value的有效期为seconds秒。

![](/uploads/upload_627182a2c5eea715d82dfbaa651450a7.png)

```go
/*
   package redislock
   基于单节点redis 分布式锁
*/
package redislock

import (
	"crypto/rand"
	"encoding/base64"
	"errors"
	"time"

	redis "gopkg.in/redis.v5"
)

type RedisLock struct {
	lockKey string        //key
	rand    string        //随机数
	expire  time.Duration //过期时间
}

//保证原子性（redis是单线程），避免del删除了，其他client获得的lock
var delScript = redis.NewScript(`
if redis.call("get", KEYS[1]) == ARGV[1] then
	return redis.call("del", KEYS[1])
else
	return 0
end`)

// new 一个锁
func NewLock(key string, expire time.Duration) *RedisLock {
	return &RedisLock{
		lockKey: key,
		expire:  expire,
	}
}

//Lock 获得锁，timeout单位秒
func (lock *RedisLock) Lock(rd *redis.Client) error {
	{ //随机数
		b := make([]byte, 16)
		_, err := rand.Read(b)
		if err != nil {
			return err
		}
		lock.rand = base64.StdEncoding.EncodeToString(b)
	}
	cmd := rd.SetNX(lock.lockKey, lock.rand, lock.expire)
	if cmd.Err() != nil {
		return errors.New("redis fail")
	}
	if cmd.Val() {
		return nil
	}
	return errors.New("lock fail")
}

//Unlock 解锁
func (lock *RedisLock) Unlock(rd *redis.Client) error {
	cmd := delScript.Run(rd, []string{lock.lockKey}, lock.rand)
	return cmd.Err()
}

```