
---
layout: post
title:  "Kafka-Produce流程"
date:   2018-07-09 12:19:12 +0800
tags:
      - Kafka
---
Kafka是一个消息订阅系统，通过接收消息顺序存储在本地磁盘，以便后端应用从kafka读取消息。本文基于Kafka 0.10.0版本对kafka的消息发送流程进行分析： 

#### 确认消息要发送到哪个分区：

Record的partition确认方法：

record的partition为非空且合法（0 =< partition <= topic.partitions.size）时，直接使用record中的partition

record的partition为空时，通过partitioner的的partition方法得到record的partition。此处的分区选择已经和之前版本的分区选择发生了变化，不再是选中一个partition使用10min，然后再次选择一个partition，而是使用如下的方案（当然用户可以根据需求自定义通过partitioner.class来设置自己的partitioner）：


    //所谓的availablepartitions也就是存在leader的partition
    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
        List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
        int numPartitions = partitions.size();
        if (keyBytes == null) {
            int nextValue = counter.getAndIncrement();
            List<PartitionInfo> availablePartitions = cluster.availablePartitionsForTopic(topic);
            //key为空时，使用一个AtomicInteger的累加值对可用partitions的值取
            if (availablePartitions.size() > 0) {
                int part = DefaultPartitioner.toPositive(nextValue) % availablePartitions.size();
                return availablePartitions.get(part).partition();
            } else {
                //可用partitions为o个时，同样使用AtomicInteger的累加值对topic.partitions.size取余
                // no partitions are available, give a non-available partition
                return DefaultPartitioner.toPositive(nextValue) % numPartitions;
            }
        } else {
        //key为非空时，直接使用key的hash值对topic.partitions.size取余
            // hash the keyBytes to choose a partition
            return DefaultPartitioner.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
        }
    }
    
#### 将消息放置recordBatch中

RecordAccumulator 作为一个queue将消息累计放置在memoryRecords中，以便成批发送至server

调用RecordAccumulator的append方法，将消息放置在对应partition的recordBatch中，以待发送，主要逻辑如下：

    public FutureRecordMetadata tryAppend(long timestamp, byte[] key, byte[] value, Callback callback, long now) {
        if (!this.records.hasRoomFor(key, value)) {
            return null;
        } else {
            //消息放入records中
            long checksum = this.records.append(offsetCounter++, timestamp, key, value);
            this.maxRecordSize = Math.max(this.maxRecordSize, Record.recordSize(key, value));
            //此时会更新lastAppendTime
            this.lastAppendTime = now;
            FutureRecordMetadata future = new FutureRecordMetadata(this.produceFuture, this.recordCount,
                                                                   timestamp, checksum,
                                                                   key == null ? -1 : key.length,
                                                                   value == null ? -1 : value.length);
            if (callback != null)
                thunks.add(new Thunk(callback, future));
            this.recordCount++;
            return future;
        }
    }
    
此时仅仅是将消息record放入memoryRecords中，然而消息是何时发送出去的呢

### 独立消息发送sender线程

KafkaProducer在初始化时会初始化并启动名为kafka-producer-network-thread的线程。线程的run方法会循环执行如下run(long now)方法：

    void run(long now) {
        Cluster cluster = metadata.fetch();
        // get the list of partitions with data ready to send
        RecordAccumulator.ReadyCheckResult result = this.accumulator.ready(cluster, now);

        // if there are any partitions whose leaders are not known yet, force metadata update
        // 如果有partition找不到leader，则需要设置重新获取metadata的标志
        if (result.unknownLeadersExist)
            this.metadata.requestUpdate();

        // remove any nodes we aren't ready to send to
        Iterator<Node> iter = result.readyNodes.iterator();
        long notReadyTimeout = Long.MAX_VALUE;
        // 根据netWorkClient中维持的与各broker的链接信息，去除部分链接状态无效的readyNodes
        while (iter.hasNext()) {
            Node node = iter.next();
            if (!this.client.ready(node, now)) {
                iter.remove();
                notReadyTimeout = Math.min(notReadyTimeout, this.client.connectionDelay(node, now));
            }
        }

        // 将每个batch要发送的消息与每个ready节点对应起来
        Map<Integer, List<RecordBatch>> batches = this.accumulator.drain(cluster,
                                                                         result.readyNodes,
                                                                         this.maxRequestSize,
                                                                         now);
        if (guaranteeMessageOrder) {
            // Mute all the partitions drained
            for (List<RecordBatch> batchList : batches.values()) {
                for (RecordBatch batch : batchList)
                    this.accumulator.mutePartition(batch.topicPartition);
            }
        }

        // 将一些长时间没有发送出去的batch，置为expire状态
        List<RecordBatch> expiredBatches = this.accumulator.abortExpiredBatches(this.requestTimeout, now);
        // update sensors
        for (RecordBatch expiredBatch : expiredBatches)
            this.sensors.recordErrors(expiredBatch.topicPartition.topic(), expiredBatch.recordCount);

        sensors.updateProduceRequestMetrics(batches);
        
        // 为每一个node创建producerequest
        List<ClientRequest> requests = createProduceRequests(batches, now);
        // If we have any nodes that are ready to send + have sendable data, poll with 0 timeout so this can immediately
        // loop and try sending more data. Otherwise, the timeout is determined by nodes that have partitions with data
        // that isn't yet sendable (e.g. lingering, backing off). Note that this specifically does not include nodes
        // with sendable data that aren't ready to send since they would cause busy looping.
        long pollTimeout = Math.min(result.nextReadyCheckDelayMs, notReadyTimeout);
        if (result.readyNodes.size() > 0) {
            log.trace("Nodes with data ready to send: {}", result.readyNodes);
            log.trace("Created {} produce requests: {}", requests.size(), requests);
            pollTimeout = 0;
        }
        //发送request
        for (ClientRequest request : requests)
            client.send(request, now);

        // if some partitions are already ready to be sent, the select time would be 0;
        // otherwise if some partition already has some data accumulated but not ready yet,
        // the select time will be the time difference between now and its linger expiry time;
        // otherwise the select time will be the time difference between now and the metadata expiry time;
        // 发送获取metadata请求，handle各种发request送和返回的response，handle各种连接状态
        this.client.poll(pollTimeout, now);
    }

可以看出，该run方法主要有如下流程：

* 获取cluster信息，cluster类包含了topic，broker等信息，可以用来表示一个kafka集群
* 获取可以发送消息的Node节点（readyNode），如果有partition找不到leader，则标志需要更新metadata，如果node连接异常，则从readyNode中去除
*  找出本次需要发送消息的Node和要发送至该节点的recordBatch
*  清理“长时间”没有发送出去的recordBatch
*  创建并发送produce 请求
*  发送获取metadata请求，handle各种发request送和返回的response，handle各种连接状态



#### Cluster 信息
cluster类包含了集群的broker，topic，partition等信息，客户端可以通过cluster类完成与集群的交互，如下：

    private final boolean isBootstrapConfigured;
    private final List<Node> nodes;
    private final Set<String> unauthorizedTopics;
    private final Map<TopicPartition, PartitionInfo> partitionsByTopicPartition;
    private final Map<String, List<PartitionInfo>> partitionsByTopic;
    private final Map<String, List<PartitionInfo>> availablePartitionsByTopic;
    private final Map<Integer, List<PartitionInfo>> partitionsByNode;
    private final Map<Integer, Node> nodesById;

#### 查找readyNodes

    //先找出readyNodes 
    public ReadyCheckResult ready(Cluster cluster, long nowMs) {
        Set<Node> readyNodes = new HashSet<>();
        long nextReadyCheckDelayMs = Long.MAX_VALUE;
        boolean unknownLeadersExist = false;
        boolean exhausted = this.free.queued() > 0;
        for (Map.Entry<TopicPartition, Deque<RecordBatch>> entry : this.batches.entrySet()) {
            TopicPartition part = entry.getKey();
            Deque<RecordBatch> deque = entry.getValue();
            Node leader = cluster.leaderFor(part);
            if (leader == null) {
                unknownLeadersExist = true;
            } else if (!readyNodes.contains(leader) && !muted.contains(part)) {
                synchronized (deque) {
                    RecordBatch batch = deque.peekFirst();
                    if (batch != null) {
                    //判断是否需要backoff attempts lastAttemptMs的值是在上次发送失败后，handleResponse时更新
                        boolean backingOff = batch.attempts > 0 && batch.lastAttemptMs + retryBackoffMs > nowMs;
                        long waitedTimeMs = nowMs - batch.lastAttemptMs;
                        long timeToWaitMs = backingOff ? retryBackoffMs : lingerMs;
                        long timeLeftMs = Math.max(timeToWaitMs - waitedTimeMs, 0);
                        boolean full = deque.size() > 1 || batch.records.isFull();
                        boolean expired = waitedTimeMs >= timeToWaitMs;
                        // 根据如下条件判断该节点是否有可以发送的消息
                        boolean sendable = full || expired || exhausted || closed || flushInProgress();
                        if (sendable && !backingOff) {
                            readyNodes.add(leader);
                        } else {
                            // Note that this results in a conservative estimate since an un-sendable partition may have
                            // a leader that will later be found to have sendable data. However, this is good enough
                            // since we'll just wake up and then sleep again for the remaining time.
                            nextReadyCheckDelayMs = Math.min(timeLeftMs, nextReadyCheckDelayMs);
                        }
                    }
                }
            }
        }

        return new ReadyCheckResult(readyNodes, nextReadyCheckDelayMs, unknownLeadersExist);
    }
    
    此处的核心逻辑为如何判断一个节点本次可以发送消息：
     （  full || expired || exhausted || closed || flushInProgress() ） && （！backingOff）
     full : 该partition有多于一个batch或者当前batch处于full状态
     expired ： 已经等待的时间(当前时间 -  上次尝试时间)  >  本身需要等待的时间（if(需要backoff) ： retry的backoff else lingerMs）
     exhausted ： 有消息处在分配状态，内存还没有回收
     closed ： 此recordAccuulator已经被关闭
     flushInProgress : 业务层调用了flush方法
     

#### 找出Node以及该Node要发送的消息的对应关系

过程 ： 遍历所有的readyNodes，针对每个readyNode，找出该Node上面的partition，如果该partition存在queued的recordBatch，且符合发送条件（size 超过限制 且针对该Node已经有partition要发送，则跳过该batch的发送）则将该消息放入List<RecordBatch>中返回

    //遍历readyNodes
    for (Node node : nodes) {
            int size = 0;
            //找出该Node的所有partitions
            List<PartitionInfo> parts = cluster.partitionsForNode(node.id());
            List<RecordBatch> ready = new ArrayList<>();
            /* to make starvation less likely this loop doesn't start at 0 */
            int start = drainIndex = drainIndex % parts.size();
            do {
                PartitionInfo part = parts.get(drainIndex);
                TopicPartition tp = new TopicPartition(part.topic(), part.partition());
                // Only proceed if the partition has no in-flight batches.
                if (!muted.contains(tp)) {
                    //找出partition对应的deque
                    Deque<RecordBatch> deque = getDeque(new TopicPartition(part.topic(), part.partition()));
                    if (deque != null) {
                        synchronized (deque) {
                        //取出第一个recordBach
                            RecordBatch first = deque.peekFirst();
                            if (first != null) {
                                boolean backoff = first.attempts > 0 && first.lastAttemptMs + retryBackoffMs > now;
                                // Only drain the batch if it is not during backoff period.
                                if (!backoff) {
                                // size 超过限制 且针对该Node已经有partition要发送，则跳过该batch的发送
                                    if (size + first.records.sizeInBytes() > maxSize && !ready.isEmpty()) {
                                        // there is a rare case that a single batch size is larger than the request size due
                                        // to compression; in this case we will still eventually send this batch in a single
                                        // request
                                        break;
                                    } else {
                                    //更新drainedMs
                                        RecordBatch batch = deque.pollFirst();
                                        batch.records.close();
                                        size += batch.records.sizeInBytes();
                                        ready.add(batch);
                                        batch.drainedMs = now;
                                    }
                                }
                            }
                        }
                    }
                }
                this.drainIndex = (this.drainIndex + 1) % parts.size();
            } while (start != drainIndex);
            batches.put(node.id(), ready);


#### 找出expiredRecordBatch并清理

过程 : 遍历batches，针对每一个partition的recordBatch，判断该batch是否过期，如果过期，则清理。

    for (Map.Entry<TopicPartition, Deque<RecordBatch>> entry : this.batches.entrySet()) {
        Deque<RecordBatch> dq = entry.getValue();
        TopicPartition tp = entry.getKey();
        if (!muted.contains(tp)) {
                synchronized (dq) {
                    // iterate over the batches and expire them if they have been in the accumulator for more than requestTimeOut
                    RecordBatch lastBatch = dq.peekLast();
                    Iterator<RecordBatch> batchIterator = dq.iterator();
                    while (batchIterator.hasNext()) {
                        RecordBatch batch = batchIterator.next();
                        boolean isFull = batch != lastBatch || batch.records.isFull();
                        // check if the batch is expired
                        if (batch.maybeExpire(requestTimeout, retryBackoffMs, now, this.lingerMs, isFull)) {
                            expiredBatches.add(batch);
                            count++;
                            batchIterator.remove();
                            deallocate(batch);
                        } else {
                            // Stop at the first batch that has not expired.
                            break;
                        }
                    }
                }
            }
        }
        
        过期判断方法如下：
            /*lastAppendTime ：  创建时生成，append消息时更新*/
            /*createdMs : 创建batch时生成*/
            /*lastAttemptMs : 创建batch时生成，批次发送失败后,如果可以重试，则会重新设置该值*/
        
            public boolean maybeExpire(int requestTimeoutMs, long retryBackoffMs, long now, long lingerMs, boolean isFull) {
            boolean expire = false;
            //首次发送，batch 已经处于full状态，now > requestTimeoutMs +  lastAppendTime
            if (!this.inRetry() && isFull && requestTimeoutMs < (now - this.lastAppendTime))
                expire = true;
            //首次发送，now >  this.createdMs + requestTimeoutMs + lingerMs
            else if (!this.inRetry() && requestTimeoutMs < (now - (this.createdMs + lingerMs)))
                expire = true;
            // 处于retry状态，now > this.lastAttemptMs + retryBackoffMs + requestTimeoutMs
            else if (this.inRetry() && requestTimeoutMs < (now - (this.lastAttemptMs + retryBackoffMs)))
                expire = true;
            if (expire) {
                this.records.close();
                this.done(-1L, Record.NO_TIMESTAMP, new TimeoutException("Batch containing " + recordCount + " record(s) expired due to timeout while requesting metadata from brokers for " + topicPartition));
            }
            return expire;
        }
        


#### 创建和发送produce请求

    //为每个节点创建一个produceRequest请求
    private List<ClientRequest> createProduceRequests(Map<Integer, List<RecordBatch>> collated, long now) {
        List<ClientRequest> requests = new ArrayList<ClientRequest>(collated.size());
        for (Map.Entry<Integer, List<RecordBatch>> entry : collated.entrySet())
            requests.add(produceRequest(now, entry.getKey(), acks, requestTimeout, entry.getValue()));
        return requests;
    }
    //遍历requests并发出
    for (ClientRequest request : requests)
            client.send(request, now);

#### 更新metadata（cluster）并处理response以及连接问题


    /*lastSuccessfulRefreshMs : 初始值为0，每次更新成功后，刷新该值*/
    /*lastRefreshMs : 初始值为0，每次更新成功或失败后，都会刷新该值*/
    /*refreshBackoffMs : 初始化metadata时生成，由参数metadata.max.age.ms控制*/
    public synchronized long timeToNextUpdate(long nowMs) {
        long timeToExpire = needUpdate ? 0 : Math.max(this.lastSuccessfulRefreshMs + this.metadataExpireMs - nowMs, 0);
        long timeToAllowUpdate = this.lastRefreshMs + this.refreshBackoffMs - nowMs;
        return Math.max(timeToExpire, timeToAllowUpdate);
    }
    
    /*lastNoNodeAvailableMs : 上次调用metadataupdate方法但没有可用node节点的时间*/
    /*metadataFetchInProgress : 默认false，在调用metadataupdate时设置为true，调用完毕设置为false*/
    /*refreshBackoffMs : 初始化metadata时生成，由参数retry.backoff.ms控制*/
    public long maybeUpdate(long now) {
        // should we update our metadata?
        long timeToNextMetadataUpdate = metadata.timeToNextUpdate(now);
        long timeToNextReconnectAttempt = Math.max(this.lastNoNodeAvailableMs + metadata.refreshBackoff() - now, 0);
        long waitForMetadataFetch = this.metadataFetchInProgress ? Integer.MAX_VALUE : 0;
        // if there is no node available to connect, back off refreshing metadata
        long metadataTimeout = Math.max(Math.max(timeToNextMetadataUpdate, timeToNextReconnectAttempt),
                waitForMetadataFetch);

            if (metadataTimeout == 0) {
                // Beware that the behavior of this method and the computation of timeouts for poll() are
                // highly dependent on the behavior of leastLoadedNode.
                Node node = leastLoadedNode(now);
                maybeUpdate(now, node);
            }

            return metadataTimeout;
        }
    
        // Do actual reads and writes to sockect
        public List<ClientResponse> poll(long timeout, long now) {
            // 判断是否需要更新metadata
            long metadataTimeout = metadataUpdater.maybeUpdate(now);
            try {
                this.selector.poll(Utils.min(timeout, metadataTimeout, requestTimeoutMs));
            } catch (IOException e) {
                log.error("Unexpected error during I/O", e);
            }

        // process completed actions
        long updatedNow = this.time.milliseconds();
        List<ClientResponse> responses = new ArrayList<>();
        //处理各种send， response，断掉的连接，新建连接，超时请求
        handleCompletedSends(responses, updatedNow);
        handleCompletedReceives(responses, updatedNow);
        handleDisconnections(responses, updatedNow);
        handleConnections();
        handleTimedOutRequests(responses, updatedNow);

        // invoke callbacks 
        for (ClientResponse response : responses) {
            if (response.request().hasCallback()) {
                try {
                    response.request().callback().onComplete(response);
                } catch (Exception e) {
                    log.error("Uncaught error in request completion:", e);
                }
            }
        }

        return responses;
    }


常用的参数：

RecordAccumulator ：消息“累加器”，kafka消息发送并非每条消息发送一个请求，而是会将消息放入“累加器”中，以recordBatch的方式发送，可通过一些参数控制相关逻辑，可配置参数如下：


参数| 功能
---|---
batch.size | 一条消息占用内存的最小值
buffer.memory | 总共可用内存空间
linger.ms | 判断batch expire使用，给判断 batch expire加上一个固定浮动时间
compression.type | 压缩类型
retry.backoff.ms | batch retry的时间间隔 




NetWorkClient ： 是一个对客户端与kafka集群连接的封装，管理IO连接，屏蔽了消息发送接收细节，可以通过传递一些参数来控制交互细节。


参数 | 功能
---|---
connections.max.idle.ms  | 一个连接可以处于idle状态的最长时间
max.in.flight.requests.per.connection | 对于单节点每次可发送的最大batch数，为1时，可以保证消息发送的顺序性，设置为大于1的值，则可能导致消息发送不是完全顺序
reconnect.backoff.ms | 连接断掉重新创建的间隔
send.buffer.bytes | 每次发送消息buffer（SO_SNDBUF）的大小
receive.buffer.bytes | 每次接收消息buffer（SO_RCVBUF）的大小
request.timeout.ms | 发送请求后等待response的时间


Sender : Sender启动独立线程完成消息发送，可通过部分参数控制相关逻辑，有如下参数可配置


参数 | 功能
---|---
max.request.size  | 每次课发送消息的最大size
acks | 发送消息“可靠性”模式控制
retries | 消息发送失败后可重试的最大次数

相关的代码逻辑课参考

    org.apache.kafka.clients.producer.KafkaProducer
    org.apache.kafka.clients.producer.internals.RecordAccumulator
    org.apache.kafka.clients.producer.internals.RecordBatch
    org.apache.kafka.clients.producer.internals.Sender
    org.apache.kafka.clients.NetworkClient

相关的参数详细说明，可参考：

    org.apache.kafka.clients.producer.ProducerConfig

