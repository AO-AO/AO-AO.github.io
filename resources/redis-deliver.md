## Redis实现分布式资源锁

### 应用场景

高并发场景中实现对资源的快速分发.

### 实现原理

#### 基础

redis的分布式锁.

> [Redis分布式锁参考文档](https://redis.io/topics/distlock)

#### 扩展

扩展redis的分布式锁, 锁的过期时间作为分发周期, 锁的value作为资源数量限制.

1. 在每次分发请求尝试重设资源锁(资源锁value为分发数量, 资源锁拥有过期时间)
2. 每次分发对资源锁-count(需要通过lua脚本自定义原子操作incrbyex)
3. 根据2中的结果判断是否分发

### 伪代码

```js
/**
 * @param {String} lockName 锁key (redis key)
 * @param {Number} resouceCount 锁value (redis value)
 * @param {Number} period 毫秒锁过期时间 (redis expire)
 * @returns true | false, 可以下发 | 不能下发
 */
async function getSetDeliverLock(lockName, resouceCount, period){

    // 尝试重设deliver
    try {
        setReply = await trySetNewDeliver(lockName, resouceCount, period);
    } catch (err) {
        console.error(err);
    }

    // 控制分发
    let incrbyexReply = null;
    try{
        incrbyexReply = await incrbyex(lockName, -1);
    } catch (err) {
        throw(err);
    }
    if(incrbyexReply !== null && Number(incrbyexReply) >= 0){
        return true;
    } else {
        return false;
    }
}

/**
 * @param {String} lockName 锁key (redis key)
 * @param {Number} resouceCount 锁value (redis value)
 * @param {Number} period 毫秒锁过期时间 (redis expire)
 * @returns true | false, set成功 | set失败
 */
async function trySetNewDelive(lockName, resouceCount, expireTimeMs) {
    let setReply = null;
    try {
        setReply = await redisCli.set(lockName, resouceCount, 'PX', period, 'NX');
    } catch (err) {
       throw (err);
    }
    if(setReply) {
        return true;
    } else {
        return false;
    }
}
```

需要注意的是, redis的incrbyex方法是自己定义的一个原子操作, 代码如下:

> 原生的redis incrby方法, 在高并发场景中, 可能因为对已经过期的key进行incrby导致死锁产生.

```js
/**
 *
 * @param {*} key
 * @param {*} count
 * @returns key不存在返回null,否则返回incrby结果
 */
incrbyex(key, count) {
    let lua_script = 'local result,v v = redis.call("exists", ARGV[1]) if v == 1 then result = redis.call("incrby", ARGV[1], ARGV[2]) end return result';
    return RedisCli.eval(lua_script, 0, key, count);
}
```

### 优化

1. trySetNewDelive 会对redis primary节点造成较大压力, 可以先通过redis replica节点校验锁是否存在来减轻primary节点的压力.