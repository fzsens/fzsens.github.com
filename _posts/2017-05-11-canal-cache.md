---
layout: post
title: Canal 构建数据缓存
date: 2017-05-11
categories: cache
tags: [canal]
description: 基于alibaba Canal构建数据缓存服务
---

### Canal
Canal Server模拟MySQL的Slave服务器，dump MySQL的binlog并解析成为事件，通过Canal Client读取，用于各种操作。可能的应用场景有：

1. 与共享缓存或者搜索引擎的集成
2. 事件驱动业务
3. 数据库同步
4. 分布式索引

使用Canal的优点，在于可以最大限度解耦业务和数据的关系。一般而言，业务操作生成数据，数据库存储数据并提供数据访问入口在MySQL中就是典型的SQL。Canal巧妙利用了数据库的[binlog机制](https://dev.mysql.com/doc/refman/5.5/en/mysqlbinlog.html)，订阅解析binlog，并在内部封装成一个个事件。利用状态机的原理，数据消费方只要通过订阅Canal事件，即可获取数据库binlog中的事件流，通过对事件流的解析，即可获取整个数据变化的记录。

更多关于Canal的介绍可以参考
[Canal Wiki](https://github.com/alibaba/canal/wiki)
[Canal](https://www.iteye.com/topic/1129002)
[Otter](https://www.iteye.com/topic/1131759)

### 构建Redis缓存

对于系统中的一些热点数据，比如MDM等，经常会被其他的系统访问，通过缓存可以有效减少数据库压力提高响应速度。一般基于数据库的缓存逻辑为

1. 应用程序从缓存中取值，如果值不存在，则从数据库中取值，并放入缓存中
2. 应用程序更新数据库，更新成功之后，将对应的缓存设置为失效

![cache](/assets/img/canal/cache.png)

这一种模式在缓存应用中大量使用，除了在少数情况下会出现数据库数据和缓存数据不一致的情况，大多数情况下都是实现缓存的best practice。

这种模式也有一个变种，将更新事件封装为消息，直接投递到消息队列中，以提供给不同的消费方使用

![cache-mq](/assets/img/canal/cache-mq.png)

引入MQ之后，将待缓存数据直接写入到MQ队列中，缓存服务器可以进行订阅消费，同时其他有需要的应用也可以通过订阅的消息的形式及时获取更新数据，这种构建模式，相比直接吸入Cache Server的模式，增加了更多的可能性。

但是这两种模式，都是将数据库操作和缓存操作结合在一起，数据的生产程序负责数据库的数据维护，同时数据库的消费程序也需要负责数据库的维护。这样的耦合在一些情况下显得比较多余，例如在数据同步和消息订阅的时候。

### 基于Canal构建Redis缓存

如果研究Kafka以及MySQL的binlog，会发现其中有很多共同的地方，本质上都是作为日志，记录数据变更的过程。因此我们可以考虑是不是第二种构建缓存的模式，可以使用binlog来作为数据的来源。再结合Canal的功能和定位，我们可以将整体缓存架构设计为

![canal-cache](/assets/img/canal/canal-cache.png)

对比这种模式和第二种模式，会发现两者之间有很多类似的地方，只是MQ队列的位置被binlog和Canal取代。

在这种模式下，对缓存的更新操作完全和业务系统隔离，而是通过binlog借助Canal来实现，数据的生产者只需要将数据写入到数据库中，而数据的消费者只需要从缓存中获取数据。同时Canal也支持多个客户端的消费，实际上Canal使用订阅和发布的模式，在很多情况下也可以结合MQ来进行数据发布。

当然这样的架构也存在一些缺点：

1. 缓存的数据范围的问题，从上图中可以看到，如果缓存服务器中的数据只是部分数据的话，当应用从缓存服务器中读取的时候，可能会出现读取不到的情况这时候还是需要从数据库中读取，如果之后这部分数据并没有更新，那么缓存可能一直会失效。
2. binlog的解析存在一定时间的延迟

因此这种架构适用于：全量数据同步、允许短时间的数据延迟，典型的应用场景有商品数据缓存和分发，搜索引擎数据构建。

### 代码实现

最后使用一个例子简单说明一下整体系统的构建，

使用到的软件主要有

1. Redis
2. Zookeeper
3. Canal

关于各个软件的运行和配置就不做说明，可以自行查看对应的文档。

数据的同步以Table作为对象，为了方便解析，构建`SchemaFilter`作为过滤器，使用`SchamaFilter`可以通过表名筛选出需要进行Cache的数据

````java
@ToString
public final class SchemaFilter {

    @Getter
    private final Map<String,List<CacheNode>> tables ;

    public SchemaFilter() {
        tables = new ConcurrentHashMap<>();
    }

    /**
     * 新增，更新策略
     * @param tableName tableName
     * @param node tableStrategy
     */
    public void add(String tableName, CacheNode node ) {

        synchronized (this) {
            List<CacheNode> tableCacheNodes = tables.get(tableName);
            if(tableCacheNodes == null){
                tableCacheNodes = new ArrayList<>();
            }
            tableCacheNodes.add(node);
            tables.put(tableName, tableCacheNodes);
        }

    }

    /**
     * 过滤
     * @param tableName 表名
     * @return 表对应的策略
     */
    public List<CacheNode> filter(String tableName) {
        return this.tables.get(tableName);
    }

}
````

缓存的对象为表中的行数据，包含主键和减值

````java
@Data
@ToString
public class CacheNode {
    /**
     * 主键
     */
    private String key;

    /**
     * 值
     */
    private List<String> values;
}
````

在配置方面，为了便于解析，使用简单的YAML作为配置语言


````yml
!!com.qiongsong.manager.filter.SchemaFilter
tables:
  xdual:
  - !!com.qiongsong.manager.filter.CacheNode
    key: ID
    values: [X]
  - !!com.qiongsong.manager.filter.CacheNode
    key: X
    values: [ID]
  md_area:
  - !!com.qiongsong.manager.filter.CacheNode
    key: ID
    values: [CODE, NAME]
````

配置使用配置的配置解析入口`FileMdmCacheConfig`

````java
@Slf4j
@Slf4j
public class FileMdmCacheConfig {

    @Setter
    private SchemaFilter filter;

    @Setter
    private String file;

    private boolean initialize = false;

    /**
     * 初始化SchemaFilter
     */
    public void init() {
        if (StringUtils.isEmpty(file)) {
            log.error("config file not exist");
        } else {
            Yaml yaml = new Yaml();
            try {
                URL url = Thread.currentThread().getContextClassLoader().getResource(file);
                if (url != null) {
                    InputStream input = (InputStream) url.getContent();
                    filter = yaml.loadAs(input, SchemaFilter.class);
                }
            } catch (IOException e) {
                log.error("load config file {} error, message {}", file, e);
            }
        }
        if (filter == null) {
            filter = new SchemaFilter();
        }
        initialize = true;
    }

    /**
     * @param tableName tableName
     * @return cache node list
     */
    public List<CacheNode> doFilter(String tableName) {
        if(!initialize) {
            log.error("please call init() to initialize config , before do filter");
            return null;
        }
        return filter.filter(tableName);
    }
}
````

从`Canal Server`中获取数据，订阅并消费

````java
@Slf4j
public class MdmCacheServer extends Thread {

    @Getter
    @Setter
    private ICache<String, String> redisCache;
    @Getter
    @Setter
    private String zkAddress;
    @Getter
    @Setter
    private String destination;
    @Getter
    @Setter
    private String configure;

    protected CanalConnector connector;

    protected FileMdmCacheConfig cacheConfig;

    protected volatile boolean running;

    /**
     *
     */
    public MdmCacheServer() {
        super("MdmCacheToRedis-Thread");
    }

    @Override
    public void run() {

        init();

        while (running) {
            // get message
            Message message = connector.getWithoutAck(5 * 1024);
            long batchId = message.getId();
            int size = message.getEntries().size();
            try {
                if (batchId == -1 || size == 0) {
                    TimeUnit.SECONDS.sleep(1L);
                } else {
                    doMessage(message.getEntries());
                }
                // commit
                connector.ack(batchId);
            } catch (InterruptedException ignored) {
                // fail rollback
                connector.rollback(batchId);
            }
        }
    }

    /**
     * @param entrys CanalEntry
     */
    protected void doMessage(List<CanalEntry.Entry> entrys) {
        for (CanalEntry.Entry entry : entrys) {

            if (entry.getEntryType() == CanalEntry.EntryType.ROWDATA) {
                CanalEntry.RowChange rowChange;
                try {
                    rowChange = CanalEntry.RowChange.parseFrom(entry.getStoreValue());
                } catch (Exception e) {
                    throw new RuntimeException("parse event has an error , data:" + entry.toString(), e);
                }
                CanalEntry.EventType eventType = rowChange.getEventType();
                for (CanalEntry.RowData rowData : rowChange.getRowDatasList()) {
                    if (eventType == CanalEntry.EventType.DELETE) {
                        deleteCache(entry, rowData.getBeforeColumnsList());
                    } else if (eventType == CanalEntry.EventType.INSERT) {
                        updateCache(entry, rowData.getAfterColumnsList());
                    } else if (eventType == CanalEntry.EventType.UPDATE) {
                        updateCache(entry, rowData.getAfterColumnsList());
                    } else {
                        log.debug("eventType {} do nothing", eventType);
                    }
                }
            }
        }
    }

    /**
     * 更新
     *
     * @param entry   canal entry
     * @param columns canal entry columns
     */
    protected void updateCache(CanalEntry.Entry entry, List<CanalEntry.Column> columns) {

        List<CacheNode> tbCacheNodes = cacheConfig.doFilter(entry.getHeader().getTableName());

        if (tbCacheNodes != null) {
            for (CacheNode strategy : tbCacheNodes) {
                String keyValue = "";
                String valValue;
                StringBuilder valueBuilder = new StringBuilder("{");
                for (CanalEntry.Column column : columns) {
                    if (column.getName().equals(strategy.getKey())) {
                        keyValue = column.getValue();
                    } else if (strategy.getValues().contains(column.getName())) {
                        valueBuilder.append(column.getName()).append(":").append(column.getValue()).append(",");
                    }
                }
                if (valueBuilder.lastIndexOf(",") != -1) {
                    valValue = valueBuilder.substring(0, valueBuilder.lastIndexOf(","));
                } else {
                    valValue = valueBuilder.toString();
                }
                valValue = valValue + "}";
                if (StringUtils.isNotEmpty(keyValue) && StringUtils.isNotEmpty(valValue)) {
                    log.debug("table {},  k [{}] : v [{}]", entry.getHeader().getTableName(), keyValue, valValue);
                    redisCache.put(keyValue, valValue);
                }
            }
        }
    }

    /**
     * 删除
     *
     * @param entry   CanalEntry
     * @param columns Canal Columns
     */
    protected void deleteCache(CanalEntry.Entry entry, List<CanalEntry.Column> columns) {
        List<CacheNode> tbCacheNodes = cacheConfig.doFilter(entry.getHeader().getTableName());
        if (tbCacheNodes != null) {
            for (CacheNode strategy : tbCacheNodes) {
                String keyValue = "";
                for (CanalEntry.Column column : columns) {
                    if (column.getName().equals(strategy.getKey())) {
                        keyValue = column.getValue();
                    }
                }
                if (StringUtils.isNotEmpty(keyValue)) {
                    log.debug("table {} delete key {}", entry.getHeader().getTableName(), keyValue);
                    redisCache.remove(keyValue);
                }
            }
        }
    }


    /**
     * 初始化
     */
    protected void init() {
        connector =
                CanalConnectors.newClusterConnector(zkAddress, destination, "", "");
        connector.connect();
        connector.subscribe();

        cacheConfig = new FileMdmCacheManager();
        cacheConfig.setFile(configure);
        cacheConfig.init();
        while (!connector.checkValid()) {
            //blocking until connector connect success.
        }
        running = true;
    }

    /**
     * 关闭
     */
    public void shutdown() {
        running = false;
        connector.disconnect();
        log.info("MdmCacheToRedis-Thread SHUTDOWN!!!");
    }

}
````

从Canal中订阅并获取Message之后，解析为程序可以识别的事件，便可以灵活进行数据的消费。