# Redis实战

# 实现文章评分排序丶时间排序丶分组功能代码实现

```java
package redis;

import redis.clients.jedis.Jedis;
import redis.clients.jedis.ZParams;

import java.util.*;

public class Chapter01 {
    private static final int ONE_WEEK_IN_SECONDS = 7 * 86400;
    private static final int VOTE_SCORE = 432;
    private static final int ARTICLES_PER_PAGE = 25;

    public static final void main(String[] args) {
        new Chapter01().run();
    }

    public void run() {
        Jedis conn = new Jedis("localhost");
        conn.select(15); //选择使用的数据库


        //生成了文章id
        String articleId = postArticle(
                conn, "username", "A title", "http://www.google.com");
        //输出文章id
        System.out.println("We posted a new article with id: " + articleId);
        System.out.println("Its HASH looks like:");

        //获取到指定的文章的Hash表,遍历文章的每个信息
        Map<String,String> articleData = conn.hgetAll("article:" + articleId);
        for (Map.Entry<String,String> entry : articleData.entrySet()){
            System.out.println("  " + entry.getKey() + ": " + entry.getValue());
        }

        System.out.println();

        //处理其他用户点赞的文章
        articleVote(conn, "other_user", "article:" + articleId);
        //取出对应的文章点赞数进行输出
        String votes = conn.hget("article:" + articleId, "votes");
        System.out.println("We voted for the article, it now has votes: " + votes);
        //判断文章数是否大于1  如果不大于报错
        assert Integer.parseInt(votes) > 1;

        //输出目前分最高的文章
        System.out.println("The currently highest-scoring articles are:");
        //拿到第一页的根据分数排序过后的所有文章
        List<Map<String,String>> articles = getArticles(conn, 1);
        //输出所有文章
        printArticles(articles);
        //文章数量必须大于等于一
        assert articles.size() >= 1;

        //分组相关,将对应的文章加入到对应的分组 new-group
        addGroups(conn, articleId, new String[]{"new-group"});
        System.out.println("We added the article to a new group, other articles include:");
        //获取到该组其他的第一页文章
        articles = getGroupArticles(conn, "new-group", 1);
        //输出该组第一页文章的所有信息
        printArticles(articles);
        //断言
        assert articles.size() >= 1;
    }

    /**将文章加入到hash中,为了方便遍历获取文章的信息
     * 将文章以当前时间+评分加入到有序集合中
     * 将文章以当前时间加入到有序集合中
     * */
    public String postArticle(Jedis conn, String user, String title, String link) {
        //生成一个文章的唯一ID
        String articleId = String.valueOf(conn.incr("article:"));

        //向无序不可重复集合中加入点赞的用户id
        String voted = "voted:" + articleId;
        conn.sadd(voted, user);
        //设置过期时间一周
        conn.expire(voted, ONE_WEEK_IN_SECONDS);

        //拿到当前时间的秒数
        long now = System.currentTimeMillis() / 1000;

        //生成对应的文章
        String article = "article:" + articleId;
        HashMap<String,String> articleData = new HashMap<String,String>();
        articleData.put("title", title);
        articleData.put("link", link);
        articleData.put("user", user);
        articleData.put("now", String.valueOf(now));
        articleData.put("votes", "1");

        //将文章和文章实体放入到hash中
        conn.hmset(article, articleData);
        //添加两个有序集合
        //根据时间+分数存入
        //根据时间存入文章
        conn.zadd("score:", now + VOTE_SCORE, article);
        conn.zadd("time:", now, article);

        return articleId;
    }

    public void articleVote(Jedis conn, String user, String article) {
        //计算出当前时间减去一周的时间剩余的秒数
        long cutoff = (System.currentTimeMillis() / 1000) - ONE_WEEK_IN_SECONDS;
        //根据time取出文章对应的发布时间,如果发布的时间已经小于一周,则无法点赞 直接返回
        if (conn.zscore("time:", article) < cutoff){
            return;
        }

        //获取到当前的文章id
        String articleId = article.substring(article.indexOf(':') + 1);

        //如果用户没有点赞过该文章,则会返回1 代表成功点赞 否则重复点赞
        if (conn.sadd("voted:" + articleId, user) == 1) {
            //使用有序集合将该文章对应的评分score增加
            conn.zincrby("score:", VOTE_SCORE, article);
            //将对应文章的点赞数增加
            conn.hincrBy(article, "votes", 1);
        }
    }


    public List<Map<String,String>> getArticles(Jedis conn, int page) {
        return getArticles(conn, page, "score:");
    }

    public List<Map<String,String>> getArticles(Jedis conn, int page, String order) {
        //0 * 25 = 0
        int start = (page - 1) * ARTICLES_PER_PAGE;
        //end = 24 0 - 24 总共25个数据
        int end = start + ARTICLES_PER_PAGE - 1;

        //取出分数最高的第一页文章25条数据,如果zset集合中总数据量小于25,则显式全部
        Set<String> ids = conn.zrevrange(order, start, end);
        List<Map<String,String>> articles = new ArrayList<Map<String,String>>();
        for (String id : ids){
            Map<String,String> articleData = conn.hgetAll(id);
            articleData.put("id", id);
            articles.add(articleData);
        }

        //取出每个文章的详细信息,封装为一个map 一个map为一个文章的详细信息
        //多个文章组成一个list
        return articles;
    }


    public void addGroups(Jedis conn, String articleId, String[] toAdd) {
        String article = "article:" + articleId;
        for (String group : toAdd) {
            //将对应的文章加入到对应的分组里面 比如讲述如何实战Redis属于Redis分组 也属于编程分组
            conn.sadd("group:" + group, article);
        }
    }

    public List<Map<String,String>> getGroupArticles(Jedis conn, String group, int page) {
        return getGroupArticles(conn, group, page, "score:");
    }

    //
    public List<Map<String,String>> getGroupArticles(Jedis conn, String group, int page, String order) {
        //score:new-group
        String key = order + group;
        //如何集合中
        if (!conn.exists(key)) {
            ZParams params = new ZParams().aggregate(ZParams.Aggregate.MAX);
            //将有序集合group一组的文章 和 有序集合score分数的文章进行交集合并
            conn.zinterstore(key, params, "group:" + group, order);
            conn.expire(key, 60);
        }
        return getArticles(conn, page, key);
    }

    //输出传入的所有文章信息
    private void printArticles(List<Map<String,String>> articles){
        for (Map<String,String> article : articles){
            System.out.println("  id: " + article.get("id"));
            for (Map.Entry<String,String> entry : article.entrySet()){
                if (entry.getKey().equals("id")){
                    continue;
                }
                System.out.println("    " + entry.getKey() + ": " + entry.getValue());
            }
        }
    }
}
```

# Redis数据结构简介

## Redis中的字符串

![image-20210424170600794](D:\学习笔记\image-20210424170600794.png)

```bash
get key 获取存储在给定键中的值
set key 设置存储在给定键中的值
del key 删除存储在给定键中的值
```

## Redis中的列表

![image-20210424170706958](D:\学习笔记\image-20210424170706958.png)

```bash
RPUSH key value  将给定值推入列表的右端
LRANGE key start stop 获取列表中给定范围的值
LINDEX key index 获取列表中指定下标的值
LPOP key 从列表的左端弹出一个值，并且返回被弹出的值
```

## Redis的集合

![image-20210424170942130](D:\学习笔记\image-20210424170942130.png)

``` bash
SADD key value 将给定的元素添加到指定的集合
SMEMBERS key   返回集合包含的所有元素
SISMEMBER key member 检查指定的元素是否在集合中
SREM key member...  如果给定的多个元素存在集合中,那么移除这些元素
```

## Redis的散列

![image-20210424171213621](D:\学习笔记\image-20210424171213621.png)

```bash
HSET key field value 在散列表中指定关联的键值对
HGET key field 获取指定散列键的值
HGETALL key 获取散列包含的所有键值对
HDEL key field 如果给定的键存在于散列中，那么移除这个键
```

## Redis的有序集合

![image-20210424171518646](D:\学习笔记\image-20210424171518646.png)

```bash
ZADD key score member 将一个带有给定分值的成员添加到有序集合里面
ZRANGE key start stop 根据元素在有序排列中的位置，从有序集合里面获取多个元素
ZRANGEBYSCORE key min max 获取集合中给定分值范围内的所有元素
ZREM key member 如果给定的成员存在该集合中，那么进行移除
```

# 使用Redis构建Web应用

## 登录和cookie缓存丶Redis实现购物车丶网页缓存丶数据行缓存

```java
package redis;
import redis.clients.jedis.Jedis;
import redis.clients.jedis.Tuple;

import java.net.MalformedURLException;
import java.net.URL;
import java.util.*;

public class Chapter02 {
    public static final void main(String[] args)
            throws InterruptedException
    {
        new Chapter02().run();
    }
    public void run()
            throws InterruptedException
    {
        Jedis conn = new Jedis("localhost");
        conn.select(15);

        testLoginCookies(conn);
        testShopppingCartCookies(conn);
        testCacheRows(conn);
        testCacheRequest(conn);
    }

    public void testLoginCookies(Jedis conn)
            throws InterruptedException
    {
        System.out.println("\n----- testLoginCookies -----");
        String token = UUID.randomUUID().toString();


        updateToken(conn, token, "username", "itemX");
        System.out.println("We just logged-in/updated token: " + token);
        System.out.println("For user: 'username'");
        System.out.println();

        System.out.println("What username do we get when we look-up that token?");
        String r = checkToken(conn, token);
        System.out.println(r);
        System.out.println();
        assert r != null;

        System.out.println("Let's drop the maximum number of cookies to 0 to clean them out");
        System.out.println("We will start a thread to do the cleaning, while we stop it later");

        CleanSessionsThread thread = new CleanSessionsThread(0);
        thread.start();
        Thread.sleep(1000);
        thread.quit();
        Thread.sleep(2000);
        if (thread.isAlive()){
            throw new RuntimeException("The clean sessions thread is still alive?!?");
        }

        long s = conn.hlen("login:");
        System.out.println("The current number of sessions still available is: " + s);
        assert s == 0;
    }

    public void testShopppingCartCookies(Jedis conn)
            throws InterruptedException
    {
        System.out.println("\n----- testShopppingCartCookies -----");
        String token = UUID.randomUUID().toString();

        System.out.println("We'll refresh our session...");
        updateToken(conn, token, "username", "itemX");
        System.out.println("And add an item to the shopping cart");
        addToCart(conn, token, "itemY", 3);
        Map<String,String> r = conn.hgetAll("cart:" + token);
        System.out.println("Our shopping cart currently has:");
        for (Map.Entry<String,String> entry : r.entrySet()){
            System.out.println("  " + entry.getKey() + ": " + entry.getValue());
        }
        System.out.println();

        assert r.size() >= 1;

        System.out.println("Let's clean out our sessions and carts");
        CleanFullSessionsThread thread = new CleanFullSessionsThread(0);
        thread.start();
        Thread.sleep(1000);
        thread.quit();
        Thread.sleep(2000);
        if (thread.isAlive()){
            throw new RuntimeException("The clean sessions thread is still alive?!?");
        }

        r = conn.hgetAll("cart:" + token);
        System.out.println("Our shopping cart now contains:");
        for (Map.Entry<String,String> entry : r.entrySet()){
            System.out.println("  " + entry.getKey() + ": " + entry.getValue());
        }
        assert r.size() == 0;
    }

    public void testCacheRows(Jedis conn)
            throws InterruptedException
    {
        System.out.println("\n----- testCacheRows -----");
        System.out.println("First, let's schedule caching of itemX every 5 seconds");
        scheduleRowCache(conn, "itemX", 5);
        System.out.println("Our schedule looks like:");
        Set<Tuple> s = conn.zrangeWithScores("schedule:", 0, -1);
        for (Tuple tuple : s){
            System.out.println("  " + tuple.getElement() + ", " + tuple.getScore());
        }
        assert s.size() != 0;

        System.out.println("We'll start a caching thread that will cache the data...");

        CacheRowsThread thread = new CacheRowsThread();
        thread.start();

        Thread.sleep(1000);
        System.out.println("Our cached data looks like:");
        String r = conn.get("inv:itemX");
        System.out.println(r);
        assert r != null;
        System.out.println();

        System.out.println("We'll check again in 5 seconds...");
        Thread.sleep(5000);
        System.out.println("Notice that the data has changed...");
        String r2 = conn.get("inv:itemX");
        System.out.println(r2);
        System.out.println();
        assert r2 != null;
        assert !r.equals(r2);

        System.out.println("Let's force un-caching");
        scheduleRowCache(conn, "itemX", -1);
        Thread.sleep(1000);
        r = conn.get("inv:itemX");
        System.out.println("The cache was cleared? " + (r == null));
        assert r == null;

        thread.quit();
        Thread.sleep(2000);
        if (thread.isAlive()){
            throw new RuntimeException("The database caching thread is still alive?!?");
        }
    }

    public void testCacheRequest(Jedis conn) {
        System.out.println("\n----- testCacheRequest -----");
        String token = UUID.randomUUID().toString();

        Callback callback = new Callback(){
            public String call(String request){
                return "content for " + request;
            }
        };

        updateToken(conn, token, "username", "itemX");
        String url = "http://test.com/?item=itemX";
        System.out.println("We are going to cache a simple request against " + url);
        String result = cacheRequest(conn, url, callback);
        System.out.println("We got initial content:\n" + result);
        System.out.println();

        assert result != null;

        System.out.println("To test that we've cached the request, we'll pass a bad callback");
        String result2 = cacheRequest(conn, url, null);
        System.out.println("We ended up getting the same response!\n" + result2);

        assert result.equals(result2);

        assert !canCache(conn, "http://test.com/");
        assert !canCache(conn, "http://test.com/?item=itemX&_=1234536");
    }

    public String checkToken(Jedis conn, String token) {
        return conn.hget("login:", token);
    }

    public void updateToken(Jedis conn, String token, String user, String item) {
        long timestamp = System.currentTimeMillis() / 1000;
        conn.hset("login:", token, user);
        conn.zadd("recent:", timestamp, token);
        if (item != null) {
            conn.zadd("viewed:" + token, timestamp, item);
            conn.zremrangeByRank("viewed:" + token, 0, -26);
            conn.zincrby("viewed:", -1, item);
        }
    }

    public void addToCart(Jedis conn, String session, String item, int count) {
        if (count <= 0) {
            conn.hdel("cart:" + session, item);
        } else {
            conn.hset("cart:" + session, item, String.valueOf(count));
        }
    }

    public void scheduleRowCache(Jedis conn, String rowId, int delay) {
        conn.zadd("delay:", delay, rowId);
        conn.zadd("schedule:", System.currentTimeMillis() / 1000, rowId);
    }

    public String cacheRequest(Jedis conn, String request, Callback callback) {
        if (!canCache(conn, request)){
            return callback != null ? callback.call(request) : null;
        }

        String pageKey = "cache:" + hashRequest(request);
        String content = conn.get(pageKey);

        if (content == null && callback != null){
            content = callback.call(request);
            conn.setex(pageKey, 300, content);
        }

        return content;
    }

    public boolean canCache(Jedis conn, String request) {
        try {
            URL url = new URL(request);
            HashMap<String,String> params = new HashMap<String,String>();
            if (url.getQuery() != null){
                for (String param : url.getQuery().split("&")){
                    String[] pair = param.split("=", 2);
                    params.put(pair[0], pair.length == 2 ? pair[1] : null);
                }
            }

            String itemId = extractItemId(params);
            if (itemId == null || isDynamic(params)) {
                return false;
            }
            Long rank = conn.zrank("viewed:", itemId);
            return rank != null && rank < 10000;
        }catch(MalformedURLException mue){
            return false;
        }
    }

    public boolean isDynamic(Map<String,String> params) {
        return params.containsKey("_");
    }

    public String extractItemId(Map<String,String> params) {
        return params.get("item");
    }

    public String hashRequest(String request) {
        return String.valueOf(request.hashCode());
    }

    public interface Callback {
        public String call(String request);
    }

    public class CleanSessionsThread
            extends Thread
    {
        private Jedis conn;
        private int limit;
        private boolean quit;

        public CleanSessionsThread(int limit) {
            this.conn = new Jedis("localhost");
            this.conn.select(15);
            this.limit = limit;
        }

        public void quit() {
            quit = true;
        }

        public void run() {
            while (!quit) {
                long size = conn.zcard("recent:");
                if (size <= limit){
                    try {
                        sleep(1000);
                    }catch(InterruptedException ie){
                        Thread.currentThread().interrupt();
                    }
                    continue;
                }

                long endIndex = Math.min(size - limit, 100);
                Set<String> tokenSet = conn.zrange("recent:", 0, endIndex - 1);
                String[] tokens = tokenSet.toArray(new String[tokenSet.size()]);

                ArrayList<String> sessionKeys = new ArrayList<String>();
                for (String token : tokens) {
                    sessionKeys.add("viewed:" + token);
                }

                conn.del(sessionKeys.toArray(new String[sessionKeys.size()]));
                conn.hdel("login:", tokens);
                conn.zrem("recent:", tokens);
            }
        }
    }

    public class CleanFullSessionsThread
            extends Thread
    {
        private Jedis conn;
        private int limit;
        private boolean quit;

        public CleanFullSessionsThread(int limit) {
            this.conn = new Jedis("localhost");
            this.conn.select(15);
            this.limit = limit;
        }

        public void quit() {
            quit = true;
        }

        public void run() {
            while (!quit) {
                long size = conn.zcard("recent:");
                if (size <= limit){
                    try {
                        sleep(1000);
                    }catch(InterruptedException ie){
                        Thread.currentThread().interrupt();
                    }
                    continue;
                }

                long endIndex = Math.min(size - limit, 100);
                Set<String> sessionSet = conn.zrange("recent:", 0, endIndex - 1);
                String[] sessions = sessionSet.toArray(new String[sessionSet.size()]);

                ArrayList<String> sessionKeys = new ArrayList<String>();
                for (String sess : sessions) {
                    sessionKeys.add("viewed:" + sess);
                    sessionKeys.add("cart:" + sess);
                }

                conn.del(sessionKeys.toArray(new String[sessionKeys.size()]));
                conn.hdel("login:", sessions);
                conn.zrem("recent:", sessions);
            }
        }
    }

    public class CacheRowsThread
            extends Thread
    {
        private Jedis conn;
        private boolean quit;

        public CacheRowsThread() {
            this.conn = new Jedis("localhost");
            this.conn.select(15);
        }

        public void quit() {
            quit = true;
        }

        public void run() {
            //Gson gson = new Gson();
            while (!quit){
                Set<Tuple> range = conn.zrangeWithScores("schedule:", 0, 0);
                Tuple next = range.size() > 0 ? range.iterator().next() : null;
                long now = System.currentTimeMillis() / 1000;
                if (next == null || next.getScore() > now){
                    try {
                        sleep(50);
                    }catch(InterruptedException ie){
                        Thread.currentThread().interrupt();
                    }
                    continue;
                }

                String rowId = next.getElement();
                double delay = conn.zscore("delay:", rowId);
                if (delay <= 0) {
                    conn.zrem("delay:", rowId);
                    conn.zrem("schedule:", rowId);
                    conn.del("inv:" + rowId);
                    continue;
                }

                Inventory row = Inventory.get(rowId);
                conn.zadd("schedule:", now + delay, rowId);
              //  conn.set("inv:" + rowId, gson.toJson(row));
            }
        }
    }

    public static class Inventory {
        private String id;
        private String data;
        private long time;

        private Inventory (String id) {
            this.id = id;
            this.data = "data to cache...";
            this.time = System.currentTimeMillis() / 1000;
        }

        public static Inventory get(String id) {
            return new Inventory(id);
        }
    }
}
```

# Redis命令

## 字符串

字符串可以存储以下三种类型的数据

- 字节串
- 整数
- 浮点数

```bash
INCR key-name 将键存储的值加上1
DECR key-name 将键存储的值减上1
INCRBY key-name amount 将键存储的值加上amount
DECRBY key-name amount 将键存储的值减上amount
INCRBYFLOAT key-name amount 将键存储的值加上浮点数amount
```

![image-20210504123357240](D:\学习笔记\image-20210504123357240.png)

供Redis处理子串和二进制位的命令

```bash
APPEDN key-name value 将值value追加到给定的key-name当前存储的值的末尾
GETRANGE key-name start end 获取一个根据key并且范围的子串,包括start和end
SETRANGE key-name offset value #将从start偏移量开始的子串设置为value
GETBIT key-name offset #将字符串看作是二进制位串,并且返回位串中偏移量为offset的二进制位的值
SETBIT key-name offset value #将字符串看作是二进制位串，并将位串中偏移量为offset的二进制位的值为value
BITCOUNT key-name [start end]#统计二进制串里面值为1的二进制位的数量
BITOP operation dest-key key-name[key-name...] # 对一个或者多个二进制串执行包括并(AND),或(OR)，异或(XOR),非(NOT)在内的任何一种按位运算操作，并且将结果保存在dest-key中	
```

![image-20210504124216598](D:\学习笔记\image-20210504124216598.png)

## 列表

```bash
RPUSH key-name value[..] 将一个或者多个值推入列表的右端
LPUSH key-name value[..] 将一个或者多个值推入列表的左端
RPOP key-name #移除并且返回列表最右端的元素
LROP key-name #移除并且返回列表最左端的元素
LINDEX key-name offset #返回列表中偏移量为offset的元素
LRANGE key-name start-end #返回列表中start偏移量到end偏移量的元素,包含end和start位置上的元素
LTRIM key-name start-end #对列表进行修建,只保留start到end的元素，包含start和end位置上的元素
```

![image-20210504175743398](D:\学习笔记\image-20210504175743398.png)

```bash
BLPOP key-name[key-name...] timeout # 从一个非空列表中弹出位于最左端的元素，或者在timeout秒之后阻塞并等待可弹出的元素出现
BRPOP key-name[key-name...] timeout # 从一个非空列表中弹出位于最右端的元素，或者在timeout秒之后阻塞并等待可弹出的元素
RPOLPUSH source-key dest-key timeout # 从source-key列表中弹出最右端的元素并且将这个元素推入dest-key列表的最左端，如果source-key为空，那么在timeout秒之内阻塞并且等待可弹出的元素出现
```

![image-20210504180159982](D:\学习笔记\image-20210504180159982.png)

## 集合

```bash
SADD key-name item[item...] #将一个或者多个元素添加到集合中，并且返回被添加元素当中原本不存在集合里面的数据
SREM key-name item[item...] #将一个或者多个元素移除,并且返回被移除元素的数量
SISMEMBER key-name item #检查该元素是否存在集合中
SCARD key-name #返回集合包含的所有元素数量
SMEMERS key-name #返回集合包含的所有元素
SRANDMEMBER key-name[count] #从集合里面随机的返回一个或者多个元素，当count为正数的时候返回的元素不会重复，当为负数的时候可能会出现重复
SPOP key-name # 随机的移除集合中的一个元素，并且返回被移除的元素
SMOVE source-key dest-key item #如果集合source-key包含元素item,那么集合source-key里面移除元素item,并将item元素加入到dest-key集合中，如果被成功移除返回1，否则返回0
```

![image-20210504181046190](D:\学习笔记\image-20210504181046190.png)

```bash
SDIFF key-name[key-name...] # 返回那些存在于第一个集合，但不存在于其他集合中的元素,数学中的差集运算。
SDIFFSTORE dest-key key-name[key-name...] # 将差集运算存储到dest-key中
SINTER key-name[key-name...] # 返回数学中交集运算的结果
SINTERSTROE dest-key key-name[key-name...]# 将数学中交集运算的结果存储到dest-key中
SUNION key-name[key-name...] # 返回并集后的结果
SUNIONSTORE destkey key-name[key-name...]# 将并集后的结果存储到dest-key集合中
```

![image-20210504181757828](D:\学习笔记\image-20210504181757828.png)

## 散列

```bash
HMGET key-name key[key...] # 从散列里面获取一个或者多个指定key的值
HMSET key-name key value[key value...] # 从散列里面设置一个或者多个key value的值
HDEL key-name key[key...] # 从散列里面删除一个或者多个键值对,返回成功找到并删除成功的键值对数量
HLEN key-name #返回散列包含的键值对数量
```

![image-20210504182331919](D:\学习笔记\image-20210504182331919.png)

```bash
HEXISTS key-name key #检查给定键中是否存在于散列
HKEYS key-name # 获取散列中的所有键
KVALS key-name # 获取散列中的所有值
HGETALL key-name # 获取散列包含的所有键值对
HINCRBY key-name key increment # 将键key存储的值加上整数increment
HINCRBYFLOAT key-name key increment # 将键key存储的值加上浮点数increment
```

![image-20210504182644352](D:\学习笔记\image-20210504182644352.png)

## 有序集合

```bash
ZADD key-name score member[score member...] #将带有给定分值的成员添加到有序集合中
ZREM key-name member[member...] # 从有序集合中移除给定的成员，并且返回被移除成员的数量
ZCARD key-name # 返回有序集合包含的成员数量
ZINCRBY key-name increment member # 将member成员的分值加上指定的increment
ZCOUNT key-name min max # 返回分值介于min和max之间的成员数量
ZRANK key-name member # 返回成员member在有序集合中的排名
ZSCORE key-name member # 返回成员member的分值
ZRANGE key-name start stop[WITHSCORE] # 返回有序集合中排名介于start到stop之间的成员，如果指定了WITHSCORE选项，那么命令也会将成员的分值也一并返回
```

![image-20210504195048692](D:\学习笔记\image-20210504195048692.png)

```bash
ZREVRANK key-name member # 返回有序集合里成员member的排名，成员按照分值从小到大排列
ZREVRANGE key-name start stop [withScore] # 返回有序集合给定排名范围内的成员，成员按照分值从大到小排列
ZRANGEBYSCORE key min max [withScores][Limit offset count]  # 返回有序集合中，分值介于min到max的成员
ZREVRANGEBYSCORE key max min[withScores][Limit offset count] # 获取有序集合中分值介于min和max之间的所有成员，并按照分值从小到达的顺序来返回它们
ZREMRANGEBYRANK key-name start stop # 移除start到stop范围的所有成员
ZREMRANGEBYSCORE key-name min max # 移除有序集合中分值介于min和max之间的所有成员
ZINTERSTORE dest-key key-count key[key...] # 对有序集合执行类似集合的交集运算
ZUNIONSTORE dest-key key-count key[key...] # 对有序集合执行类似集合的并集运算
```

![image-20210504200118102](D:\学习笔记\image-20210504200118102.png)

![image-20210504200127472](D:\学习笔记\image-20210504200127472.png)

![image-20210504200133311](D:\学习笔记\image-20210504200133311.png)

![image-20210504200141574](D:\学习笔记\image-20210504200141574.png)

## 发布与订阅

```bash
SUBSCRIBE channel[channel...] # 订阅给定的一个或多个频道
UNSUBSCRIBE [channel[channel...]] # 订阅给定的一个或者多个频道，如果执行没指定频道，则退订所有频道
PUBLISH channel message # 向给定频道发送消息
PSUBSCRIBE pattern[pattern ...] # 订阅与给定模式相匹配的所有频道
PUNSUBSCRIBE [pattern[pattern...]] # 退订给定的模式
```

## 排序

```bash
SORT source-key[BY pattern][Limit offset count][ASC|DESC][ALPHA][STORE dest-key] # 根据给定的选项，对输入列表丶集合或者有序集合进行排序，然后返回
```

![image-20210504202533022](D:\学习笔记\image-20210504202533022.png)

## 事务

watch丶multi丶exec丶unwatch和discard

## 键的过期时间

```bash
PERSIST key-name 移除key的过期时间
TTL key-name # 查看给定键距离过期还有多久
EXPIRE key-name seconds # 将给定的键设置过期时间
PTTL key-name # 查看过期时间 毫秒
PEXPIRE key-name millseconds # 将给定的键设置过期时间 毫秒
PEXPIPEAT key-name timestamp-milliseconds # 将一个毫秒级精度的unix时间戳设置为给定的过期时间
```

![image-20210504203137215](D:\学习笔记\image-20210504203137215.png)

