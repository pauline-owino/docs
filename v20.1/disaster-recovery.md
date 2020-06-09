---
title: Disaster Recovery
summary: Learn how about CockrachDB disaster recovery capabilities and what to do if you encounter an issue.
toc: true
---

While CockroachDB is built to be fault-tolerant and to recover automatically, sometimes disasters happen. A disaster is any event that puts your cluster at risk, and usually means you are experiencing hardware failure or data corruption. Having a disaster recovery plan enables you to recover quickly, while limiting the consequences.

## Hardware failure

When planning to survive through hardware failures, these are the minimum replication factors to apply in order to have greater resilience.  Do note that increasing data replication factors may have an impact on transaction performance since quorum levels are increasing.

### Single-region survivability planning

The table below shows the minimum replication factor to have in order to survive simultaneous hardware failures (disk, node, availability zone (AZ), etc).

<table>
  <thead>
    <tr>
      <th style="border-left: hidden, border-bottom: hidden"></th>
      <th colspan="3" style="text-align: center"># of Nodes in Cluster</th>
    </tr>
    <tr>
      <th>Simultaneous Failure</th>
      <th>3 nodes <br>(3 AZ/Racks)</th>
      <th>4 nodes <br>(4 AZ/Racks)</th>
      <th>5 nodes <br>(5 AZ/Racks)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="color:#46a417"><b>Disk</b></td>
      <td>Rep Factor = 3</td>
      <td>Rep Factor = 3</td>
      <td>Rep Factor = 3</td>
    </tr>
      <td style="color:#46a417"><b>Node</b></td>
      <td>Rep Factor = 3</td>
      <td>Rep Factor = 3</td>
      <td>Rep Factor = 3</td>
    <tr>
      <td style="color:#46a417"><b>AZ/Rack</b></td>
      <td>Rep Factor = 3</td>
      <td>Rep Factor = 3</td>
      <td>Rep Factor = 3</td>
    </tr>
    <tr>
      <td style="color:#46a417"><b>2 Nodes</b></td>
      <td>Rep Factor = 5</td>
      <td>Rep Factor = 5</td>
      <td>Rep Factor = 5</td>
    </tr>
    <tr>
      <td style="color:#46a417"><b>AZ/Rack + Node</b></td>
      <td>Rep Factor = 9</td>
      <td>Rep Factor = 7</td>
      <td>Rep Factor = 5</td>
    </tr>
    <tr>
      <td style="color:#46a417"><b>2 AZ/Racks</b></td>
      <td>Not possible</td>
      <td>Not possible</td>
      <td>Rep Factor = 5</td>
    </tr>
  </tbody>
</table>

### Single-region recovery

For hardware failures, depending on the type of infrastructure being used (e.g., virtual machines, Kubernetes, bare metal), the actions taken could vary. The actions listed in the following table make some of those distinctions for CockroachDB in a single region (i.e., 5 nodes, each node in a separate availability zone or rack, with a replication factor of 5):

<table>
  <thead>
    <tr>
      <th>Simultaneous Failure</th>
      <th>Availability</th>
      <th>Consequence</th>
      <th>Action to Take</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="color:#46a417"><b>1 Disk</td>
      <td style="color:#228B22"><b>√</b></td>
      <td rowspan="7">Less resources are available. Some data will be under-replicated until the failed node is marked dead. <br><br>Once marked dead, data is replicated to other nodes and the cluster remains healthy.
      </td>
      <td>Restart Cockroachdb on the node with a new disk</td>
    </tr>
      <td style="color:#46a417"><b>1 Node</td>
      <td style="color:#228B22"><b>√</b></td>
      <td rowspan="6">If using a Kubenetes StatefulSet, a node will be restarted automatically. <br><br>If the Server, Rack or AZ becomes available, check the Admin UI on the Overview page:
      <br>- If the down server is marked ‘Suspect’, try restarting the node.
      <br>- If the down server is marked ‘Dead’, decommission the node and add a new server.  If you try to rejoin the same decommissioned node back into the server, you should wipe the store path before rejoining.</td>
    <tr>
      <td style="color:#46a417"><b>1 Rack</b></td>
      <td style="color:#228B22"><b>√</b></td>
    </tr>
    <tr>
      <td style="color:#46a417"><b>1 AZ</b></td>
      <td style="color:#228B22"><b>√</b></td>
    </tr>
    <tr>
      <td style="color:#46a417"><b>2 Nodes</b></td>
      <td style="color:#228B22"><b>√</b></td>
    </tr>
    <tr>
      <td style="color:#46a417"><b>1 AZ + 1 Node</b></td>
      <td style="color:#228B22"><b>√</b></td>
    </tr>
    <tr>
      <td style="color:#46a417"><b>2 AZ</b></td>
      <td style="color:#228B22"><b>√</b></td>
    </tr>
    <tr>
      <td style="color:#46a417"><b>3 Nodes</b></td>
      <td style="color:#FF0000"><b>X</b></td>
      <td>Cluster will become unavailable.</td>
      <td>Recover one of the 3 nodes that are down to regain quorum. <br><br>If you can’t recover one of the 3 failed nodes, contact Cockroach Labs support so we can assist in your cluster’s recovery.</td>
    </tr>
    <tr>
      <td style="color:#46a417"><b>1 Region</td>
      <td style="color:#FF0000"><b>X</b></td>
      <td>Cluster will become unavailable. <br>Potential data loss between last backup and time of outage if region and servers did not come back online. <br><br>This would be a considerable disaster (i.e. an entire data center was destroyed).</td>
      <td>When the region comes back online, try restarting the nodes in the cluster. <br><br>If region does not come back online and servers are lost or destroyed, try restoring the latest cluster backup into a new cluster.</td>
    </tr>
  </tbody>
</table>

<span style="color:#228B22"><b>√</b></span> = Available
<span style="color:#FF0000"><b>X</b></span> = Outage

### Multi-region survivability planning

For multi-region, the chart below describes default behavior without using geo-partitioning or other advanced topologies.  If locality flags are correctly set, data will be balanced across regions and support this survivability matrix.

<table>
  <thead>
    <tr>
      <th style="border-left: hidden, border-bottom: hidden"></th>
      <th colspan="3" style="text-align: center"># of Nodes in Cluster</th>
    </tr>
    <tr>
      <th>Simultaneous Failure</th>
      <th>3 Regions <br>(3 AZ/Racks) <br>(9 Nodes)</th>
      <th>4 Regions <br>(4 AZ/Racks) <br>(12 Nodes)</th>
      <th>5 Regions <br>(5 AZ/Racks) <br>(15 Nodes)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="color:#46a417"><b>Disk</b></td>
      <td>Rep Factor = 3</td>
      <td>Rep Factor = 3</td>
      <td>Rep Factor = 3</td>
    </tr>
      <td style="color:#46a417"><b>Node</b></td>
      <td>Rep Factor = 3</td>
      <td>Rep Factor = 3</td>
      <td>Rep Factor = 3</td>
    <tr>
      <td style="color:#46a417"><b>AZ/Rack</b></td>
      <td>Rep Factor = 3</td>
      <td>Rep Factor = 3</td>
      <td>Rep Factor = 3</td>
    </tr>
    <tr>
      <td style="color:#46a417"><b>Region</b></td>
      <td>Rep Factor = 3</td>
      <td>Rep Factor = 3</td>
      <td>Rep Factor = 3</td>
    </tr>
    <tr>
      <td style="color:#46a417"><b>Region + 1 Node</b></td>
      <td>Rep Factor = 7</td>
      <td>Rep Factor = 7</td>
      <td>Rep Factor = 5</td>
    </tr>
    <tr>
      <td style="color:#46a417"><b>2 Nodes</b></td>
      <td>Rep Factor = 5</td>
      <td>Rep Factor = 5</td>
      <td>Rep Factor = 5</td>
    </tr>
    <tr>
      <td style="color:#46a417"><b>2 Regions</b></td>
      <td>Not possible</td>
      <td>Not possible</td>
      <td>Rep Factor = 5</td>
    </tr>
    <tr>
      <td style="color:#46a417"><b>2 Regions +1 Node</b></td>
      <td>Not possible</td>
      <td>Not possible</td>
      <td>Rep Factor = 11</td>
    </tr>
  </tbody>
</table>

### Multi-region recovery

For hardware failures, depending on the type of infrastructure being used (virtual machines, kubernetes, bare metal), the actions taken could vary a bit. The actions listed in the following tale make some of those distinctions for CockroachDB in multiple regions (i.e., 9 nodes, each node in a separate availability zone, 3 nodes in each region, replication factor of 3):

<table>
  <thead>
    <tr>
      <th>Simultaneous Failure</th>
      <th>Availability</th>
      <th>Consequence</th>
      <th>Action to Take</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="color:#46a417"><b>1 Disk</td>
      <td style="color:#228B22"><b>√</b></td>
      <td rowspan="5">Under-replicated data. Less resources for workload.</td>
      <td>Restart Cockroachdb on the node with a new disk</td>
    </tr>
      <td style="color:#46a417"><b>1 Server</td>
      <td style="color:#228B22"><b>√</b></td>
      <td>If using a Kubenetes StatefulSet, a node will be restarted automatically. <br><br>If the Server, Rack or AZ becomes available, check the Admin UI on the Overview page. If the down server is marked ‘Suspect’, try restarting the node.
</td>
    <tr>
      <td style="color:#46a417"><b>1 Rack</b></td>
      <td style="color:#228B22"><b>√</b></td>
      <td rowspan="2">If the down server is marked ‘Dead’, decommission the node and add a new server.  If you try to rejoin the same decommissioned node back into the server, you should wipe the store path before rejoining.</td>
    </tr>
    <tr>
      <td style="color:#46a417"><b>1 AZ</b></td>
      <td style="color:#228B22"><b>√</b></td>
      <td></td>
    </tr>
    <tr>
      <td style="color:#46a417"><b>1 Region</b></td>
      <td style="color:#228B22"><b>√</b></td>
      <td>If servers are marked ‘Dead’, decommission the nodes and add 3 new servers in a new region.</td>
    </tr>
    <tr>
      <td style="color:#46a417"><b>2 or More Regions</b></td>
      <td style="color:#FF0000"><b>X</b></td>
      <td>Cluster will be unavailable. <br><br>Potential data loss between last backup and time of outage if region and servers did not come back online.  This would be a considerable disaster (i.e. 2 or more data centers destroyed).</td>
      <td>When the regions comes back online, try restarting the nodes in the cluster. <br><br>If the regions do not come back online and servers are lost or destroyed, try restoring the latest cluster backup into a new cluster.</td>
    </tr>
  </tbody>
</table>

<span style="color:#228B22"><b>√</b></span> = Available
<span style="color:#FF0000"><b>X</b></span> = Outage

## Data failure

At the end of the day, bad actors, rogue applications or data corruption require domain expertise to get exactly right in the long term. There are a few options for short term remediation that you can take.

Basic:
Restore to a point in time from your backup file for the corrupted data to where you are certain there was no corruption.
Recommend renaming the table/database so that you have historical data to compare to rather than dropping table/database
Run as of system time queries / `create table as … select * from` to create comparison data and run “diffs” to see if you can find the offending rows to adjust

Advanced (Probably don’t mention to start? These can be in addition to the above):
Put your application in “read only” mode, query as of system only from the application, while you clean things up

### Bad or corrupted data in table

Restore the table from a prior backup or a point in time if revision history is in the backup.  If the table has foreign keys, careful consideration should be applied to make sure data integrity is maintained during the restore process.

- Add more about GC windows on AOST queries (PIT)

### How to recover from corrupted data in a database

Restore the table from a prior backup or a point in time if revision history is in the backup.

### How to recover from compromised security keys

As a best practice, keys should be rotated on an occasional basis to ensure an extra layer of security.

CockroachDB does a great job at maintaining a secure environment for your data.  However, there are bad actors who may find ways to gain access or expose important security information. In the event, a bad actor mucks around in your environment, there are a few things you can do to get ahead of a security issue:

#### Changefeeds to cloud storage

1. [Cancel the changefeed job](cancel-job.html) immediately and [record the high water mark](change-data-capture.html#monitor-a-changefeed) for where the changefeed was stopped .  
2. Remove the access keys from the identify management system of your cloud provider and replace with a new set of access keys.  
3. [Create a new changefeed](create-changefeed.html#start-a-new-changefeed-where-another-ended) with the new access credentials using the last high water mark.

#### Encryption at rest

If you believe the user defined store keys have been compromised, quickly attempt to rotate your store keys that are being used for your encryption at rest setup.  If this key has already been compromised and the store keys were rotated by a bad actor, the cluster should be wiped if possible and restored from a prior backup.

If the compromised key were not rotated by a bad actor, quickly attempt to [rotate the store key](encryption.html#rotating-keys) by restarting each of the nodes with the old key and the new key. For an example on how to do this, see [Encryption](encryption.html#changing-encryption-algorithm-or-keys).

Once all of the nodes are restarted with the new key, put in a request to revoke the old key from the Certificate Authority.  CockroachDB takes an extra step here and does not allow prior store keys to be used again.

#### Wire Encryption / TLS

As a best practice, [keys should be rotated](rotate-certificates.html). In the event that keys have been compromised, quickly attempt to rotate your keys.  This can include rotating node, client and the CA certificate.
