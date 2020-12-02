# Mongodb points in analysis of mongodb rollback and recovering
## Mongdb HA
### CAP

The CAP theorem states that networked shared-data systems can only guarantee/strongly support two of the following three properties, it is impossible for a distributed data store to simultaneously provide more than two out of the following three guarantees.
- Consistency — Every read receives the most recent write or an error. Consistency refers to every client having the same view of the data. There are various types of consistency models. Consistency in CAP (used to prove the theorem) refers to linearizability or sequential consistency, a very strong form of consistency.
  - Linearizability consistency
    Refer to [this link] (https://zhuanlan.zhihu.com/p/42239873).
- Availability — Every request receives a (non-error) response, without the guarantee that it contains the most recent write. Every non-failing node returns a response for all read and write requests in a reasonable amount of time. The key word here is every. To be available, every node on (either side of a network partition) must be able to respond in a reasonable amount of time.
- Partition Tolerant — The system continues to operate despite an arbitrary number of messages being dropped (or delayed) by the network between nodes. The system continues to function and upholds its consistency guarantees in spite of network partitions. Network partitions are a fact of life. Distributed systems guaranteeing partition tolerance can gracefully recover from partitions once the partition heals.
When a network partition failure happens should we decide to
  - Cancel the operation and thus decrease the availability but ensure consistency
  - Proceed with the operation and thus provide availability but risk inconsistency
#### Explanation
No distributed system is safe from network failures, thus network partitioning generally has to be tolerated[5][6]. In the presence of a partition, one is then left with two options: consistency or availability. When choosing consistency over availability, the system will return an error or a time out if particular information cannot be guaranteed to be up to date due to network partitioning. When choosing availability over consistency, the system will always process the query and try to return the most recent available version of the information, even if it cannot guarantee it is up to date due to network partitioning.

In the absence of network failure – that is, when the distributed system is running normally – both availability and consistency can be satisfied.

CAP is frequently misunderstood as if one has to choose to abandon one of the three guarantees at all times. In fact, the choice is really between consistency and availability only when a network partition or failure happens; at all other times, no trade-off has to be made.

The CAP theorem implies that in the presence of a network partition, one has to choose between consistency and availability.
### Mongodb election
In the following diagram, the primary was unavailable for longer than the configured timeout and triggers an automatic failover process, one of the remainder secondaries calls for an election to selct a new primary and automatically resume normal operations. More detail can refer to [this link](https://docs.mongodb.com/v3.6/core/replica-set-elections/).
![replica set triggers election](./images/replica-set-trigger-election.bakedsvg.svg)
Format: ![Alt Text](url)

(Members in replica set send heartbeats to each other.)

### What is failover?
The process that allows a seconday member of a replica set to become primary in the event of a failure. More detail can refer to [this link](https://docs.mongodb.com/v3.6/reference/glossary/#term-failover).
### Rollbacks during replica set failover
A rollback reverts write operations on a former primary when the member rejoins its replica set after a failover. A rollback is necessary only if the primary had accepted write operations that the secondaries had not successfully replicated before the primary stepped down. When the primary rejoins the set as a secondary, it revents, or "rolls back", its write operations to maintain database consistency with the other members. More detail can refer to (this link) (https://docs.mongodb.com/v3.6/core/replica-set-rollbacks/).

### Replica set oplog
Write database first, then write oplog
The oplog (operations log) is a special capped collection that keeps a rolling record of all operations that modify the data stored in your databases. MongoDB applies database operations on the primary and then records the operations on the primary’s oplog. More detail can refer to [this link](https://docs.mongodb.com/v3.6/core/replica-set-oplog/).
**Question: how to handle the loss of rollback data? Are they lost?**
The rollback data was saved in a rollback file, and connect to current primary as soon as possible.

### Write concern
{ w: <value>, j: <boolean>, wtimeout: <number> }
- w: 
1: that is requests acknowledgment that the write operation has propagated to the standalone mongod or the primary in a replica set. It is default write concern for MongoDB.)

majority: Requests acknowledgment that write operations have propagated to the majority (M) of data-bearing voting members.

NOTE

With writeConcernMajorityJournalDefault set to false, MongoDB does not wait for w: "majority" writes to be written to the on-disk journal before acknowledging the writes. As such, majority write operations could possibly roll back in the event of a transient loss (e.g. crash and restart) of a majority of nodes in a given replica set.
- j: The j option requests acknowledgment from MongoDB that the write operation has been written to the on-disk journal.
- wtimeout: This option specifies a time limit, in milliseconds, for the write concern. wtimeout is only applicable for w values greater than 1.

wtimeout causes write operations to return with an error after the specified limit, even if the required write concern will eventually succeed. When these write operations return, MongoDB does not undo successful data modifications performed before the write concern exceeded the wtimeout time limit.

More detail can refer to [this link](https://docs.mongodb.com/v3.6/reference/write-concern/#wc-w)


### Solution
Set w: majority. It can avoid rollback.

### Mongodb Security
#### Forbid specific encryption type from client connection
```bash
net:
  ssl:
      disabledProtocols: TLS1_0,TLS1_1
```
##### Test method
```bash
openssl
```
#### Internal authentication
You can authenticate members of replica sets and sharded clusters. For the internal authentication of the members, MongoDB can use either keyfiles or x.509 certificates.
- keyfiles
The contents of the keyfiles serve as the shared password for the members.
  - Enforce keyfile access control in a replica set
  Enforcing access control on an existing replica set requires configuring:
    - Security between members of the replica set using Internal Authentication, and
    - Security between connecting clients and the replica set using User Access Controls.
  - Keyfile Security
  Keyfiles are bare-minimum forms of security and are best suited for testing or development environments. For production environments we recommend using x.509 certificates.
  - How to
  Restart each member of the replica set with access control enforced.
Restart each mongod in the replica set with either the security.keyFile configuration file setting or the --keyFile command-line option. Running mongod with the --keyFile command-line option or the security.keyFile configuration file setting enforces both Internal Authentication and Role-Based Access Control.

Configuration File
If using a configuration file, set

security.keyFile to the keyfile’s path, and
replication.replSetName to the replica set name.
Include additional options as required for your configuration. For instance, if you wish remote clients to connect to your deployment or your deployment members are run on different hosts, specify the net.bindIp setting. 
```bash
security:
  keyFile: <path-to-keyfile>
replication:
  replSetName: <replicaSetName>
net:
   bindIp: localhost,<ip address>
```
- Create the user administrator.
IMPORTANT

After you create the first user, the localhost exception is no longer available.

The first user must have privileges to create other users, such as a user with the userAdminAnyDatabase. This ensures that you can create additional users after the Localhost Exception closes.

If at least one user does not have privileges to create users, once the localhost exception closes you may be unable to create or modify users with new privileges, and therefore unable to access necessary operations.
- x.509 internal authentication
Member Certificate and PEMKeyFile
To configure MongoDB for client certificate authentication, the mongod and mongos specify a PEMKeyFile to prove its identity to clients, either through net.ssl.PEMKeyFile setting in the configuration file or --sslPEMKeyFile command line option.

**If no clusterFile certificate is specified for internal member authentication, MongoDB will attempt to use the PEMKeyFile certificate for member authentication.** In order to use PEMKeyFile certificate for internal authentication as well as for client authentication, then the PEMKeyFile certificate must either:

  - Omit extendedKeyUsage or
  - Specify extendedKeyUsage values that include clientAuth in addition to serverAuth.





