# ProxySQL_checker
## Concept
ProxySQL_checker is an application that can run as standalone or invoked from ProxySQL scheduler. Its function is to manage integration between ProxySQL and Galera (from Codership), including its different implementations like PXC.
The scope of ProxySQL_checker is to maintain the ProxySQL mysql_server table, in a way that the PXC cluster managed will suffer of minimal negative effects due to possible data node: failures, service degradation and maintenance.

ProxySQL_checker is also ProxySQL cluster aware and can deal with multiple instances running on more ProxySQL cluster nodes. When in presence of a cluster the application will attempt to set a lock at cluster level such that no other node will interfere with the actions.

At the moment galera_check analyze and manage the following:

Node states:
 * pxc_main_mode
 * read_only
 * wsrep_status
 * wsrep_rejectqueries
 * wsrep_donorrejectqueries
 * wsrep_connected
 * wsrep_desinccount
 * wsrep_ready
 * wsrep_provider
 * wsrep_segment
 * Number of nodes in by segment
 * Retry loop
 * PXC cluster state:

It also makes possible to isolate the Primary Writer from READS (when read/write split is in place in ProxySQL), such that reads will be redirect only to READ HostGroup.

### Understand the HGs relation
ProxySQL_checker leverage the HostGroup concept existing in ProxySQL to identify 3 main set of HGs, the set that needs to be handled (your pair of HGs for R/W) , the set used for configuration (the ones in the 8XXX range) and a special set use to isolate the nodes when not active (in the 9XXX range).

To clarify, let assume you have your PXC cluster compose by 3 Nodes, and you have already set query rules to split traffic R/W with the following HGs:
* 10 is the ID for the HG dealing with Writes
* 11 is the ID for the HG dealing with Reads

What you need to add is 2 CONFIGURATION HGs that as ID have the ID from your HG + 8000.
In our example:
* 8010 is the ID for the configuration HG for HG 10
* 8011 is the ID for the configuration HG for HG 11

The settings used in the 8XXX HGs like weight, use of SSL etc. are used as templates when in the need to deal with the nodes in the managed HGs (10, 11). This is it, settings in 8010/11 are persistent, the ones in 10/11 are not.


## Failover
A fail-over will occur anytime a Primary writer node will need to be demoted. This can be for planned maintenance as well as a node crash. To be able to fail-over ProxySQL_checker require to have valid Node(s) in the corresponding 8XXX HostGroup (8000 + original HG id).
Given that assume we have:
```editorconfig
node4 : 192.168.4.22 Primary Writer
node5 : 192.168.4.23
node6 : 192.168.4.233
```
If node4 will fail/maintenance,  ProxySQL_checker will choose the next one with higher weight in the 8XXX corresponding HGs.
Actions will also be different if the Node is going down because a crash or maintenance. In case of the latter the Node will be set as OFFLINE_SOFT to allow existing connections to end their work. In other cases, the node will be moved to HG 9XXX which is the special HG to isolate non active nodes.

### Failback
ProxySQL_checker by default IS NOT doing failback, this is by design. Nevertheless, if the option is activated in the config file, ProxySQL_checker will honor that.
What is a failback? Assume you have only ONE writer (Primary writer) elected because its weight is the highest in the 8XXX writer HG. If this node is removed another will take its place. When the original Node will come back and failback is active, this node will be re-elected as Primary, while the one who take its place is moved to OFFLINE_SOFT.
Why failback is bad? Because with automatic failback, your resurrecting node will be immediately move to production. This is not a good practice when in real production environment, because normally is better to warmup the Buffer-Pool to reduce the access to storage layer, and then move the node as Primary writer.

## Other notes about how nodes are managed
If a node is the only one in a segment, the check will behave accordingly. IE if a node is the only one in the MAIN segment, it will not put the node in OFFLINE_SOFT when the node become donor to prevent the cluster to become unavailable for the applications. As mention is possible to declare a segment as MAIN, quite useful when managing prod and DR site.

The check can be configure to perform retry in a X interval. Where X is the time define in the ProxySQL scheduler. As such if the check is set to have 2 retry for UP and 4 for down, it will loop that number before doing anything.

This feature is very useful in some not well known cases where Galera behave weird. IE whenever a node is set to READ_ONLY=1, galera desync and resync the node. A check not taking this into account will cause a node to be set OFFLINE and back for no reason.



## Working with ProxySQL cluster
Working with ProxySQL cluster is a real challenge give A LOT of things that should be there are not.

ProxySQL do not even has idea if a ProxySQL cluster node is up or down. To be precise it knows internally but do not expose this at any level, only as very noisy log entry.
Given this the script must not only get the list of ProxySQL nodes but validate them.

In terms of checks we do:
We check if the node were we are has a lock or if can acquire one.
If not we will return nil to indicate the program must exit given either there is already another node holding the lock or this node is not in a good state to acquire a lock.
All the DB operations are done connecting locally to the ProxySQL node running the scheduler.

Find lock method review all the nodes existing in the Proxysql for an active Lock it checks only nodes that are reachable.
Checks for:
- an existing lock locally
- lock on another node
- lock time comparing it with lockclustertimeout parameter

### Related Parameters

```editorconfig
[proxysql]
clustered = true

[global]
lockfiletimeout = 60 #seconds 
lockclustertimeout = 600 # seconds
```
Last important thing. When using Proxysql in cluster mode the address indicated in the `[proxysql]` section `host` __MUST__ be the same reported in the `proxysql_servers` table or the cluster operations will fail.
The user/password as well must be the same used for the cluster, or be able to access proxysql from another node (admin/admin will not work).

ProxySQL Documentation reports:
```
TODO
 - add support for MySQL Group Replication
 - add support for Scheduler
Roadmap
This is an overview of the features related to clustering, and not a complete list. None the following is impletemented yet.
Implementation may be different than what is listed right now:
 - support for master election: the word master was intentionally chosen instead of leader
 - only master proxy is writable/configurable
 - implementation of MySQL-like replication from master to slaves, allowing to push configuration in real-time instead of pulling it
 - implementation of MySQL-like replication from master to candidate-masters
 - implementation of MySQL-like replication from candidate-masters to slaves
 - creation of a quorum with only candidate-masters: normal slaves are not part of the quorum
```
None of the above is implemented


## Example of proxySql setup 
Assuming we have 3 nodes:
- node4 : `192.168.4.22`
- node5 : `192.168.4.23`
- node6 : `192.168.4.233`

As Hostgroup:
- HG 100 for Writes
- HG 101 for Reads
We have to configure also nodes in 8XXX:
- HG 8100 for Writes 
- HG 8101 for Reads

We will need to :
```MySQL
delete from mysql_servers where hostgroup_id in (100,101,8100,8101);
INSERT INTO mysql_servers (hostname,hostgroup_id,port,weight,max_connections,comment) VALUES ('192.168.4.22',100,3306,1000,2000,'Preferred writer');
INSERT INTO mysql_servers (hostname,hostgroup_id,port,weight,max_connections,comment) VALUES ('192.168.4.23',100,3306,999,2000,'Second preferred ');
INSERT INTO mysql_servers (hostname,hostgroup_id,port,weight,max_connections,comment) VALUES ('192.168.4.233',100,3306,998,2000,'Las chance');
INSERT INTO mysql_servers (hostname,hostgroup_id,port,weight,max_connections,comment) VALUES ('192.168.4.22',101,3306,998,2000,'last reader');
INSERT INTO mysql_servers (hostname,hostgroup_id,port,weight,max_connections,comment) VALUES ('192.168.4.23',101,3306,1000,2000,'reader1');    
INSERT INTO mysql_servers (hostname,hostgroup_id,port,weight,max_connections,comment) VALUES ('192.168.4.233',101,3306,1000,2000,'reader2');        

INSERT INTO mysql_servers (hostname,hostgroup_id,port,weight,max_connections,comment) VALUES ('192.168.4.22',8100,3306,1000,2000,'Failover server preferred');
INSERT INTO mysql_servers (hostname,hostgroup_id,port,weight,max_connections,comment) VALUES ('192.168.4.23',8100,3306,999,2000,'Second preferred');    
INSERT INTO mysql_servers (hostname,hostgroup_id,port,weight,max_connections,comment) VALUES ('192.168.4.233',8100,3306,998,2000,'Thirdh and last in the list');      

INSERT INTO mysql_servers (hostname,hostgroup_id,port,weight,max_connections,comment) VALUES ('192.168.4.22',8101,3306,998,2000,'Failover server preferred');
INSERT INTO mysql_servers (hostname,hostgroup_id,port,weight,max_connections,comment) VALUES ('192.168.4.23',8101,3306,999,2000,'Second preferred');    
INSERT INTO mysql_servers (hostname,hostgroup_id,port,weight,max_connections,comment) VALUES ('192.168.4.233',8101,3306,1000,2000,'Thirdh and last in the list');      


LOAD MYSQL SERVERS TO RUNTIME; SAVE MYSQL SERVERS TO DISK;    

```
Will create entries in ProxySQL for 2 main HG (100 for write and 101 for read) It will also create 3 entries for the SPECIAL group 8100 and other 3 for HG 8101. As already mention above these groups will be used by the application to manage all the mysql_servers table maintenance operation.

In the above example, what will happen is that if you have set active_failover=1, the application will check the nodes, if node 192.168.4.22',100 is not up, the application will try to identify another node within the same segment that has the higest weight IN THE 8100 HG. In this case it will elect as new writer the node'192.168.4.23',8100.

Please note that active_failover=1, is the only deterministic method to failover, based on what YOU define. If set correctly across a ProxySQL cluster, all nodes will act the same.



## How to configure ProxySQL_checker
There are many options that can be set, and I foresee we will have to add even more. Given that I have abandoned the previous style of using command line and move to config-file definition. Yes this is a bit more expensive, given the file access but it is minimal.

First let us see what we have:
- 3 sections
    - Global 
    - pxccluster
    - proxysql
    
### Global:
```[global]
debug = true
logLevel = "debug"
logTarget = "stdout" #stdout | file
logFile = "/Users/marcotusa/work/temp/pscheduler.log"
daemonize = true
daemonInterval = 2000
performance = true
OS = "na"
```

- debug : [false] will active some additional features to debug locally as more verbose logs
- daemonize : [false] Will allow the script to run in a loop without the need to be call by ProxySQL scheduler 
- daemonInterval : Define in ms the time for looping when in daemon mode
- loglevel : [error] Define the log level to be used 
- logTarget : [stdout] Can be either a file or stdout 
- logFile : In case file for loging define the target 
- OS : for future use
-  lockfiletimeout  Time ins seconds after which the file lock is considered expired [local instance lock]
-  lockclustertimeout Time in seconds after which the cluster lock is considered expired

### ProxySQL
- port : [6032] Port used to connect 
- host : [127.0.0.1] IP address used to connect to ProxySQL __IF using cluster you must match this with the IP in the proxysql_servers table__
- user : [] User able to connect to ProxySQL
- password : [] Password 
- clustered : [false] If this is __NOT__ a single instance then we need to put a lock on the running scheduler (see [Working with ProxySQL cluster](Working-with-ProxySQL-cluster) section)
- initialized : not used (for the moment) 
- respectManualOfflineSoft : [false] When true the checker will NOT modify an `OFFLINE_SOFT` state manually set. It will also DO NOT put back any OFFLINE_SOFT given it will be impossible to discriminate the ones set by hand from the ones set by the application. Given that this feature is to be consider, __UNSAFE__ and should never be used unless you know very well what you are doing.

### Pxccluster
- activeFailover : [1] Failover method
- failBack : [false] If we should fail-back automatically or wait for manual intervention 
- checkTimeOut : [4000] This is one of the most important settings. When checking the Backend node (MySQL), it is possible that the node will not be able to answer in a consistent amount of time, due the different level of load. If this exceeds the Timeout, a warning will be print in the log, and the node will not be processed. Parsing the log it is possible to identify which is the best value for checkTimeOut to satisfy the need of speed and at the same time to give the nodes the time they need to answer.
- mainSegment : [1] This is another very important value to set, it defines which is the MAIN segment for failover
- sslClient : "client-cert.pem" In case of use of SSL for backend we need to be able to use the right credential
- sslKey : "client-key.pem" In case of use of SSL for backend we need to be able to use the right credential
- sslCa : "ca.pem" In case of use of SSL for backend we need to be able to use the right credential
- sslCertificatePath : ["/full-path/ssl_test"] Full path for the SSL certificates
- hgW : Writer HG
- hgR : Reader HG 
- bckHgW : Backup HG in the 8XXX range (hgW + 8000)
- bckHgR :  Backup HG in the 8XXX range (hgR + 8000)
- singlePrimary : [true] This is the recommended way, always use Galera in Single Primary to avoid write conflicts
- maxNumWriters : [1] If SinglePrimary is false you can define how many nodes to have as Writers at the same time
- writerIsAlsoReader : [1] Possible values 0 - 1. The default is 1, if you really want to exclude the writer from read set it to 0. When the cluster will lose its last reader, the writer will be elected as Reader, no matter what. 
- retryUp : [0] Number of retry the script should do before restoring a failed node
- retryDown : [0] Number of retry the script should do to put DOWN a failing node
- clusterId : 10 the ID for the cluster 
 
## Examples of configurations in ProxySQL
Simply pass max 2 arguments 

```MySQL 
INSERT  INTO scheduler (id,active,interval_ms,filename,arg1,arg2) values (10,0,2000,"/var/lib/proxysql/proxysql_checker","--configfile=config.toml","--configpath=<path to config>");
LOAD SCHEDULER TO RUNTIME;SAVE SCHEDULER TO DISK;
```


To Activate it:
```MySQL 
update scheduler set active=1 where id=10;
LOAD SCHEDULER TO RUNTIME;
```

## Logic Rules used in the check:
Set to offline_soft :

- any non 4 or 2 state, read only =ON donor node reject queries - 0 size of cluster > 2 of nodes in the same segments more then one writer, node is NOT read_only

Change HG for maintenance HG:

- Node/cluster in non primary wsrep_reject_queries different from NONE Donor, node reject queries =1 size of cluster

Node comes back from offline_soft when (all of them):
- Node state is 4
- wsrep_reject_queries = none
-Primary state

Node comes back from maintenance HG when (all of them):
- node state is 4
- wsrep_reject_queries = none
- Primary state
- PXC (pxc_maint_mode).

PXC_MAIN_MODE is fully supported. Any node in a state different from pxc_maint_mode=disabled will be set in OFFLINE_SOFT for all the HostGroup.

__Single Writer__
You can define IF you want to have multiple writers. Default is 1 writer only (I strongly recommend you to do not use multiple writers unless you know very well what are you doing), but you can now have multiple writers at the same time.



## Download and compile from source
This roject follows the gGO layout standards as for https://github.com/golang-standards/project-layout
Once you have GO installed and running (version 1.15.8 is recommended. Version 1.16 is not supported yet)

Clone from github: `git clone https://github.com/Tusamarco/proxysql_scheduler.git`

Install the needed library within go:
```bash
go get github.com/Tusamarco/toml
go get github.com/go-sql-driver/mysql
go get github.com/sirupsen/logrus
go get golang.org/x/text/language
go get golang.org/x/text/message

go build -o proxysql_checker .  

```
First thing to do then is to run `./proxysql_scheduler --help`
to navigate the parameters.

Then adjust the config file in `./config/config.toml` better to do a copy and modify for what u need
Then to test it OUTSIDE the ProxySQL scheduler script, in the config file `[Global]` section change `daemonize=false` to `true`. 
The script will auto-loop as if call by the scheduler. 

 
#### Details


![function flow](./docs/flow-Funtions-calls-flow.png "Function flow")

<!--
![overview](./docs/flow-overall.png "overview")
![nodes check](./docs/flow-check_nodes.png "Nodes check")
-->