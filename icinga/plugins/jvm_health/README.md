# JVM GC Performance Tuning and Alerts

The garbage collector's young object memory space is reported so that memory leaks can be detected in developing applications.  To learn more about performance tuning and garbage collection in Java 1.6 then check out the following articles.

* http://www.petefreitag.com/articles/gctuning/
* http://java.sun.com/performance/reference/whitepapers/6_performance.html

---
## How it works

### Prerequisites

Be sure to place `jvm_health.py` and `parsegarbagelogs.py` in `/usr/local/sbin/`.

Must enable garbage collection logs in the JVM options.

    JAVA_OPTS="$JAVA_OPTS -XX:+PrintGCDetails -Xloggc:/path/to/log/garbage.log"

Create an sqlite database which will be used by `parsegarbagelogs.py`.


    sqlite3 /var/db/sqldb
    CREATE TABLE `datapoints` (`timestamp` varchar(40),`young_space` varchar(20), `old_space` varchar(20), `perm` varchar(20), `young_gc` varchar(25),`young_gc_collection_time` varchar(25), `full_gc` varchar(25), `full_gc_collection_time` varchar(25),`total_gc_time` varchar(25));
    .quit

There should be a cron job which runs parsegarbagelogs.py every 15 minutes.

    0,15,30,45 * * * * /usr/local/sbin/parsegarbagelogs.py -f /path/to/log/garbage.log -s /var/db/sqldb > /dev/null 2>&1


`parsegarbagelogs.py` parses the garbage collector logs and calculates the percentage of memory of the young object memory space against the total space. It takes that calculated value and stores it in an sqlite database located at /var/db/sqldb. parsegarbagelogs.py should have owner `root:nsca` with `755` permissions if you're using passive checks in Icinga. The cron job for `parsegarbagelogs.py` is run by root and `crontab -l` shows the cron job listing.

### Monitoring JVM-HEALTH

`jvm_health.py` is a Icinga plugin which reads the calculated percentage from the sqlite database and reports a status to Icinga. If the young object memory percentage is less than 50% then it is good. If greater than 50% then warning. If greater than 90% then critical and a crash is imminent. Here is the `cmds` variable in the [report-status](https://github.com/sag47/drexel-university/blob/master/icinga/scripts/report-status.py) script for [passive Icinga checks](http://docs.icinga.org/latest/en/passivechecks.html).

    cmds = {
      "LOAD": "check_load -w 5,5,5 -c 10,10,10",
      "DISK": "check_disk -w 5% -c 1%",
      "PROCS-SENDMAIL": "check_procs -u root -C sendmail -w 1: -c 1:",
      "PROCS-NTPD": "check_procs -u ntp -C ntpd -w 1: -c 1:",
      "JVM-HEALTH": "/usr/local/sbin/jvm_health.py"
    }



# This is a set of munin plugins which I wrote or coauthored.

---
## java\_vm\_time

This script uses [parsegarbagelogs.py](https://github.com/sag47/drexel-university/blob/master/icinga/plugins/jvm_health/).  To set this plugin up you must first get `parsegarbagelogs.py` working.  From there you must use symlinks to execute the different plugin types for monitoring Java with munin.

    #e.g. let's say we place it at /usr/share/munin/plugins/java_vm_time
    source="/usr/share/munin/plugins/java_vm_time"
    ln -s $source /etc/munin/plugins/java_graph
    ln -s $source /etc/munin/plugins/java_vm_threads
    ln -s $source /etc/munin/plugins/java_vm_time
    ln -s $source /etc/munin/plugins/java_vm_uptime
