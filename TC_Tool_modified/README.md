官方给的工具是将解析的数据存到elasticsearch的，但是数据集的解压增长率非常恐怖，对空间要求很高。因此针对这个问题，我对工具主要进行了两个修改：
 - 利用logstash的插件直接将json输出到本地文件，删掉了grafana
 - 参考engagement3的数据格式重写logstash过滤器,对字段进行了删减和修改，剔除不必要字段。

开源不易，记得star一下，感激不尽！
## 1. 文件树介绍
![image-20230807112843855](https://raw.githubusercontent.com/LDragon-yu/blog-img/main/202308071128893.png)

| 文件                                                   | 内容                                                         |
| ------------------------------------------------------ | ------------------------------------------------------------ |
| theia                                                  | 存放原始数据的文件夹                                         |
| elasticsearch                                          | 数据库，已经不需要了，但是logstash以来这个数据库，所以还是保留了 |
| logs                                                   | 存放json文件的地方                                           |
| logstash                                               | 日志收集器，负责收集解压出来的log4j日志，然后输出到本地文件  |
| docker-compose.yml                                     | 镜像的配置文件                                               |
| TCCDMDatum.avsc                                        | 一个模式文件，用于规范化数据格式，负责从log到json的转换      |
| tc-das-importer-1.0-SNAPSHOT-jar-with-dependencies.jar | 官方的java包，用于解压、读取并参考上述数据规范生成标准格式的数据通过socket发送 |
## 2. 可修改配置
### 2.1 elastic search的内存限制（非必要）
在`docker-compose.yml`中存在对于elasticsearch的内存限额，如果1G对于你的机器存在负担，可以尝试改为512、256等。

![image-20230807112938157](https://raw.githubusercontent.com/LDragon-yu/blog-img/main/202308071129194.png#pic_center)

### 2.2 初始日志输出地址
我们可以通过命令`java -Dlog4j.debug=true -cp .:tc-das-importer-1.0-SNAPSHOT-jar-with-dependencies.jar main.java.com.bbn.tc.DASImporter [原属数据路径] [模式文件路径] [输出IP] [输出端口] -v`启动对于原始日志的解压和解析，启动前确保已有JAVA环境且logstash已成功启动。如果你采用C/S模式，这里的IP和端口可以修改为需要的地址。
```shell
java -Dlog4j.debug=true -cp .:tc-das-importer-1.0-SNAPSHOT-jar-with-dependencies.jar main.java.com.bbn.tc.DASImporter ./theia/ ./TCCDMDatum.avsc 127.0.0.1 4712 -v
```
### 2.3 初始日志接收地址
logstash负责接收Java包发送来的日志进行处理和输出到本地文件，可修改的的东西主要为4个：

 - docker-compose.yml中挂载的本地路径。

![image-20230807113010247](https://raw.githubusercontent.com/LDragon-yu/blog-img/main/202308071130284.png)

 - logstash/pipline/logstash.conf中的监听端口。如果有修改发送地址，此处也应该修改为对应的端口

 - ![image-20230807113035922](https://raw.githubusercontent.com/LDragon-yu/blog-img/main/202308071130951.png)
 - logstash/pipline/logstash.conf中的过滤器。如果有额外需求，可以通过修改过滤器对字段进行调整

```cpp
filter {
    json {
        source => "message"
    }
    mutate {
    //移除不必要字段
       remove_field=>["message","timestamp","file","@version","path","thread","host","method","priority","logger_name","class"]
    }
    //转换时间格式
    mutate {
        convert => {
            "[datum][com.bbn.tc.schema.avro.cdm20.Event][timestampNanos]" => "string"
        }
    }
    mutate {
        gsub => ["[datum][com.bbn.tc.schema.avro.cdm20.Event][timestampNanos]", "\d{6}$", ""]
    }
    date {
        match => ["[datum][com.bbn.tc.schema.avro.cdm20.Event][timestampNanos]", "UNIX_MS"]
        timezone => "America/New_York"
        locale => "en"  
        target => "@timestamp"
    }
}
```
 - logstash/pipline/logstash.conf中的输出文件的命名规则。为了避免单个文件过大，这里采用以小时为单位的时间格式命名。注释掉的输出方式为控制台输出，可以打开用以观察是否正常接收到数据，正式转换时再注释掉。

![image-20230807113109132](https://raw.githubusercontent.com/LDragon-yu/blog-img/main/202308071131163.png)