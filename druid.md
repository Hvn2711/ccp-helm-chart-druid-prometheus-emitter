# zookeeper data directory was occupied 100% in pv volume

## Symptoms

zookeeper data directory was occupied 100% in pv volume

## Detection

```
Error message in zookeeper pod:

zookeeper@zk-0:/$ echo status | nc localhost 2181



This ZooKeeper instance is not currently serving requests
zookeeper@zk-0:/$ exit
```

## logs

```
2021-09-21 06:58:22,227 [myid:1] - ERROR [QuorumPeer[myid=1]/0.0.0.0:2181:QuorumPeer@1347] - Failed to write new file /var/lib/zookeeper/data/version-2/currentEpoch
java.io.IOException: No space left on device
    at java.io.FileOutputStream.write(Native Method)
    at java.io.FileOutputStream.write(FileOutputStream.java:290)
    at java.io.FilterOutputStream.write(FilterOutputStream.java:77)
    at java.io.FilterOutputStream.write(FilterOutputStream.java:125)
    at sun.nio.cs.StreamEncoder.writeBytes(StreamEncoder.java:221)
    at sun.nio.cs.StreamEncoder.implFlushBuffer(StreamEncoder.java:291)
```

Also after exec into zk pod on doinf df -h we see higher value for zookeeper size.

```
/dev/nvme1n1    9.8G  9.6G  234M  98% /var/lib/zookeeper
```

## Customer Impact

Zookeeper and historical pods will be down whihc results to query failures

## Resolution

### Steps

1. Before deleting snapshots, please ensure that we have more than 20 snapshots in the snapshot directory. Then only, please go ahead and delete it.
   ```
   cd /opt/zookeeper/data/Version-2
   ```
   ```
   ls -trl | grep snapshot | wc -l
   ```
2. Login into each zookeeper pod and need to execute below steps.
   ```
   cd /opt/zookeeper/bin
   ```
   ```
   ./zkCleanup.sh -n 10
   ```
3. Steps to cleanup data-backup in zk pods

   ```
   kubectl exec -it zk-1 -n zk-druid bash
   ```

   ```
   cd /var/lib/zookeeper
   ```

   ```
   ls
   ```

   ```
   rm -rf data-backup
   ```

4. Then we can resize pvc without dowtime following this wiki - <a href="https://hpe.atlassian.net/wiki/spaces/CCSPLATFORM/pages/1936492091/Resizing+zookeeper+PVC+with+zero+downtime " title="Resizing zookeeper PVC with zero downtime" rel="nofollow" target="_blank">Resizing zookeeper PVC with zero downtime</a>

# Unable to load database on disk(zookeeper)

## Symptoms

Unable to load database on disk (zookeeper)

## Detection

Below logs can be seen

```
2021-09-21 17:15:55,950 [myid:2] - ERROR [main:QuorumPeer@648] - Unable to load database on disk
java.io.IOException: The accepted epoch, 20 is less than the current epoch, 443
    at org.apache.zookeeper.server.quorum.QuorumPeer.loadDataBase(QuorumPeer.java:645)
    at org.apache.zookeeper.server.quorum.QuorumPeer.start(QuorumPeer.java:591)
    at org.apache.zookeeper.server.quorum.QuorumPeerMain.runFromConfig(QuorumPeerMain.java:164)
    at org.apache.zookeeper.server.quorum.QuorumPeerMain.initializeAndRun(QuorumPeerMain.java:111)
    at org.apache.zookeeper.server.quorum.QuorumPeerMain.main(QuorumPeerMain.java:78)
2021-09-21 17:15:55,951 [myid:2] - ERROR [main:QuorumPeerMain@89] - Unexpected exception, exiting abnormally
```

## Resolution

1. After exec into zookeeper pod execute below query
   ```
   mv /var/lib/zookeeper/data/version-2   /tmp
   ```
2. Then restart the zookeeper it should copy the snapshot from one of the healthy nodes in the quorum

# Historical pods goes into Crashloopback

## Symptoms

Historical pods goes into Crashloopback

## Detection

Below logs can be seen

    ```
    There is insufficient memory for the Java Runtime Environment to continue.
    Native memory allocation (mmap) failed to map 12288 bytes for committing reserved memory.
    An error report file with more information is saved as:
    //hs_err_pid40.log
    OpenJDK 64-Bit Server VM warning: INFO: os::commit_memory(0x00007f821b2f9000, 12288, 0) failed; error='Cannot allocate memory' (errno=12)
    ```

## Resolution

1. ssh into one of the nodes of historical.

2. Add the following line to /etc/sysctl.conf:(For vm.max_map_count value please connect with CCP team)
   ```
   vm.max_map_count=<map_count>
   ```
3. Reload the config as root:
   ```
   sysctl -p
   ```
4. Check the new value:
   ```
   cat /proc/sys/vm/max_map_count
   ```

# Historical and broker pods are restarting

## Symptoms

Historical and broker pods are restarting

## Detection

We can see below error log line

```
ERROR ocurator.framework.imps.EnsembleTracker 214 processConfigData- Invalid config event received: {server.1=192.168.0.19:2888:3888:participant, version=100000000, server.0=******:2888:3888:participant, server.2=******:2888:3888:participant}rg.apache.
```

## Resolution

Apply the max jute buffer changes. For applying max jute buffer changes follow this wiki -
<a href="https://hpe.atlassian.net/wiki/spaces/CCSPLATFORM/pages/1966245459/Druid+max+jute+buffer+change+and+druid+task+cleanup+changes+SRE+handover" title="Druid max jute buffer changes SRE handover" rel="nofollow" target="_blank">Druid max jute buffer changes SRE handover</a>

```
containers:
      - command:
        - sh
        - -c
        - start-zookeeper --servers=3 --data_dir=/var/lib/zookeeper/data --data_log_dir=/var/lib/zookeeper/data/log
          --conf_dir=/opt/zookeeper/conf --client_port=2181 --election_port=3888 --server_port=2888
          --tick_time=2000 --init_limit=10 --sync_limit=5 --heap=512M --max_client_cnxns=0
          --snap_retain_count=3 --purge_interval=12 --max_session_timeout=40000 --min_session_timeout=4000
          --log_level=INFO && export JAVA_OPTS="-Djute.maxbuffer=11000000"
        env:
        - name: JAVA_OPTS
          value: -Djute.maxbuffer=11000000
```

DevOps

- [Arvind Singh](mailto:arvind.singh@hpe.com)
- [Sridhara Reddy](mailto:sridhara-reddy.kovuru@hpe.com)
