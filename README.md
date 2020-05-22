# proxysql_galera_checker usage info.

`proxysql_galera_checker` script will check Percona XtraDB Cluster desynced nodes, and temporarily deactivate them.

This script will also call `proxysql_node_monitor` script. Monitor script will check cluster node membership, and re-configure ProxySQL if cluster membership changes occur. 

eg: If any node goes out from cluster this script will mark as `OFFLINE_HARD` in proxysql database. When it comes back it will mark the node as `ONLINE`.

The galera checker script needs to be added in ProxySQL [scheduler](https://github.com/sysown/proxysql/blob/master/doc/scheduler.md) table .

Galera checker usage

```
Usage: ${path##*/} "--config-file=/etc/proxysql-admin-sample.cnf --writer-is-reader=always --write-hg=200 --read-hg=201 --writer-count=1 --priority=10.0.0.22:3306,10.0.0.23:3306,10.0.0.33:3306 --mode=singlewrite --log=/var/lib/proxysql/pxc_test_proxysql_galera_check.log --debug"

Usage in ProxySQL scheduler: INSERT  INTO scheduler (id,active,interval_ms,filename,arg1) values (50,0,3000,"/opt/tools/proxysql-admin-tool/proxysql_galera_checker","--config-file=/opt/tools/proxysql-admin-tool/proxysql-admin-sample.cnf --writer-is-reader=always --write-hg=200 --read-hg=201 --writer-count=1 --priority=--priority=10.0.0.22:3306,10.0.0.23:3306,10.0.0.33:3306 --mode=singlewrite --log=/var/lib/proxysql/pxc_test_proxysql_galera_check.log");
```
Record will looks like this:
```
id: 50
     active: 0
interval_ms: 3000
   filename: /opt/tools/proxysql-admin-tool/proxysql_galera_checker
       arg1: --config-file=/opt/tools/proxysql-admin-tool/proxysql-admin-sample.cnf --writer-is-reader=always --write-hg=200 --read-hg=201 --writer-count=1 --priority=--priority=10.0.0.22:3306,10.0.0.23:3306,10.0.0.33:3306 --mode=singlewrite --log=/var/lib/proxysql/pxc_test_proxysql_galera_check.log
       arg2: NULL
       arg3: NULL
       arg4: NULL
       arg5: NULL
    comment: 
1 row in set (0.01 sec)
```
We strongly advice to test first the configuration on a test environment and to add the instruction in production with `ACTIVE=0` and after activate it: `update scheduler set active =1 where id=10;Load scheduler to run;`

It is also Best Practice to keep the scheduler with `ACTIVE = 0` when doing `SAVE scheduler to DISK`. Such that it will NOT be activated by default. 

```
Options:
  --config-file=PATH              Specify ProxySQL-admin configuration file.
  NOTE: --config-file MUST be the first parameter and use long option format
  
  ---------------------------------------------------------------------------
  -c, --config-file=PATH              Specify ProxySQL-admin configuration file. !! Note this MUST be the first argument you pass !!
  -w, --write-hg=<NUMBER>             Specify ProxySQL write hostgroup.
  -r, --read-hg=<NUMBER>              Specify ProxySQL read hostgroup.
  -l, --log=PATH                      Specify proxysql_galera_checker log file.
  --log-text=TEXT                     This is text that will be written to the log file
                                      whenever this script is run (useful for debugging).
  --node-monitor-log=PATH             Specify proxysql_node_monitor log file.
  -n, --writer-count=<NUMBER>         Maximum number of write hostgroup_id nodes
                                      that can be marked ONLINE
                                      When 0 (default), all nodes can be marked ONLINE
  -p, --priority=<HOST_LIST>          Can accept comma delimited list of write nodes priority
  -m, --mode=[loadbal|singlewrite]    ProxySQL read/write configuration mode,
                                      currently supporting: 'loadbal' and 'singlewrite'
  --writer-is-reader=<value>          Defines if the writer node also accepts writes.
                                      Possible values are 'always', 'never', and 'ondemand'.
                                      'ondemand' means that the writer node only accepts reads
                                      if there are no other readers.
                                      (default: 'never')
  --use-slave-as-writer=<yes/no>      If this is 'yes' then slave nodes may
                                      be added to the write hostgroup if all other
                                      cluster nodes are down.
                                      (default: 'yes')
  --max-connections=<NUMBER>          Value for max_connections in the mysql_servers table.
                                      This is the maximum number of connections that
                                      ProxySQL will open to the backend servers.
                                      (default: 1000)
  --debug                             Enables additional debug logging.
  -h, --help                          Display script usage information
  -v, --version                       Print version info

Notes about the mysql_servers in ProxySQL:

- NODE STATUS   * Nodes that are in status OFFLINE_HARD will not be checked
                  nor will their status be changed
                * SHUNNED nodes are not to be used with Galera based systems,
                  they will be checked and their status will be changed
                  to either ONLINE or OFFLINE_SOFT.


When no nodes were found to be in wsrep_local_state=4 (SYNCED) for either
read or write nodes, then the script will try 5 times for each node to try
to find nodes wsrep_local_state=4 (SYNCED) or wsrep_local_state=2 (DONOR/DESYNC)
```

