# 简介

canal 是阿里巴巴的一个开源项目，基于java实现，整体已经在很多大型的互联网项目生产环境中使用，包括阿里、美团等都有广泛的应用，是一个非常成熟的数据库同步方案，基础的使用只需要进行简单的配置即可。canal是通过模拟成为mysql 的slave的方式，监听mysql 的binlog日志来获取数据，binlog设置为row模式以后，不仅能获取到执行的每一个增删改的脚本，同时还能获取到修改前和修改后的数据，基于这个特性，canal就能高性能的获取到mysql数据数据的变更。

[github地址](https://github.com/itmifen/canal)

以下实现一个使用cannal监控Linux数据库，远程LInux的数据库数据发生变化，本地的windows相应的数据库也发生相应的变化

# Linux中Cannal环境配置

1. 查看MySQL是否开启log_bin

```sql
mysql> show variables like 'log_bin';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_bin       | OFF   |
+---------------+-------+
1 row in set (0.00 sec)

```



2. 如果log_bin关闭，修改MySQL配置开启log_bin

```sql
1，修改 mysql 的配置文件 my.cnf
vi /etc/my.cnf 
追加内容：
log-bin=mysql-bin     #binlog文件名
binlog_format=ROW     #选择row模式
server_id=1           #mysql实例id,不能和canal的slaveId重复

2，重启 mysql：
service mysql restart	

3，登录 mysql 客户端，查看 log_bin 变量
mysql> show variables like 'log_bin';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_bin       | ON|
+---------------+-------+
1 row in set (0.00 sec)
```



<img src="https://gitee.com/little_broken_child_9527/images/raw/master/20200602102438.png" alt="image-20200602102429743" style="zoom: 80%;" />

> 注意：如果自己使用的MySQL账户没有开启远程登录功能（例如root账户默认没有远程登录功能）是无法做到数据同步的，如果能够使用MySQL客户端工具账号密码远程连接Linux上的MySQL就说明该账户有远程登录功能，反之需要给创建一个能够远程登录MySQL的账号
>
> 创建一个具有远程登录功能的账号
>
> CREATE USER 'canal'@'%' IDENTIFIED BY 'canal';
> GRANT SHOW VIEW, SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
> FLUSH PRIVILEGES;  //刷新权限
>
> 授予账号远程登录权限
>
> GRANT ALL PRIVILEGES ON \*.\* TO 'root'@'%'WITH GRANT OPTION                                                     FLUSH PRIVILEGES;  //刷新权限

3. [下载cannel](https://github.com/alibaba/canal/releases)，将cannell安装的Linux，修改cannal的配置文件，cannal解压后的目录结构如图所示
4. ![image-20200602104750241](https://gitee.com/little_broken_child_9527/images/raw/master/20200602104752.png)

```sql
1. 解压文件
tar -zxvf canal.deployer-1.1.4.tar.gz -C  cannal-1.1.4

2.解压后进入cannal的配置文件，修改配置文件
vi conf/example/instance.properties
#需要改成自己的数据库信息
canal.instance.master.address=192.168.159.160:3306

#需要改成自己的数据库用户名与密码
canal.instance.dbUsername=root
canal.instance.dbPassword=1234

#需要改成同步的数据库表规则，例如只是同步一下表
canal.instance.filter.regex=.*\\..* #所有的表都同步
#canal.instance.filter.regex=guli_ucenter.ucenter_member#只是同步指定的数据表

3.启动cannal和关闭cannal
cd ./bin
./startup.sh
./stop.sh

```

> ## 注：
>
> mysql 数据解析关注的表，Perl正则表达式。多个正则之间以逗号(,)分隔，转义符需要双斜杠(\\) 
> 常见例子：
>
> 1.  所有表：.*   or  .*\\..*
> 2.  canal schema下所有表： canal\\..*
> 3.  canal下的以canal打头的表：canal\\.canal.*
> 4.  canal schema下的一张表：canal.test1
> 5.  多个规则组合使用：canal\\..*,mysql.test1,mysql.test2 (逗号分隔)，注意：此过滤条件只针对row模式的数据有效(ps. mixed/statement因为不解析sql，所以无法准确提取tableName进行过滤)

**最后一定要关闭Linux的防火墙`systemctl stop firewalld`或者开启Cannal的连接端口号默认11111**



# 编写代码Cannal数据同步

在本地的windows中编写代码连接Linux的Cannal，Cannal监控Linux中的数据库，如果数据库发生变化，则本地的windows会通过cannal获取到数据的变化，编写java代码同步相关的数据到windows的数据库在windows和linux的MySQL中建立相同名称的数据库以及数据表

+ 引入相关依赖，这里使用的db-utils作为数据库同步工具

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!--mysql-->
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
    </dependency>

    <dependency>
        <groupId>commons-dbutils</groupId>
        <artifactId>commons-dbutils</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-jdbc</artifactId>
    </dependency>

    <dependency>
        <groupId>com.alibaba.otter</groupId>
        <artifactId>canal.client</artifactId>
    </dependency>
</dependencies>
```

+ 配置yml

```yml
server:
  port: 9999

spring:
  application:
    name: cannal-client
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    username: root
    password: 1234
    url: jdbc:mysql://localhost:3306/guli?serverTimezone=GMT%2B8
```

+ 编写cannal的配置

```java
package com.whpu.cannal.confing;

import com.alibaba.otter.canal.client.CanalConnector;
import com.alibaba.otter.canal.client.CanalConnectors;
import com.alibaba.otter.canal.protocol.CanalEntry.*;
import com.alibaba.otter.canal.protocol.Message;
import com.google.protobuf.InvalidProtocolBufferException;
import org.apache.commons.dbutils.DbUtils;
import org.apache.commons.dbutils.QueryRunner;
import org.springframework.stereotype.Component;

import javax.annotation.Resource;
import javax.sql.DataSource;
import java.net.InetSocketAddress;
import java.sql.Connection;
import java.sql.SQLException;
import java.util.List;
import java.util.Queue;
import java.util.concurrent.ConcurrentLinkedQueue;

@Component
public class CanalClient {

  //sql队列
  private Queue<String> SQL_QUEUE = new ConcurrentLinkedQueue<>();

  @Resource
  private DataSource dataSource;

  /**
   * canal入库方法
   */
  public void run() {

    //ip需要修改成Linux的ip，端口号固定为11111
    CanalConnector connector = CanalConnectors.newSingleConnector(new InetSocketAddress("192.168.159.160",
      11111), "example", "", "");
    int batchSize = 1000;
    try {
      connector.connect();
      connector.subscribe(".*\\..*");
      connector.rollback();
      try {
        while (true) {
          //尝试从master那边拉去数据batchSize条记录，有多少取多少
          Message message = connector.getWithoutAck(batchSize);
          long batchId = message.getId();
          int size = message.getEntries().size();
          if (batchId == -1 || size == 0) {
            Thread.sleep(1000);
          } else {
            dataHandle(message.getEntries());
          }
          connector.ack(batchId);

          //当队列里面堆积的sql大于一定数值的时候就模拟执行
          if (SQL_QUEUE.size() >= 1) {
            executeQueueSql();
          }
        }
      } catch (InterruptedException e) {
        e.printStackTrace();
      } catch (InvalidProtocolBufferException e) {
        e.printStackTrace();
      }
    } finally {
      connector.disconnect();
    }
  }

  /**
   * 模拟执行队列里面的sql语句
   */
  public void executeQueueSql() {
    int size = SQL_QUEUE.size();
    for (int i = 0; i < size; i++) {
      String sql = SQL_QUEUE.poll();
      System.out.println("[sql]----> " + sql);

      this.execute(sql.toString());
    }
  }

  /**
   * 数据处理
   *
   * @param entrys
   */
  private void dataHandle(List<Entry> entrys) throws InvalidProtocolBufferException {
    for (Entry entry : entrys) {
      if (EntryType.ROWDATA == entry.getEntryType()) {
        RowChange rowChange = RowChange.parseFrom(entry.getStoreValue());
        EventType eventType = rowChange.getEventType();
        if (eventType == EventType.DELETE) {
          saveDeleteSql(entry);
        } else if (eventType == EventType.UPDATE) {
          saveUpdateSql(entry);
        } else if (eventType == EventType.INSERT) {
          saveInsertSql(entry);
        }
      }
    }
  }

  /**
   * 保存更新语句
   *
   * @param entry
   */
  private void saveUpdateSql(Entry entry) {
    try {
      RowChange rowChange = RowChange.parseFrom(entry.getStoreValue());
      List<RowData> rowDatasList = rowChange.getRowDatasList();
      for (RowData rowData : rowDatasList) {
        List<Column> newColumnList = rowData.getAfterColumnsList();
        StringBuffer sql = new StringBuffer("update " + entry.getHeader().getTableName() + " set ");
        for (int i = 0; i < newColumnList.size(); i++) {
          sql.append(" " + newColumnList.get(i).getName()
            + " = '" + newColumnList.get(i).getValue() + "'");
          if (i != newColumnList.size() - 1) {
            sql.append(",");
          }
        }
        sql.append(" where ");
        List<Column> oldColumnList = rowData.getBeforeColumnsList();
        for (Column column : oldColumnList) {
          if (column.getIsKey()) {
            //暂时只支持单一主键
            sql.append(column.getName() + "=" + column.getValue());
            break;
          }
        }
        SQL_QUEUE.add(sql.toString());
      }
    } catch (InvalidProtocolBufferException e) {
      e.printStackTrace();
    }
  }

  /**
   * 保存删除语句
   *
   * @param entry
   */
  private void saveDeleteSql(Entry entry) {
    try {
      RowChange rowChange = RowChange.parseFrom(entry.getStoreValue());
      List<RowData> rowDatasList = rowChange.getRowDatasList();
      for (RowData rowData : rowDatasList) {
        List<Column> columnList = rowData.getBeforeColumnsList();
        StringBuffer sql = new StringBuffer("delete from " + entry.getHeader().getTableName() + " where ");
        for (Column column : columnList) {
          if (column.getIsKey()) {
            //暂时只支持单一主键
            sql.append(column.getName() + "=" + column.getValue());
            break;
          }
        }
        SQL_QUEUE.add(sql.toString());
      }
    } catch (InvalidProtocolBufferException e) {
      e.printStackTrace();
    }
  }

  /**
   * 保存插入语句
   *
   * @param entry
   */
  private void saveInsertSql(Entry entry) {
    try {
      RowChange rowChange = RowChange.parseFrom(entry.getStoreValue());
      List<RowData> rowDatasList = rowChange.getRowDatasList();
      for (RowData rowData : rowDatasList) {
        List<Column> columnList = rowData.getAfterColumnsList();
        StringBuffer sql = new StringBuffer("insert into " + entry.getHeader().getTableName() + " (");
        for (int i = 0; i < columnList.size(); i++) {
          sql.append(columnList.get(i).getName());
          if (i != columnList.size() - 1) {
            sql.append(",");
          }
        }
        sql.append(") VALUES (");
        for (int i = 0; i < columnList.size(); i++) {
          sql.append("'" + columnList.get(i).getValue() + "'");
          if (i != columnList.size() - 1) {
            sql.append(",");
          }
        }
        sql.append(")");
        SQL_QUEUE.add(sql.toString());
      }
    } catch (InvalidProtocolBufferException e) {
      e.printStackTrace();
    }
  }

  /**
   * 入库
   * @param sql
   */
  public void execute(String sql) {
    Connection con = null;
    try {
      if(null == sql) return;
      con = dataSource.getConnection();
      QueryRunner qr = new QueryRunner();
      int row = qr.execute(con, sql);
      System.out.println("update: "+ row);
    } catch (SQLException e) {
      e.printStackTrace();
    } finally {
      DbUtils.closeQuietly(con);
    }
  }
}

```

+ 修改cannal的启动类

```java
@SpringBootApplication
public class CanalApplication implements CommandLineRunner {
  @Resource
  private CanalClient canalClient;

  public static void main(String[] args) {
    SpringApplication.run(CanalApplication.class, args);
  }

  @Override
  public void run(String... strings) throws Exception {
    //项目启动，执行canal客户端监听
    canalClient.run();
  }
}
```











































