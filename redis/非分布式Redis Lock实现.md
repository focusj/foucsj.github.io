Redis(2.6.12+ version) SET命令提供了NX, XX写入选项. NX表示只有当key不存在时才执行写入操作. XX与之相反, 当key存在时执行写入操作. 基于Redis实现锁的思路非常简单: 创建一个KV对. 因为KEY是非议的, 所以可以在KEY上产生互斥效果. 借助刚才NX选项, 如果set成功表示获得了锁, 否则锁获取失败. 详细解释一下.



给定锁: lock1, 用户A首次获该锁, 并hold这个锁一段时间. 这个过程其实我们执行了以下命令: `SET lock1 value NX EX lock-time`. 这时用户B再次尝试获取锁lock1, 因为lock1已经被用户A占用, 所以, 用户B是无法获取lock1的. 我们执行`SET lock1 value NX EX lock-time`时, 返回nil.



这样我们已经通过redis的set命令描述清楚锁的基本原理. 但是这种实现有一个潜在的问题: 假如进程A耗时很长, 超过了lock-time, key就会被redis删掉. 这时进程B重新获取该锁是可以获取成功的. 当进程A执行完成后释放锁, 它其实释放的是Thread B的锁.



解决这个问题, 需要额外做两件事情: 1. value值设定为random的, 用于验证lock身份; 2. 删除key要是用脚本, 并检查value是不是获取锁时设定的值, 防止误删其它进程获取的锁.



解锁的时使用的lua脚本如下:

```lua

if redis.call('get', 'key') == 'value' then return redis.call('del', 'key') else return 0 end

```



Redis执行脚本的时使用单个Lua解释器去运行所有脚本, 并且, Redis也保证脚本会以*原子性*(atomic)的方式执行: 当某个脚本正在运行的时候, 不会有其他脚本或Redis命令被执行. 所以, 结合这两个措施可以确保正确的释放锁.