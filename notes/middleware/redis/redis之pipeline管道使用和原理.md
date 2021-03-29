
# redis之pipeline管道使用和原理

0.175s  

默认53个队列，一次提交

``` java
    private Boolean setLock(String event) {
        String lockKey = "";
        logger.info("lockKey : {}" , lockKey);
        SessionCallback<Boolean> sessionCallback = new SessionCallback<Boolean>() {
            List<Object> exec = null;
            @Override
            @SuppressWarnings("unchecked")
            public Boolean execute(RedisOperations operations) throws DataAccessException{
                operations.multi();
                redisTemplateForGeneralize.opsForValue().setIfAbsent(lockKey,JSON.toJSONString(event));
                redisTemplateForGeneralize.expire(lockKey,finalDefaultTTLwithKey, TimeUnit.SECONDS);
                exec = operations.exec();
                if(exec.size() > 0) {
                    return (Boolean) exec.get(0);
                }
                return false;
            }
        };
        return redisTemplateForGeneralize.execute(sessionCallback);
    }

    //管道利用，批量新增数据到redis
```