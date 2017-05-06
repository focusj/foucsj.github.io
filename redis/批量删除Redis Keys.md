```

redis-cli -h ec-diamond-redis.1ck6sb.0001.usw2.cache.amazonaws.com -n 1 KEYS "order_statemachine:*" | xargs redis-cli -h ec-diamond-redis.1ck6sb.0001.usw2.cache.amazonaws.com -n 1 DEL

```