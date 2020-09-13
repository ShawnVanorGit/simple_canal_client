##### 什么是Canal

1. canal是阿里巴巴旗下的一款开源项目，纯Java开发。**基于数据库增量日志解析，提供增量数据订阅&消费**，目前主要支持了Mysql（也支持mariaDB）。

2. 先了解下**Mysql主备复制**的工作原理：

> 1.  master将改变记录到二进制日志(binary log)中（这些记录叫做二进制日志事件，binary log events，可以通过show binlog events进行查看）；
>     
> 2.  slave将master的binary log events拷贝到它的中继日志(relay log)；
>     
> 3.  slave重做中继日志中的事件，将改变反映它自己的数据。
>     

3. **Canal的工作原理**

> 1.  canal模拟mysql slave的交互协议，伪装自己为mysql slave，向mysql master发送dump协议
>     
> 2.  mysql master收到dump请求，开始推送binary log给slave(也就是canal)
>     
> 3.  canal解析binary log对象(原始为byte流)
>     

4. **Canal架构设计**

*   server代表一个canal运行实例，对应于一个jvm
    
*   instance对应于一个数据队列 （1个server对应1…n个instance)

5. **Canal的instance模块**

*   eventParser (数据源接入，模拟slave协议和master进行交互，协议解析)
    
*   eventSink (Parser和Store链接器，进行数据过滤，加工，分发的工作)
    
*   eventStore (数据存储)
    
*   metaManager (增量订阅&消费信息管理器)


##### mysql开启binlog步骤

1. 登录Mysql后使用`show variables like 'log_%';`查看是否开启binlog

2. 编辑配置文件`vim /etc/my.cnf`

```
server_id=1 #配置 MySQL replaction 需要定义，不要和 canal 的 slaveId 重复
log-bin=mysql-bin #开启 binlog
binlog-format=ROW #选择 ROW模式
expire_logs_days=30
```

3. 重启Mysql服务 `systemctl restart mysqld`，然后再次使用命令`show variables like 'log_%';`进行查看，为 `ON`表明binlog已成功开启

4. 授权 canal 链接 MySQL 账号具有作为 MySQL slave 的权限

```
CREATE USER canal IDENTIFIED BY 'canal';  
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
FLUSH PRIVILEGES;
```

其中，` /usr/local/mysql/data` 为binlog日志文件存放路径，mysql-bin是日志文件系统的前缀名

##### 安装Canal到CentOS7

```
#下载安装包
wget https://github.com/alibaba/canal/releases/download/canal-1.0.17/canal.deployer-1.0.17.tar.gz
#解压
tar zxvf canal.deployer-$version.tar.gz  -C /usr/local/canal
```

修改配置`vi conf/example/instance.properties`

启动canal `sh bin/startup.sh`

查看server日志 `vi logs/canal/canal.log`

查看 instance 的日志`vim logs/example/example.log`

##### 连接本地Java客户端测试canal是否成功解析binlog

```
#配置pom文件
<dependency>  
    <groupId>com.alibaba.otter</groupId>  
    <artifactId>canal.client</artifactId>  
    <version>1.1.1</version>  
</dependency>
```

```
import java.net.InetSocketAddress;  
import java.util.List;  
  
import com.alibaba.otter.canal.client.CanalConnectors;  
import com.alibaba.otter.canal.client.CanalConnector;  
import com.alibaba.otter.canal.protocol.Message;  
import com.alibaba.otter.canal.protocol.CanalEntry.Column;  
import com.alibaba.otter.canal.protocol.CanalEntry.Entry;  
import com.alibaba.otter.canal.protocol.CanalEntry.EntryType;  
import com.alibaba.otter.canal.protocol.CanalEntry.EventType;  
import com.alibaba.otter.canal.protocol.CanalEntry.RowChange;  
import com.alibaba.otter.canal.protocol.CanalEntry.RowData;  
  
public class SimpleCanalClientExample {  
    public static void main(String args[]) {  
        // 创建链接  
 CanalConnector connector = CanalConnectors.newSingleConnector(new InetSocketAddress("120.27.233.226",  
                11111), "example", "", "");  
        int batchSize = 1000;  
        int emptyCount = 0;  
        try {  
            connector.connect();  
            connector.subscribe(".*..*");  
            connector.rollback();  
            int totalEmptyCount = 120;  
            while (emptyCount < totalEmptyCount) {  
                Message message = connector.getWithoutAck(batchSize); // 获取指定数量的数据  
 long batchId = message.getId();  
                int size = message.getEntries().size();  
                if (batchId == -1 || size == 0) {  
                    emptyCount++;  
                    System.out.println("empty count : " + emptyCount);  
                    try {  
                        Thread.sleep(1000);  
                    } catch (InterruptedException e) {  
                    }  
                } else {  
                    emptyCount = 0;  
                    System.out.printf("message[batchId=%s,size=%s] n", batchId, size);  
                    printEntry(message.getEntries());  
                }  
  
                connector.ack(batchId); // 提交确认  
 // connector.rollback(batchId); // 处理失败, 回滚数据  
 }  
  
            System.out.println("empty too many times, exit");  
        } finally {  
            connector.disconnect();  
        }  
    }  
  
    private static void printEntry(List<Entry> entrys) {  
        for (Entry entry : entrys) {  
            if (entry.getEntryType() == EntryType.TRANSACTIONBEGIN || entry.getEntryType() == EntryType.TRANSACTIONEND) {  
                continue;  
            }  
  
            RowChange rowChage = null;  
            try {  
                rowChage = RowChange.parseFrom(entry.getStoreValue());  
            } catch (Exception e) {  
                throw new RuntimeException("ERROR ## parser of eromanga-event has an error , data:" + entry.toString(),  
                        e);  
            }  
  
            EventType eventType = rowChage.getEventType();  
            System.out.println(String.format("binlog[%s:%s]: 库名是[%s],表名是 [%s],eventType是[%s]",  
                    entry.getHeader().getLogfileName(), entry.getHeader().getLogfileOffset(),  
                    entry.getHeader().getSchemaName(), entry.getHeader().getTableName(),  
                    eventType));  
  
            for (RowData rowData : rowChage.getRowDatasList()) {  
                if (eventType == EventType.DELETE) {  
                    printColumn(rowData.getBeforeColumnsList());  
                } else if (eventType == EventType.INSERT) {  
                    printColumn(rowData.getAfterColumnsList());  
                } else {  
                    System.out.println("------- before -------");  
                    printColumn(rowData.getBeforeColumnsList());  
                    System.out.println("------- after -------");  
                    printColumn(rowData.getAfterColumnsList());  
                }  
            }  
        }  
    }  
  
    private static void printColumn(List<Column> columns) {  
        for (Column column : columns) {  
            System.out.println("字段名" + column.getName() + "值为" + column.getValue() + " ,更新状态是" + column.getUpdated());  
        }  
    }  
  
}
```

触发数据库变更，可以从控制台看到

具体图片网页请移步：[SimpleCanalClientSample](https://segmentfault.com/a/1190000024440565)
