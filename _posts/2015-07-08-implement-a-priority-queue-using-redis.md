---
title: 基于 Redis 实现优先队列
layout: post
---


最近有一个项目需要使用队列传递消息，并在给定时间发布消息。消息在生成时会带上一个发布时间，当达到发布时间时需要立即发布该消息，且消息生成和发布分为位于两个不同的微服务中。在以前的项目中，我们会使用RabbitMQ用于消息传递。但是，这个项目中由于需要定时发布消息，因此该消息队列是一个优先队列，无法使用
RabbitMQ 。我们的实现方式是使用 redis [sorted
set](http://redis.io/commands#sorted_set) 实现优先队列。
Redis sorted set 跟 set 的区别是它有一个 score 值，它有如下特点：

-   元素都是唯一的。因此，如果 “foo” 是其中一个元素，当再向该 set
    中放入一个 “foo”时，”foo" 只会在 set 中保存一个
-   每一个元素都有一个 score 值
-   当 set 作为队列读取时，score 值低的元素先被取出

项目中的message可以简单描述为：

    public class IdTime {
        private int id;
        private long time;
    }

主要实现优先队列的add, remove, peek 操作，基于
redis 的以下几个命令：zadd, zrangeByScoreWithScores, zrem实现，并用 time 属性
来计算score值。实现代码如下：

    public class RedisPriorityQueue { 
        private static final String QUEUE_NAME = "publish_message";
        
        public int size() {
            Jedis jedis = borrowReadJedis();
            try {
                return jedis.llen(QUEUE_NAME).intValue();
            } finally {
                returnReadJedis(jedis);
            }
        }
        
        public IdTime peek() {
            Jedis jedis = borrowWriteJedis();
            try {
                Set<Tuple> set = jedis.zrangeWithScores(QUEUE_NAME, 0, 0);
                if (CollectionUtils.isEmpty(set)) {
                    return null;
                }

                Tuple tuple = set.iterator().next();
                return tuple2IdTime(tuple);
             } finally { 
                returnReadJedis(jedis); 
             } 
        }
 	
		public void add(IdTime idTime) {
			 Jedis jedis = borrowWriteJedis();
			 try {
			 String field = String.valueOf(idTime.getId());
			 double score = toScore(idTime.getTime());
			 jedis.zadd(QUEUE_NAME, score, field);
			 } finally {
			 returnWriteJedis(jedis);
			 }
		 }
 	     
         public boolean remove(IdTime idTime) {
			 Jedis jedis = borrowWriteJedis();
			 try {
			 String field = String.valueOf(idTime.getId());
			 long ret = jedis.zrem(QUEUE_NAME, field);

			 return ret > 0;
			 } finally {
			 returnWriteJedis(jedis);
			 }
		 }
	 
		 public IdTime poll() {
			 Jedis jedis = borrowWriteJedis();
			 try {
			 List<Object> unformatted;
			 IdTime idTime;
			 do {
				 jedis.watch(QUEUE_NAME);
				 Set<Tuple> set = jedis.zrangeWithScores(QUEUE_NAME, 0, 0);
				 if (CollectionUtils.isEmpty(set)) {
					 return null;
				 }

				 Transaction tran = jedis.multi();
				 Tuple tuple = set.iterator().next();
				 idTime = tuple2IdTime(tuple);
				 String field = String.valueOf(idTime.getId());
				 tran.zrem(QUEUE_NAME, field);
				 unformatted = tran.exec();
			 } while (unformatted == null);

				return idTime;
			} finally {
			returnReadJedis(jedis);
			}
    	 }
    
    	 private double toScore(long time) {
             return (double) (time << 8);
    	 }
    
    	 private long scoreToTime(double score) {
             return ((long) score >> 8);
    	 }
    
    	 private IdTime tuple2IdTime(Tuple tuple) {
    	     if (tuple  == null) {
    		 return null;
     	     }
     
     	     int id = NumberUtils.toInt(tuple.getElement());
     	     long time = scoreToTime(tuple.getScore());
     	     return new IdTime(id, time);
    	 }
    }
