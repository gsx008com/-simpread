> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/RcBi9EJu_oOZdeUTlvPIAA)

![](https://mmbiz.qpic.cn/mmbiz_png/iaIdQfEric9TwDLiaiaWVWJILbpcFk8iaYKHEpKlmD7krfWCzRvtA8hP2HovZAxh4om2Ssz8UqFz9JS0bISLrNSD1YA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

JavaGuide 在线网站：javaguide.cn  
来源：https://developer.aliyun.com/article/531067

昨晚在阿里云社区看到一份阿里云官方 Redis 开发规范，是一位阿里云数据库技术专家 (Redis 方向) 写的，感觉有很多地方值得参考。我对原文排版和内容进行了简单完善，这里分享一下。

一、键值设计
------

### 1. key 名设计

**(1)【建议】: 可读性和可管理性**

以业务名 (或数据库名) 为前缀(防止 key 冲突)，用冒号分隔，比如业务名: 表名: id

```
ugc:video:1
```

**(2)【建议】：简洁性**

保证语义的前提下，控制 key 的长度，当 key 较多时，内存占用也不容忽视，例如：

```
user:{uid}:friends:messages:{mid}简化为u:{uid}:fr:m:{mid}。
```

**(3)【强制】：不要包含特殊字符**

反例：包含空格、换行、单双引号以及其他转义字符

详细解析：[Redis 开发规范解析 (一)-- 键名设计](https://mp.weixin.qq.com/s?spm=a2c6h.12873639.article-detail.7.753b1feeTX187Q&__biz=Mzg2NTEyNzE0OA==&mid=2247483663&idx=1&sn=7c4ad441eaec6f0ff38d1c6a097b1fa4&chksm=ce5f9e8cf928179a2c74227da95bec575bdebc682e8630b5b1bb2071c0a1b4be6f98d67c37ca&scene=21#wechat_redirect) 。

### 2. value 设计

**(1)【强制】：拒绝 bigkey(防止网卡流量、慢查询)**

string 类型控制在 10KB 以内，hash、list、set、zset 元素个数不要超过 5000。

反例：一个包含 200 万个元素的 list。

非字符串的 bigkey，不要使用 del 删除，使用 hscan、sscan、zscan 方式渐进式删除，同时要注意防止 bigkey 过期时间自动删除问题 (例如一个 200 万的 zset 设置 1 小时过期，会触发 del 操作，造成阻塞，而且该操作不会不出现在慢查询中 (latency 可查))，查找方法 [1] 和删除方法 [2] 。

详细解析：[Redis 开发规范解析 (二)-- 老生常谈 bigkey](https://mp.weixin.qq.com/s?spm=a2c6h.12873639.article-detail.10.753b1feeTX187Q&__biz=Mzg2NTEyNzE0OA==&mid=2247483677&idx=1&sn=5c320b46f0e06ce9369a29909d62b401&chksm=ce5f9e9ef928178834021b6f9b939550ac400abae5c31e1933bafca2f16b23d028cc51813aec&scene=21#wechat_redirect)

**(2)【推荐】：选择适合的数据类型。**

例如：实体类型 (要合理控制和使用数据结构内存编码优化配置, 例如 ziplist，但也要注意节省内存和性能之间的平衡)

反例：

```
set user:1:name tomset user:1:age 19set user:1:favor football
```

正例:

```
hmset user:1 name tom age 19 favor football
```

### 3.【推荐】：控制 key 的生命周期，redis 不是垃圾桶。

建议使用 expire 设置过期时间 (条件允许可以打散过期时间，防止集中过期)，不过期的数据重点关注 idletime。

> [《Java 面试指北》](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247525881&idx=1&sn=33d4565668cff95f5ee239c6e1d660fe&chksm=cea12a32f9d6a324432c964bf536181f2d55ee50afba9a53d9b09fa0e1a7126a03bb1dde4c60&token=698289097&lang=zh_CN&scene=21#wechat_redirect)来啦！这是一份教你如何更高效地准备面试的小册，涵盖常见八股文（系统设计、常见框架、分布式、高并发 ......）、优质面经等内容。

二、命令使用
------

### 1.【推荐】 O(N) 命令关注 N 的数量

例如 hgetall、lrange、smembers、zrange、sinter 等并非不能使用，但是需要明确 N 的值。有遍历的需求可以使用 hscan、sscan、zscan 代替。

### 2.【推荐】：禁用命令

禁止线上使用 keys、flushall、flushdb 等，通过 redis 的 rename 机制禁掉命令，或者使用 scan 的方式渐进式处理。

### 3.【推荐】合理使用 select

redis 的多数据库较弱，使用数字进行区分，很多客户端支持较差，同时多业务用多数据库实际还是单线程处理，会有干扰。

### 4.【推荐】使用批量操作提高效率

*   原生命令：例如 mget、mset。
    
*   非原生命令：可以使用 pipeline 提高效率。
    

但要注意控制一次批量操作的 **元素个数** (例如 500 以内，实际也和元素字节数有关)。

注意两者不同：

*   原生是原子操作，pipeline 是非原子操作。
    
*   pipeline 可以打包不同的命令，原生做不到
    
*   pipeline 需要客户端和服务端同时支持。
    

### 5.【建议】Redis 事务功能较弱，不建议过多使用

Redis 的事务功能较弱 (不支持回滚)，而且集群版本(自研和官方) 要求一次事务操作的 key 必须在一个 slot 上(可以使用 hashtag 功能解决)

### 6.【建议】Redis 集群版本在使用 Lua 上有特殊要求：

*   所有 key 都应该由 KEYS 数组来传递，redis.call/pcall 里面调用的 redis 命令，key 的位置，必须是 KEYS array, 否则直接返回 error，"-ERR bad lua script for redis cluster, all the keys that the script uses should be passed using the KEYS array"
    
*   所有 key，必须在 1 个 slot 上，否则直接返回 error, "-ERR eval/evalsha command keys must in same slot"
    

### 7.【建议】必要情况下使用 monitor 命令时，要注意不要长时间使用。

三、客户端使用
-------

### 1.【推荐】避免多个应用使用一个 Redis 实例

正例：不相干的业务拆分，公共数据做服务化。

### 2.【推荐】使用带有连接池的数据库

使用带有连接池的数据库，可以有效控制连接，同时提高效率，标准使用方式：

```
执行命令如下：Jedis jedis = null;try {    jedis = jedisPool.getResource();    //具体的命令    jedis.executeCommand()} catch (Exception e) {    logger.error("op key {} error: " + e.getMessage(), key, e);} finally {    //注意这里不是关闭连接，在JedisPool模式下，Jedis会被归还给资源池。    if (jedis != null)        jedis.close();}
```

下面是 JedisPool 优化方法的文章:

*   Jedis 常见异常汇总 [3]
    
*   JedisPool 资源池优化 [4]
    

### 3.【建议】高并发下建议客户端添加熔断功能 (例如 netflix hystrix)

在通过 Redis 客户端操作 Redis 中的数据时，我们会在其中加入熔断器的逻辑。比如，当节点处于熔断状态时，直接返回空值以及熔断器三种状态之间的转换，具体的示例代码像下面这样：

这样，当某一个 Redis 节点出现问题，Redis 客户端中的熔断器就会实时监测到，并且不再请求有问题的 Redis 节点，避免单个节点的故障导致整体系统的雪崩。

### 4.【推荐】确保登录安全

设置合理的密码，如有必要可以使用 SSL 加密访问（阿里云 Redis 支持）

### 5.【建议】选择合适的内存淘汰策略

根据自身业务类型，选好 maxmemory-policy(最大内存淘汰策略)，设置好过期时间。

默认策略是 volatile-lru，即超过最大内存后，在过期键中使用 lru 算法进行 key 的剔除，保证不过期数据不被删除，但是可能会出现 OOM 问题。

**其他策略如下** ：

*   allkeys-lru：根据 LRU 算法删除键，不管数据有没有设置超时属性，直到腾出足够空间为止。
    
*   allkeys-random：随机删除所有键，直到腾出足够空间为止。
    
*   volatile-random: 随机删除过期键，直到腾出足够空间为止。
    
*   volatile-ttl：根据键值对象的 ttl 属性，删除最近将要过期数据。如果没有，回退到 noeviction 策略。
    
*   noeviction：不会剔除任何数据，拒绝所有写入操作并返回客户端错误信息 "(error) OOM command not allowed when used memory"，此时 Redis 只响应读操作。
    

> [《Java 面试指北》](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247525881&idx=1&sn=33d4565668cff95f5ee239c6e1d660fe&chksm=cea12a32f9d6a324432c964bf536181f2d55ee50afba9a53d9b09fa0e1a7126a03bb1dde4c60&token=698289097&lang=zh_CN&scene=21#wechat_redirect)来啦！这是一份教你如何更高效地准备面试的小册，涵盖常见八股文（系统设计、常见框架、分布式、高并发 ......）、优质面经等内容。

四、相关工具
------

### 1.【推荐】：数据同步

redis 间数据同步可以使用：redis-port

### 2.【推荐】：big key 搜索

[Redis 为什么变慢了？一文讲透如何排查 Redis 性能问题 | 万字长文](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247499195&idx=2&sn=13fe477aac7cbb8cd03d91511ea98ba0&chksm=cea1b270f9d63b667b72c142321dfccdb64deb11f4fd082e1cfedf7d45768e125a9db7fa6918&token=973133388&lang=zh_CN&scene=21#wechat_redirect)

### 3.【推荐】：热点 key 寻找

京东开源的 hotkey[5] 支持毫秒级探测热点数据，毫秒级推送至服务器集群内存，大幅降低热 key 对数据层查询压力。

五 附录：删除 bigkey
--------------

下面操作可以使用 pipeline 加速。

redis 4.0 已经支持 key 的异步删除，欢迎使用。

### 1. Hash 删除: hscan + hdel

```
public void delBigHash(String host, int port, String password, String bigHashKey) {    Jedis jedis = new Jedis(host, port);    if (password != null && !"".equals(password)) {        jedis.auth(password);    }    ScanParams scanParams = new ScanParams().count(100);    String cursor = "0";    do {        ScanResult<Entry<String, String>> scanResult = jedis.hscan(bigHashKey, cursor, scanParams);        List<Entry<String, String>> entryList = scanResult.getResult();        if (entryList != null && !entryList.isEmpty()) {            for (Entry<String, String> entry : entryList) {                jedis.hdel(bigHashKey, entry.getKey());            }        }        cursor = scanResult.getStringCursor();    } while (!"0".equals(cursor));    //删除bigkey    jedis.del(bigHashKey);}
```

### 2. List 删除: ltrim

```
public void delBigList(String host, int port, String password, String bigListKey) {    Jedis jedis = new Jedis(host, port);    if (password != null && !"".equals(password)) {        jedis.auth(password);    }    long llen = jedis.llen(bigListKey);    int counter = 0;    int left = 100;    while (counter < llen) {        //每次从左侧截掉100个        jedis.ltrim(bigListKey, left, llen);        counter += left;    }    //最终删除key    jedis.del(bigListKey);}
```

### 3. Set 删除: sscan + srem

```
public void delBigSet(String host, int port, String password, String bigSetKey) {    Jedis jedis = new Jedis(host, port);    if (password != null && !"".equals(password)) {        jedis.auth(password);    }    ScanParams scanParams = new ScanParams().count(100);    String cursor = "0";    do {        ScanResult<String> scanResult = jedis.sscan(bigSetKey, cursor, scanParams);        List<String> memberList = scanResult.getResult();        if (memberList != null && !memberList.isEmpty()) {            for (String member : memberList) {                jedis.srem(bigSetKey, member);            }        }        cursor = scanResult.getStringCursor();    } while (!"0".equals(cursor));    //删除bigkey    jedis.del(bigSetKey);}
```

### 4. SortedSet 删除: zscan + zrem

```
public void delBigZset(String host, int port, String password, String bigZsetKey) {    Jedis jedis = new Jedis(host, port);    if (password != null && !"".equals(password)) {        jedis.auth(password);    }    ScanParams scanParams = new ScanParams().count(100);    String cursor = "0";    do {        ScanResult<Tuple> scanResult = jedis.zscan(bigZsetKey, cursor, scanParams);        List<Tuple> tupleList = scanResult.getResult();        if (tupleList != null && !tupleList.isEmpty()) {            for (Tuple tuple : tupleList) {                jedis.zrem(bigZsetKey, tuple.getElement());            }        }        cursor = scanResult.getStringCursor();    } while (!"0".equals(cursor));    //删除bigkey    jedis.del(bigZsetKey);}
```

### 参考资料

[1]

查找方法: _https://developer.aliyun.com/article/531067#cc1_

[2]

删除方法: _https://developer.aliyun.com/article/531067#cc2_

[3]

Jedis 常见异常汇总: _https://yq.aliyun.com/articles/236384?spm=a2c6h.12873639.article-detail.11.753b1feeTX187Q_

[4]

JedisPool 资源池优化: _https://yq.aliyun.com/articles/236383?spm=a2c6h.12873639.article-detail.12.753b1feeTX187Q_

[5]

hotkey: _https://gitee.com/jd-platform-opensource/hotkey_

  

··················END················

**近期文章精选** ：

*   [这个 SpringBoot+ Vue 开源博客系统太酷炫了！](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247526969&idx=1&sn=a2f61455c0e855dd2bab25ba77d584cb&scene=21#wechat_redirect)
    
*   [美团二面：Redis 5 种基础数据结构？](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247526650&idx=1&sn=15042ffbe9ef97cccafbe9e4c2de5dad&scene=21#wechat_redirect)
    
*   [IDEA 版防沉迷插件，有点意思！](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247526937&idx=1&sn=e22afe08fb2f86517b794ec0f092aad9&scene=21#wechat_redirect)
    
*   [该死的单元测试，写起来是真的折磨！](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247527023&idx=1&sn=6f7673808dcb0730c65d51e1df850271&scene=21#wechat_redirect)
    
*   [外包一年，裸辞冲一波中厂！](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247525633&idx=1&sn=d5d227b0dbff5e086cc03ae27e79a193&chksm=cea12acaf9d6a3dca3871507139a12f220c07de53aad8fdd030e39674a9c08742552879625c8&token=698289097&lang=zh_CN&scene=21#wechat_redirect)
    
*   [程序员深漂 6 年，回西安工作了](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247524803&idx=1&sn=c5df3564f67d3c66d1c5357face5b001&chksm=cea12e08f9d6a71e6889f615918cef6232dea7045a7dee56530f6c86a3c979880dda266cbae2&token=608581386&lang=zh_CN&scene=21#wechat_redirect)
    
*   [3 年 Java 工作经验裸辞](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247522784&idx=1&sn=23d7ff9d51158e0e0cd085c327eb5a60&scene=21#wechat_redirect)
    
*   [上岸美团、华为、字节！](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247520849&idx=1&sn=cc6631b532239ffcf615df7a79c018da&chksm=cea1dd9af9d6548cece4e4819a9ebf5002ace11fe61aed7e10c50a2ac29ae1b525587b508db8&token=2094715744&lang=zh_CN&scene=21#wechat_redirect)
    

👉[《Java 面试指北》](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247525881&idx=1&sn=33d4565668cff95f5ee239c6e1d660fe&chksm=cea12a32f9d6a324432c964bf536181f2d55ee50afba9a53d9b09fa0e1a7126a03bb1dde4c60&token=698289097&lang=zh_CN&scene=21#wechat_redirect)来啦！这是一份教你如何更高效地准备面试的小册，涵盖常见八股文（系统设计、常见框架、分布式、高并发 ......）、优质面经等内容。

👉如果本文对你有帮助的话，欢迎 **点赞 & 在看 & 分享** ，这对我继续分享 & 创作优质文章非常重要。非常感谢！