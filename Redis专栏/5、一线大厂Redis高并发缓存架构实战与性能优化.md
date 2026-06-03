**目标**
![](assets/5、一线大厂Redis高并发缓存架构实战与性能优化/file-20260603113009865.png)

# 一、缓存的使用
1、使用缓存是加入超时时间
2、查询数据时，如果查询的是缓存数据对缓存数据进行延期


# 二、缓存各类问题
## 1、缓存击穿（失效）
由于大批量缓存在同一时间失效可能导致大量请求同时穿透缓存直达数据库，可能会造成数据库瞬间压力过大甚至挂掉。
解决方案：对于这种情况我们在批量增加缓存时最好将这一批数据的缓存过期时间设置为一个时间段内的不同时间。设置不同的过期时间
```java
String get(String key) {
    // 从缓存中获取数据
    String cacheValue = cache.get(key);
    // 缓存为空
    if (StringUtils.isBlank(cacheValue)) {
        // 从存储中获取
        String storageValue = storage.get(key);
        cache.set(key, storageValue);
        //设置一个过期时间(300到600之间的一个随机数)
        int expireTime = new Random().nextInt(300)  + 300;
        if (storageValue == null) {
            cache.expire(key, expireTime);
        }
        return storageValue;
    } else {
        // 缓存非空
        return cacheValue;
    }
}
```