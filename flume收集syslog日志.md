# 简单介绍

收集系统日志或者交换机/路由器日志到kafka，供spark分析

# syslog配置

这里采用的rsyslog,系统默认的,也可以采用logstash或者syslog-ng

* 日志格式

日志格式以$template开头

```
$template ServerLog, "%FROMHOST% %PROGRAMNAME% %TIMESTAMP:::date-rfc3339% %syslogtag%%msg:::sp-if-no-1st-sp%%msg:::drop-last-lf%\n"
```

然后将日志转发到flume的侦听端口。有两种方式：转发所有日志和转发指定日志

* 转发所有日志

什么都不用改，在/etc/rsyslog.conf最后加入一行就行

```
*.* @zjdw-pre0064:5140;ServerLog
```

* 转发指定文件日志

```
$ModLoad imfile
$InputFileName /var/log/secure
$InputFileTag secure_log
$InputFileStateFile secure_log
$InputFileSeverity info
$InputFileFacility local6
$InputRunFileMonitor
```

然后:

```
local6.* @zjdw-pre0064:5140;myFormat
```


完整的配置如下：

```
$ModLoad imuxsock # provides support for local system logging (e.g. via logger command)
$ModLoad imklog   # provides kernel logging support (previously done by rklogd)
$ModLoad imfile
$InputFileName /var/log/secure
$InputFileTag secure_log
$InputFileStateFile secure_log
$InputFileSeverity info
$InputFileFacility local6
$InputRunFileMonitor
$ActionFileDefaultTemplate RSYSLOG_TraditionalFileFormat
$template myFormat,"%TIMESTAMP:::date-rfc3339% %HOSTNAME% %syslogtag%%msg:::sp-if-no-1st-sp%%msg:::drop-last-lf%\n"
$template ForwardFormat,"<%PRI%>%TIMESTAMP:::date-rfc3339% %HOSTNAME% %syslogtag:1:32%%msg:::sp-if-no-1st-sp%%msg%"
$template ServerLog, "%FROMHOST% %PROGRAMNAME% %TIMESTAMP:::date-rfc3339% %syslogtag%%msg:::sp-if-no-1st-sp%%msg:::drop-last-lf%\n"
$template ServerLog, "%FROMHOST% %FROMHOST-IP% %PROGRAMNAME% %TIMESTAMP:::date-rfc3339% %syslogtag%%msg:::sp-if-no-1st-sp%%msg:::drop-last-lf%\n"
$IncludeConfig /etc/rsyslog.d/*.conf
*.info;mail.none;authpriv.none;cron.none                /var/log/messages
authpriv.*                                              /var/log/secure
mail.*                                                  -/var/log/maillog
cron.*                                                  /var/log/cron
*.emerg                                                 *
uucp,news.crit                                          /var/log/spooler
local7.*                                                /var/log/boot.log
*.* @zjdw-pre0064:5140;ServerLog
local6.* @zjdw-pre0064:5140;myFormat
```

详细配置可以参考：http://www.rsyslog.com/doc/master/index.html 注意选择对应的版本

# flume端配置

这个就不说了，直接上配置

需要注意的加上a1.sources.r1.keepFields = true 这个，要不然都不知道哪台主机过来的日志

```
[root@zjdw-pre0064 apache-flume-1.7.0-bin]# cat conf/syslog.conf
# Name the components on this agent
a1.sources = r1
a1.sinks = k1
a1.channels = c1

# Describe/configure the source
a1.sources.r1.type = syslogudp
a1.sources.r1.port = 5140
a1.sources.r1.keepFields = true 
a1.sources.r1.host = zjdw-pre0064

# Describe the sink
a1.sinks.k1.type = org.apache.flume.sink.kafka.KafkaSink
a1.sinks.k1.topic = syslog
a1.sinks.k1.brokerList = zjdw-pre0065:9092,zjdw-pre0066:9092,zjdw-pre0067:9092,zjdw-pre0068:9092,zjdw-pre0069:9092
a1.sinks.k1.requiredAcks = 1
a1.sinks.k1.batchSize = 10000

# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 100000000
a1.channels.c1.transactionCapacity = 10000
a1.channels.c1.byteCapacityBufferPercentage = 20
a1.channels.c1.byteCapacity = 800000000
a1.channels.c1.keep-alive = 60

# Bind the source and sink to the channel
a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1

```





