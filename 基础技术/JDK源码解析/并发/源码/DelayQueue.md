DelayQueue是一个无界阻塞队列，队列中每个元素都有过期时间，当队列获取元素时，只有过期元素才会出队列，队列头元素是快要过期的元素。

DelayQueue内部使用PriorityQueue存放数据，使用ReentrantLock实现线程同步，队列里面的每一个元素要实现Delayed接口，由于每个元素都有一个过期时间，所以要实现获知当前元素还剩下多少时间就过期了的接口，由于内部使用优先队列来实现，所以还要实现元素之间相互比较的接口。

![DelayQueue](.../img/delayQueue.png)