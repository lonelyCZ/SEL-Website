+++

id= "83"

title = "Cloud Foundry中syslog_aggregator的实现分析"
description = "在Cloud Foundry中，用来收集Cloud Foundry各组件日志信息的组件，名为syslog_aggregator。 syslog_aggregator可以做到方便的收集Cloud Foundry中所有组件的日志信息，并将这些信息进行初步处理，比如说：将不同月份产生的日志，进行分类存储；另外还对同一月份内产生的日志，将其通过不同的日期进行分类。这样的话，当Cloud Foundry平台的开发者，在运营该平台时需要查看Cloud Foundry中某一个组件产生的日志时，可以方便的查找到对应日期的日志。syslog_aggregator除了可以对日志进行分组件，分月份，分日期进行存储外，还提供一些对日志进行打包或剪枝的功能，比如：syslog_aggregator会将一定期限内的日志，进行压缩，以达到节省存储空间的功能；另外syslog_aggregator还会定期对日志进行清除，比如只保存一定期限时间长度的日志，当日志超过该时限，syslog_aggregator会将其清除。 "
tags= [ "syslog_aggregator" , "cloudfoundry" ]
date= "2014-05-07 10:19:32"
author = "丁轶群"
banner= "img/blogs/83/cloud-foundry.jpg"
categories = [ "cloudfoundry" ]

+++

在Cloud Foundry中，用来收集Cloud Foundry各组件日志信息的组件，名为syslog\_aggregator。 

syslog\_aggregator可以做到方便的收集Cloud Foundry中所有组件的日志信息，并将这些信息进行初步处理，比如说：将不同月份产生的日志，进行分类存储；另外还对同一月份内产生的日志，将其通过不同的日期进行分类。这样的话，当Cloud Foundry平台的开发者，在运营该平台时需要查看Cloud Foundry中某一个组件产生的日志时，可以方便的查找到对应日期的日志。syslog\_aggregator除了可以对日志进行分组件，分月份，分日期进行存储外，还提供一些对日志进行打包或剪枝的功能，比如：syslog\_aggregator会将一定期限内的日志，进行压缩，以达到节省存储空间的功能；另外syslog\_aggregator还会定期对日志进行清除，比如只保存一定期限时间长度的日志，当日志超过该时限，syslog\_aggregator会将其清除。 

<!--more-->

以下是对syslog\_aggregator实现的简单分析： syslog\_aggregator组件主要包括monit模块，日志管理模块。


**monit模块** 
----------

monit模块主要是实现：监控syslog\_aggregator组件的运行状态，一旦监控过syslog\_aggregator组件中该进程不存活时，即刻重启该进程；另外，syslog\_aggregator组件还将自身的信息通过cloud\_agent传送给NATS，这里的信息包括syslog\_aggregator组件所在的宿主机的存活状态以及资源使用情况。 以下通过monit监控进程的代码：

```ruby
check process syslog_aggregator  
  with pidfile /var/vcap/sys/run/syslog_aggregator/syslog_aggregator.pid  
  start program "/var/vcap/jobs/syslog_aggregator/bin/syslog_aggregator_ctl start"  
  stop program "/var/vcap/jobs/syslog_aggregator/bin/syslog_aggregator_ctl stop"  
  group vcap  
```


该段代码中清晰的标明了进程的pid，进程的start命令以及stop命令。 cloud\_agent作为BOSH监控Cloud Foundry组件级信息的辅助工具，负责收集syslog\_aggregator组件所在宿主机的运行状态以及资源使用情况，并发送给health\_monitor，由health\_monitor统一管理。由于cloud\_agent不是本文的重点，所以本文不再赘述。 

**日志管理模块** 
----------

实现日志管理，syslog\_aggregator是通过启动syslog\_aggregator\_ctl脚本来实现的。上文中提到的monit模块中，也正是监控这个脚本命令启动的进程。以下来分析一下该脚本的代码实现： 

```shell
#!/bin/bash  

RUN_DIR=/var/vcap/sys/run/syslog_aggregator  
LOG_DIR=/var/vcap/store/log  
JOB_DIR=/var/vcap/jobs/syslog_aggregator  
PACKAGE_DIR=/var/vcap/packages/syslog_aggregator  

BIN_DIR=$JOB_DIR/bin  
PIDFILE=$RUN_DIR/syslog_aggregator.pid  

source /var/vcap/packages/common/utils.sh  

case $1 in  

  start)  

    apt-get -y install rsyslog-relp  

    pid_guard $PIDFILE "Syslog aggregator"  

    mkdir -p $RUN_DIR  
    mkdir -p $LOG_DIR  
    chown -R vcap:vcap $LOG_DIR  

    rm -f /etc/cron.daily/bzip_old_logs  
    cp -a $BIN_DIR/gzip_old_logs /etc/cron.daily  
    cp -a $BIN_DIR/reap_old_logs /etc/cron.hourly  
    cp -a $BIN_DIR/symlink_logs /etc/cron.hourly  
    cp -a $PACKAGE_DIR/send_error_mail /etc/cron.daily  

    exec /usr/sbin/rsyslogd -f $JOB_DIR/config/rsyslogd.conf -i $PIDFILE -c 4 \  
         >>$LOG_DIR/rsyslogd.stdout.log \  
         2>>$LOG_DIR/rsyslogd.stderr.log  
    ;;  

  stop)  
    kill_and_wait $PIDFILE  
    rm /etc/cron.daily/gzip_old_logs  
    rm /etc/cron.hourly/reap_old_logs  
    ;;  

  *)  
    echo "Usage: syslog_aggregator_ctl {start|stop}"  
    ;;  

esac  
```


在通过该脚本来实现启动syslog\_aggregator进程的时候，使用的是start命令。进入start命令，可以看到，安装了rsyslog-relp；然后通过/var/vcap/packages/common/utils.sh中定义的pid\_guard()方法来实现对该进程pid的保护，当系统中已经由相应的进程以该pid在运行时，删除该进程，以保证syslog\_aggregator可以按预先设置的pid进行运行；随后创建几个定义好的目录，RUN\_DIR，LOG\_DIR，还对LOG\_DIR进行拥有用户修改。 脚本中随后的5行代码，涉及到的是Linux操作系统中cron 定期任务删除与添加的实现：

```shell
rm -f /etc/cron.daily/bzip_old_logs  
cp -a $BIN_DIR/gzip_old_logs /etc/cron.daily  
cp -a $BIN_DIR/reap_old_logs /etc/cron.hourly  
cp -a $BIN_DIR/symlink_logs /etc/cron.hourly  
cp -a $PACKAGE_DIR/send_error_mail /etc/cron.daily  
```


首先在每日的执行任务中删除掉bzip\_old\_logs任务，如果该任务存在的话；随后将4个任务分别加入到了指定的目录位置，分别是：gzip\_old\_logs, reap\_old\_logs, symlink\_logs, send\_error\_mail。也就是让Linux操作系统每天一次执行gzip\_old\_logs脚本，每小时执行一次reap\_old\_logs脚本，每小时执行一次symlink\_logs脚本，每周一次执行一次send\_error\_mail脚本。 添加完这些定义任务之后，syslog\_aggregator随后启动了rsyslog server，实现日志服务器的启动：

```shell
exec /usr/sbin/rsyslogd -f $JOB_DIR/config/rsyslogd.conf -i $PIDFILE -c 4 \  
         >>$LOG_DIR/rsyslogd.stdout.log \  
         2>>$LOG_DIR/rsyslogd.stderr.log  
```


启动rsyslog server的具体配置可以参看rsyslogd.conf的各参数: 

```shell
$MaxMessageSize 4k  

# Caveat - This always binds to all interfaces (cannot specify otherwise).  
$ModLoad imtcp  
$InputTCPMaxSessions 1024  
$InputTCPServerRun 54321  
$PrivDropToUser vcap  
$PrivDropToGroup vcap  

# Write each component's messages to a separate log  
# programname is assumed to be 'vcap.<component>'  
# Directory is created automatically  
$template VcapComponentLogFile, "/var/vcap/store/log/%programname:6:$%/%$year%/%$month%/%$day%/%programname:6:$%.log"  
$template VcapComponentLogFormat, "%fromhost-ip% -- %msg%\n"  

# The initial '-' tells rsyslogd to not sync the file after each write  
:programname, startswith, "vcap." -?VcapComponentLogFile;VcapComponentLogFormat  

# Prevent them from reaching anywhere else  
:programname, startswith, "vcap." ~  
```


当然有server的话，自然会accept来自client的请求，所以在Cloud Foundry中每个组件都会安装一个resyslog的client端，然后启动该client，连接rsyslog server，并发送日志请求，以此来实现日志的传输，又通过刚才涉及到的那些脚本实现对日志的管理。 以下分析添加到周期性任务中的脚本功能。 1.gzip\_old\_logs 该脚本的实现很简单，如下：

```shell
#!/bin/bash  
# Compress log files that haven't changed in the last day  
find /var/vcap/store/log -type f -mmin +1440 -name '*.log' -exec gzip '{}' \;  
```


功能为找到/var/vcap/store/log目录下，1440分钟（24小时）内没有被修改的文件，然后进行压缩操作。 2.symlink\_logs 该脚本实现的是：为每一个当天的创建出来的文件创建符号链接，`date + %Y`执行结果为执行时的年份，依此类推。代码如下：

```shell
#!/bin/bash  
# create symlinks to today's logs in /var/vcap/store/log  
for x in /var/vcap/store/log/*/`date +%Y`/`date +%m`/`date +%d`/* ; do if [ -f "$x" ]; then ln -sf "$x" /var/vcap/store/log; fi; done  
```


3.reap\_old\_logs 该脚本实现的是：清除保存已超过7天的日志。代码如下：

```shell
#!/bin/bash  

set -u  

LOG_DIR=/var/vcap/store/log  
# Reap logs that are older than 7 days  
DAYS_TO_KEEP=7  
let "MIN_TO_KEEP=${DAYS_TO_KEEP}*24*60"  

# get the last $DAYS_TO_KEEP date to exclude from the rmdir  

DAYS_TO_EXCLUDE=$(  
for i in $(seq 0 ${DAYS_TO_KEEP}); do  
date -d "now - $i days" +%Y/%m/%d  
done  
)  
# example output: "2012/09/21|2012/09/20|.....|2012/09/15|2012/09/14"  
EGREP_FORMAT_DAYS_TO_EXCLUDE=$(echo $DAYS_TO_EXCLUDE | tr ' ' '|')  

find ${LOG_DIR} -mmin +${MIN_TO_KEEP} -name '*.log.gz' -exec rm -f '{}' \;  

# Reap empty dirs in 3 passes, clear empty 'day' dirs, then 'month'  
# and lastly 'year'  
for i in '/[0-9]{4}/[0-1][0-9]/[0-9]{2}$' \  
         '/[0-9]{4}/[0-1][0-9]$' \  
         '/[0-9]{4}$'; do  
    /usr/bin/find ${LOG_DIR} -type d -empty |  
      egrep -v "${EGREP_FORMAT_DAYS_TO_EXCLUDE}" |  
      egrep ${i} |  
      xargs -r -n 200 rmdir  
done  
```


其中，EGREP\_FORMAT\_DAYS\_TO\_EXCLUDE是为了获取一个通过‘|’字符串联起来的字符串，随后实现对指定路径进行清除。 以上便是对Cloud Foundry中syslog\_aggregator的简单分析。

**转载请注明出处。**
------------
这篇文档更多出于我本人的理解，肯定在一些地方存在不足和错误。如果你对这方面感兴趣，并有更好的想法和建议，也请联系我。 